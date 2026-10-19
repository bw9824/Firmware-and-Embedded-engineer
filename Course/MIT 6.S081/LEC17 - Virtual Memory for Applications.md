# LEC17 - Virtual Memory for Applications

## Big Picture
- Paper: **Appel & Li (1991)**, "Virtual Memory Primitives for User Programs".
- Kernels use page tables + page faults creatively (lazy allocation, COW). Argument: **user applications** should get the same power — take page faults, change page protections.
- Very different apps (concurrent GC, data compression, shared virtual memory across machines, ...) all need the **same small set of primitives**.
- Examples here: a huge lookup table backed by one page, and a **concurrent garbage collector**.

---

## 1. Primitives Apps Need
| Primitive | Meaning |
|---|---|
| **TRAP** | Deliver a page fault to a **user-level handler**; handler returns, instruction resumes |
| **PROT1** | Decrease accessibility of one page (RW → R, R → none) |
| **PROTN** | Same for **N pages** — cheaper: N PTE changes + **one TLB flush** instead of N flushes |
| **UNPROT** | Increase accessibility (R → RW) |
| **DIRTY** | Find which pages were written |
| **MAP2** | Map the same physical memory at **two addresses** with different protections |

- xv6 exposes none of these to user space (the alarm handler from the traps lab is close to TRAP).

## 2. Unix Today
```c
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t off);
int   mprotect(void *addr, size_t len, int prot);   // PROT_READ, PROT_WRITE, PROT_NONE...
int   munmap(void *addr, size_t len);
int   sigaction(int sig, const struct sigaction *act, ...);   // e.g. SIGSEGV handler
```
- **`mmap`**: map an object into the address space.
  - `addr = NULL` → kernel picks the address.
  - With a file: **memory-mapped file** — manipulate file contents with pointers instead of `read`/`write` (mmap lab).
  - **Anonymous** memory: an alternative to `sbrk`.
- **`mprotect`**: change permissions on a range (page granularity) → PROT1, PROTN, UNPROT.
  - `PROT_READ` → loads OK, stores fault; `PROT_NONE` → any access faults.
- **`munmap`**: remove a mapping.
- **`sigaction(SIGSEGV, handler)`** → TRAP. Without a handler a segfault kills the app; with one, the handler can fix things (e.g. call `mprotect`) and the instruction resumes. General version of the alarm lab's upcall.
- **DIRTY**: no direct syscall (possible with tricks). **MAP2**: achievable with multiple `mmap`s of a shared-memory object.

## 3. Implementation Sketch
**VMAs (Virtual Memory Areas)**: OS data structures beside the hardware page table.
- One VMA per **contiguous** range of virtual addresses (no holes) with the **same permissions** and the **same backing object** (e.g. file + offset for an mmap'd file).
- Roughly one VMA per non-overlapping `mmap` call. The mmap lab implements simple VMAs.

**User-level trap flow:**
1. Access a page whose PTE is invalid / doesn't allow the access → trap into kernel (trampoline, save trapframe)
2. Kernel VM system looks at the VMAs: what is this address?
3. If the app installed a SIGSEGV handler → **upcall** into user space (like `sigalarm`)
4. Handler may call `mprotect`, then returns into the kernel
5. Kernel resumes the process → instruction succeeds now (or traps again if still not fixed)

**Security?** The handler runs in the **same address space** as the app that installed it — it can only affect that app. If it misbehaves, the kernel can kill the process. No isolation broken.

## 4. Example 1: Huge Memoization Table
- Table `T[i] = f(i)` for an expensive `f`; look-up instead of recompute. But the table may be **bigger than physical memory**.
- Plan:
  - Reserve a huge address range, **no physical memory** behind it.
  - Access `T[i]` → page fault → handler maps a page there and fills **all entries on that page** with `f(...)` → resume. Neighbors on the same page are now free.
  - If using too much memory: unmap / protect some pages so they fault again later.
- Demo (sqrt table): `mmap` the region, then unmap it; `sigaction(SIGSEGV, handle_sigsegv)`; handler: get faulting address → `mmap` one page there → compute square roots for that page → **unmap the previously used page**. Random lookups all check out.
- The whole giant table is represented by **one physical page** — extreme (a real app keeps a working set), but shows the power of the primitives.

## 5. Garbage Collection Basics
- GC languages (Java, Python, Go — almost everything except C and Rust): program allocates; the **collector frees** memory no longer reachable.
- **Copying collector**:
  - Heap split into **from-space** and **to-space**. Allocate from from-space.
  - When full: starting from the **roots** (registers, stack variables), copy reachable objects to to-space.
  - After copying an object, update pointers to it and leave a **forwarding pointer** in from-space ("now lives over there") → shared/cyclic structures copied once.
  - Done → everything left in from-space is garbage; swap roles.

## 6. Baker's Incremental ("Real-Time") Collector
- Don't stop the world: on a flip, **copy only the roots**.
- Each `new` call forwards a few more objects → copying cost spread out.
- Problem: the app may follow a pointer that still points into from-space → **every pointer dereference must check** "in from-space? → forward it". Maintains the invariant: pointers in to-space point only to to-space.
- Downsides:
  1. Every load/store becomes several instructions (compare, branch, maybe call) → **expensive**.
  2. Hard to make **concurrent**: a collector thread and the app could both copy the same object → race.

## 7. Using VM Instead
- Divide to-space into **scanned** and **unscanned** regions. Unscanned pages are mapped **`PROT_NONE`** for the app.
- After a flip: copy the roots into the (unscanned) to-space.
- App touches an unscanned page → **page fault** → handler **scans that page** (forwards the objects its pointers refer to into unscanned to-space, fixes pointers) → **unprotects** the page → app continues.
- Benefits:
  - **No pointer checks in code** — the VM hardware does the check.
  - Still incremental.
  - **Concurrency almost for free**: a collector thread scans unscanned pages; the app can't touch them (they fault), and the collector never touches scanned pages → no races.
- Order matters: **scan first, then unprotect** (otherwise other app threads could see unscanned objects).

**MAP2 is required**: how does the collector access pages that are `PROT_NONE`?
- Map the **same physical memory twice**: the app's view (unscanned = no access) and the collector's view (read/write). Same page table, two virtual ranges.
- On Unix: `shm_open` (in-memory "file" object) + `ftruncate` to size of both spaces + **two `mmap`s** of it.

### Toy implementation
- API: `new()` and `readptr(p)`.
- Non-VM version: `readptr` calls `forward`: if `p` is in from-space and not yet copied → copy, leave forwarding pointer; return new address. `new` bumps a pointer; if full → `flip()` (swap spaces, forward the roots).
- VM version: `readptr` just returns `p`. `flip` protects all of to-space for the app, forwards roots. SIGSEGV handler: `scan_page` (using the collector's mapping), then `mprotect` it accessible.
- App: circular linked list built/checked 1000 times → lots of garbage.
- Collection ends when the unscanned area stops growing (all reachable objects copied).

## 8. Is It Worth It?
- Many real GCs **don't use VM tricks**: the compiler knows the generated code and **instruments** it, with tricks to reduce overhead.
- But apps with **no compiler involved** (checkpointing, distributed shared virtual memory) really need these primitives → enough demand that today's OSes support them.
- **VM keeps evolving** (Linux changes it continuously):
  - **5-level page tables** for larger address spaces
  - **Address space identifiers** to reduce TLB flush cost
  - **KPTI** (kernel page-table isolation) because of Meltdown (LEC22)
