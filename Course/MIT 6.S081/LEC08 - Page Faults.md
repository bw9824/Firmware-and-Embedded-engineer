# LEC08 - Page Faults

## Big Picture
- Virtual memory gives two things:
  1. **Isolation** (separate address spaces; user vs kernel)
  2. **A level of indirection**: the kernel controls the VA → PA mapping
- So far xv6's mappings are **static** (set up once, at boot or fork/exec). **Page faults make them dynamic** → page tables + page faults = huge flexibility.
- Features covered (all in Linux; none in unmodified xv6, which just kills the process on a user page fault):
  - **Lazy allocation** (lazy lab)
  - **Zero-fill on demand**
  - **Copy-on-write fork** (cow lab)
  - **Demand paging** (+ eviction)
  - **Memory-mapped files** (mmap lab)

---

## 1. Information Available on a Page Fault
Page faults use the **same trap machinery** as syscalls.

| Info | Where |
|---|---|
| **Faulting virtual address** | `stval` register |
| **Type of fault** | `scause`: **12** = instruction, **13** = load, **15** = store (8 = ecall) |
| **Faulting instruction address** | `sepc` → saved in `trapframe->epc` |

- Why the instruction address? After **repairing the page table**, we **restart the same instruction** — now it succeeds.

## 2. Lazy Allocation
**Eager** `sbrk(n)` (unmodified xv6): allocate physical memory, zero it, map it, return.
- Problem: apps can't predict memory needs → **over-ask** (allocate for the biggest possible matrix, use a small one).

**Lazy**: `sbrk` only bumps the size; allocate on first use.
```c
// sys_sbrk
myproc()->sz += n;      // grow virtual size, no physical memory
```
On a page fault with `va < p->sz` (and above the stack):
- allocate a page with `kalloc`, zero it
- map it at **`PGROUNDDOWN(va)`**
- restart the instruction
- If `kalloc` fails (out of memory) → for now, **kill the process**.

### Demo
- After the sbrk change, `echo hi` → page fault: `scause = 15` (store), pid 3 (shell), `sepc = 0x12a4`, `stval = 0x4008`.
- `sh.asm`: 0x12a4 is a store in **`malloc`** (shell's malloc initializing its free list with memory from `sbrk`). 0x4008 = 8 bytes into the 5th page (shell has 4 pages of text + data).
- Handler added to `usertrap()`:
```c
} else if (r_scause() == 15) {
    uint64 va = r_stval();
    char *mem = kalloc();
    if (mem == 0) {
        p->killed = 1;
    } else {
        memset(mem, 0, PGSIZE);
        if (mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE,
                     (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U) != 0) {
            kfree(mem);
            p->killed = 1;
        }
    }
}
```
- Next crash: **`uvmunmap: not mapped`** — freeing lazily allocated pages that were never touched. That invariant ("all user memory below sz is mapped") **no longer holds** → just `continue` instead of panic.
- Now `echo hi` works (with 2 page faults).
- Still missing (the lab): check `va < p->sz`, negative `sbrk` (shrinking), all the cases `usertests` exercises.

## 3. Zero-Fill on Demand
- Binary has **text**, **data** (initialized globals), and **BSS** (globals that start as **zero** — only their size is in the file, saves file space).
- Many BSS pages must all contain zero → map them all to **one physical zero page, read-only**.
- On the first **write** to one: page fault → allocate a new page, zero it, map it **RW**, restart.

Benefits:
- Less memory (big global arrays may be mostly unused)
- **Faster `exec`** — only write PTEs, no allocating/zeroing

Cost: each first write = a **page fault**, much more expensive than a store (trap entry/exit alone saves/restores 100+ registers' worth of loads/stores).

## 4. Copy-on-Write (COW) Fork
- Shell: `fork` copies all pages → child immediately `exec`s → copies thrown away. Wasteful.
- **COW**: child's PTEs point to the **same physical pages** as the parent. Map them **read-only in both** (isolation: one's writes must not be visible to the other).
- On a write fault (say, in the child):
  1. Allocate a new page, **copy** the faulting page into it
  2. Map it **RW** in the child (now private)
  3. The parent's page can be made **RW** too
  4. Restart the instruction (return via `userret`)
- Same virtual address in both, since the address spaces are duplicates.
- Usually only one page gets copied instead of four → less memory and **faster fork**.

**How does the kernel know a write fault is COW and not a real error?**
- PTE has 2 **RSW bits** (reserved for supervisor software) → use one as a **COW flag**.
- Alternatively keep other metadata about address ranges.

**Freeing**: a physical page may now be shared by several processes → **reference count per physical page**; `kfree` only when the count drops to 0. (Otherwise `kfree` fills the page with junk while a child still uses it.)

## 5. Demand Paging
- Unmodified `exec` **eagerly** loads text and data from the file.
- **Demand paging**: `exec` sets up the address space but PTEs are **not valid**; remember which file/offset each page comes from.
- First fault: the **very first instruction at address 0**.
- On fault: read that page from the file, map it, restart.
- Worst case: one fault per page. Usually: less memory, and **`exec` returns much faster** (program starts immediately).

### Running out of memory → Eviction
- Program files may be bigger than physical memory (or many programs together).
- When `kalloc` fails: **evict a page** (write back to its file if needed), reuse that physical page, restart the instruction.
- Which page? **LRU (least recently used)** — most common.
- Prefer **non-dirty** pages: just mark the PTE invalid and reuse — no write to disk needed.
- Hardware helps via PTE bits:
  - **D (dirty)**: set when the page is written
  - **A (accessed)**: set when read or written
- Implement LRU: periodically **clear the A bits** (e.g. every 100 ms); pages that still have A clear weren't recently used → eviction candidates. Famous: the **clock algorithm**.
- In steady state an OS runs with **all memory in use**; starting something new means evicting something.

## 6. Memory-Mapped Files
```c
void *mmap(void *va, size_t len, int prot, int flags, int fd, off_t off);
int munmap(void *va, size_t len);
```
- Map `len` bytes of an open file (starting at `off`) at `va` → read/write the file with **plain loads and stores** instead of `read`/`write` syscalls.
- `prot`: read/write/execute. `flags`: e.g. **shared vs private** mapping.
- **`munmap`**: write back **dirty** pages (D bit tells which).
- Real implementations are **lazy**: record the mapping in a **VMA (Virtual Memory Area)** — file, offset, permissions for an address range. On a fault inside a VMA, read that page from the file.
- Two processes writing the same file region: Unix guarantees nothing about ordering (same as with `write`); use **file locking** if needed.

## 7. Summary
| Feature | Trick | Benefit |
|---|---|---|
| Lazy allocation | `sbrk` only grows size; allocate on fault | Don't pay for unused memory |
| Zero-fill on demand | Share one RO zero page | Less memory, faster exec |
| COW fork | Share RO, copy on write fault | Fast fork, less memory |
| Demand paging | Load page from file on fault | Fast program start |
| Eviction (LRU) | Write out / drop pages, use A/D bits | Use more than physical memory |
| mmap | File-backed pages via VMA | File I/O via load/store |

Common pattern: **leave a PTE invalid or read-only on purpose → catch the fault → do the real work → fix the PTE → restart the instruction**.
