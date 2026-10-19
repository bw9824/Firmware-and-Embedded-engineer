# LEC07 - Q&A Labs pgtbl

## Big Picture
- Q&A walking through the staff solution of the page table lab.
- OS lessons: reading a real user page table, **per-process kernel page tables**, and **mapping user memory into the kernel page table** (why real kernels do it).
- Virtual-memory bugs are notoriously hard: few lines of code, but **spectacular failures far from the cause**.

---

## 1. Why VM Programming Is Hard
- Worst case: QEMU just **freezes**, no output. Best case: a **panic** — but the real cause (a wrong page-table entry) happened much earlier.
- The first VM lab has always been the hardest, even for the staff.
- Strategy: **baby steps** — plan the whole change, but make one or two changes at a time and test. **Keep the old code**, add new code beside it, switch over → you can always compare and roll back.

## 2. Setting Reminder
- Page tables themselves **live in memory** (allocated from free memory by `kalloc`).
- `satp = 0` → no translation (VA = PA), as at boot. Load a root page-table PA into `satp` → MMU translates.

## 3. Reading init's Page Table (vmprint)
```
page table 0x...
 ..0: pte ... pa 0x87f6...          (L2 entry 0)
 .. ..0: pte ... pa ...             (L1 entry 0)
 .. .. ..0: pte ... flags 0x1f     → text + data   (V R W X U)
 .. .. ..1: pte ... flags 0x0f     → guard page    (no U!)
 .. .. ..2: pte ... flags 0x1f     → user stack
 ..255: pte ...
 .. ..511: pte ...
 .. .. ..510: flags 0x07           → trapframe  (R W, no U)
 .. .. ..511: flags 0x0b           → trampoline (R X, no U)
```
Observations:
- User address space = text, data, guard page, stack, then heap (grows with `sbrk`), trampoline + trapframe at the top.
- **Guard page**: mapped but **no `U`** → a user stack overflow faults instead of corrupting data.
- **Stack** has `X` set — could be tighter (no executing code from the stack).
- **Trampoline/trapframe have no `U`** → only the kernel can use them (during the transition, while still on the user page table).
- **Physical pages are not contiguous** even though virtual pages are → flexibility for allocating/freeing.
- **Text and data share a page** → only for simplicity (simplest `exec`; linker flag forces them contiguous). A real OS would separate them to set permissions precisely.

> 💡 Why is the top at L2 index **255**, not 511? xv6 uses only **38 bits** of the 39. If bit 38 were set, all higher bits 39–63 would have to be 1 too (sign extension) — xv6 avoids dealing with that. Downside: can't use RAM beyond 2^38 (not an issue here).

## 4. Per-Process Kernel Page Table
Task: give each process its own copy of the kernel page table. Harder than it sounds:
- xv6 code is specialized for **one** kernel page table.
- `kvminit` isn't the whole story — **`procinit`** adds kernel stacks, **`virtio_disk`** also uses it.
- Must **free** it when the process exits — without freeing parts still used by others.
- Small mistakes show up as panics **much later**.

**Two approaches:**
| Approach | How |
|---|---|
| **Copy** | Allocate pages and literally build a full copy |
| **Share** | Copy L2 entries **1–511** (PTEs pointing at the global kernel's lower-level tables); rebuild only **entry 0** |

- Entry 0 of the top directory covers **1 GB** (entry 0 in L1 covers 2 MB, in L0 4 KB). It contains CLINT, PLIC, UART, VIRTIO; **KERNBASE (0x80000000) is entry 2**. User memory will be mapped there (Part 3), so only that subtree is per-process.
- Free: walk only entry 0's subtree, free its L1/L0 pages, then the root. Never free shared subtrees.

**Scheduler must switch `satp`:**
- Before running a process → load **its kernel page table**.
- When the process stops and we return to the scheduler → switch back to the **global `kernel_pagetable`**.
- Why: a process's page table is freed (by the parent's `wait`) when it's not running. **Never run on a page table that may be freed.** The scheduler may also run with no processes at all.

## 5. Mapping User Memory Into the Kernel Page Table
Goal: map the user address space into the bottom of each process's kernel page table (below PLIC).

**Why do this?** Compare `copyin`:
- **Old**: user memory isn't mapped in the kernel → for each page: translate user VA with the user page table (software `walkaddr`), then copy via the identity-mapped PA. A struct spanning a page boundary is copied in pieces.
- **New**: just `memmove` from the user address — **hardware walks the table**.

Advantages:
- Simpler kernel code
- **Faster** for big copies (e.g. `write` of a huge buffer into a pipe or file — no per-page software walk)
- Kernel could **read/write user structures in place** (update one field) instead of copyin + copyout
→ Many real kernels map user memory into the kernel address space.

**Code idea (`kvmmapuser`)**: for each page from old size to new size:
- `walk(user_pt, va, 0)` → user PTE (should be valid)
- `walk(kernel_pt, va, 1)` → kernel PTE (allocate intermediate levels)
- Copy the PTE (physical memory is **shared**), clearing:
  - **`PTE_U`** — on RISC-V, supervisor mode **can't access `U` pages** (by default)
  - `PTE_W`, `PTE_X` — kernel only reads this memory (conservative)

> 💡 Clearing these is mainly a **debugging aid**, not isolation: the kernel controls everything anyway (it can change `satp` or turn off paging). It catches bugs like the kernel accidentally dereferencing or jumping to user addresses.

**Where to call it**: `fork`, `exec`, `sbrk` (`growproc`), `userinit`.
- **`fork` must map the child's own user pages**, not the parent's. If the parent exits first, its pages are freed (and filled with junk) while the child's kernel table still points at them → crashes in `usertests`.
- Remove panics in `uvmfree`/`freewalk` that assumed an invariant of unmodified xv6 (leaf pages present).

## 6. Design Questions From Students
**Let user programs grow beyond PLIC?** Remap devices (UART, VIRTIO, PLIC, CLINT) to unused virtual addresses **above PHYSTOP**, in every kernel page table → frees the low gigabyte for users. Real kernels do this.

**Why are kernel stacks mapped high?** So there's an **unmapped guard page** below each one with **no physical memory** behind it. Overflow → page fault → panic instead of overwriting data. Not bulletproof: a huge stack buffer could jump past the guard page — but very effective in practice.

**With user memory in the kernel page table, do we still need to switch page tables on traps?**
- In principle, **one page table** for user + kernel makes entry/exit simpler (no switch).
- Linux did exactly this until recently (relying on `U` bits to protect kernel pages from user code).
- After **Meltdown**, Linux added **KPTI** mode: separate kernel and user page tables — like xv6.
- On **Intel**, the kernel can access `U` pages; on RISC-V you can allow it too by setting the **`SUM`** bit in `sstatus`.

## 7. Other Notes
- Pipes: a buffer in kernel memory; `write` to a pipe uses `copyin` → benefits from the faster copyin.
- Labs don't build on each other on purpose (a hidden bug in an earlier lab shouldn't break a later one).
