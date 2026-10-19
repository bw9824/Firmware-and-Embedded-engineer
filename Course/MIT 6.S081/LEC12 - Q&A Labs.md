# LEC12 - Q&A Labs

## Big Picture
- Live walkthrough of the **copy-on-write (COW) fork** lab, driven by test failures.
- OS lessons: write-protecting shared pages, the **COW page-fault handler**, **reference counting** physical pages, and why **`copyout` must handle COW** (kernel writes bypass the MMU).
- Debugging lesson: small steps; each crash tells you the next thing to fix.

---

## 1. Why COW Fork
- Avoid copying pages that are never modified after `fork`.
- Measured on `usertests`: **~90% fewer bytes copied** — mostly **instruction pages** (never modified).
- Also: less **peak RAM**, **faster `fork`**.
- Downside: if the program ends up writing every page, you pay all the copies **plus** a page fault per page. In practice usually a net win.
- Main challenges:
  1. Don't **free** a shared page until the last user is done → bookkeeping
  2. **`copyout`** writes user memory without going through the MMU → must mimic the page fault

## 2. Strategy
- Take **small steps** (5–10 lines), get each working before moving on. Surprises always appear; a full up-front solution often wastes time.
- Next step = whatever **panic / crash / test failure** shows up next.

## 3. Step 1: Share Pages in fork (`uvmcopy`)
- Test `simpletest`: allocates more than half of memory, then forks → plain copying runs out of memory.
- Change `uvmcopy`: instead of `kalloc` + `memmove`, map the **parent's physical page** into the child:
```c
pa = PTE2PA(*pte);
flags = PTE_FLAGS(*pte);
if (mappages(new, i, PGSIZE, pa, flags) != 0) goto err;
```
- Result: `scause = 2` (**illegal instruction**). Why no store fault? **`PTE_W` still set** → parent and child share pages **read-write**, and their writes corrupt each other (including the shell's code/data).
- Fix: clear `PTE_W` in **both** child and parent (the child must not see the parent's later writes either):
```c
*pte &= ~PTE_W;            // parent
flags = PTE_FLAGS(*pte);   // child gets the same, write-protected
```
- Now: `scause = 15` (**store page fault**), pid 3, `sepc = 0x9da`.
  - Not in `cowtest.asm` — pid 3 is the **shell's child before `exec`**. In `sh.asm`, 0x9da is a store in the prologue of `parsecmd` (saving registers on the **stack**) → as expected, the first thing after fork is writing the stack.

## 4. Step 2: The COW Fault Handler
In `usertrap`:
```c
} else if (r_scause() == 15) {
  if (cowfault(p->pagetable, r_stval()) < 0)
    p->killed = 1;
}
```
- On failure (e.g. out of memory) the only option is to **kill** — there's no syscall to return an error from. (A fancier OS could upcall into the process, like the alarm handler.)

```c
int cowfault(pagetable_t pagetable, uint64 va) {
  if (va >= MAXVA) return -1;                  // walk() panics on huge va
  pte_t *pte = walk(pagetable, va, 0);
  if (pte == 0) return -1;
  if ((*pte & PTE_U) == 0 || (*pte & PTE_V) == 0) return -1;
  uint64 pa1 = PTE2PA(*pte);
  uint64 pa2 = (uint64)kalloc();
  if (pa2 == 0) return -1;
  memmove((void*)pa2, (void*)pa1, PGSIZE);
  *pte = PA2PTE(pa2) | PTE_U | PTE_V | PTE_W | PTE_X | PTE_R;
  kfree((void*)pa1);                            // drop one reference (added later)
  return 0;
}
```
- **Defend against crazy addresses**: most have no PTE. The ones that do but must not be written (trampoline, trapframe, stack guard page) all have **`PTE_U` clear**.
- Here "write-protected + `PTE_U`" ⇒ COW page. With more features (lazy, mmap, swap) you'd need better bookkeeping — real OSes keep their own **per-address-space data structure** describing each page, separate from the page table. (Or use an RSW bit.)
- **The other process sharing the page: do nothing.** If it ever writes, it faults and copies too.
- Modifying xv6 helpers (e.g. a remap that doesn't panic) is fine — their panics guard invariants of *unmodified* xv6.

## 5. Step 3: Reference Counts
- Crash: **illegal instruction** again. The child shell **execs** `cowtest` → `exec` **frees all its old pages** → those are the **shared** pages of the parent shell, including its **instructions** → immediately reallocated and overwritten → parent runs garbage. (Could show up as any kind of weird fault.)
- Rule: free a physical page only when **no page table refers to it**.
```c
int refcount[PHYSTOP / 4096];   // one count per physical page
```
- Counts go **up** in `fork` (`uvmcopy`), **down** on `exit`, `exec`, and COW copy.
- `kalloc`: set count to **1** (sanity check it was 0).
- **`kfree` changed meaning**: decrement; actually free only when it reaches **0**. All existing `kfree` callers then do the right thing automatically.
```c
acquire(&kmem.lock);
if (refcount[pa/4096] <= 0) panic("kfree ref");
refcount[pa/4096] -= 1;
int still = refcount[pa/4096] > 0;
release(&kmem.lock);
if (still) return;
// fill with junk, put on free list...
```
- **Needs a lock**: two processes sharing a page may `exit` on different cores at the same time. Any lock works as long as all count updates use the same one. Check must happen **before** filling the page with junk.
- `incref(pa)` called in `uvmcopy`.
- Boot panic: `freerange` calls `kfree` on pages with count 0 → set counts to 1 first.
- Then: `cowtest` runs **out of memory** on the second round → forgot to drop the old page's reference in `cowfault` → add `kfree(pa1)`.

> 💡 Global arrays like `refcount` live at addresses fixed by the **linker** in the kernel image, just like globals in a user program.

## 6. Step 4: copyout and COW
- `filetest`: `read` returns **-1** in several forked children. Prints show `argfd` fails: **fd 9 doesn't exist** — and every child got fd 9.
- Cause: `pipe()` returns fds via **`copyout`** into the user array. `copyout` uses `walkaddr` + direct physical write → **doesn't check `PTE_W`**, no MMU involved → writes the **shared** COW page → every child sees the same fds.
- Fix in `copyout`:
```c
if (va0 >= MAXVA) return -1;
pte_t *pte = walk(pagetable, va0, 0);
if (pte == 0 || (*pte & PTE_V) == 0 || (*pte & PTE_U) == 0) return -1;
if ((*pte & PTE_W) == 0) {
  if (cowfault(pagetable, va0) < 0) return -1;
}
pa0 = PTE2PA(*pte);     // re-read: cowfault may have changed it
```
- Here we **can return an error** to the syscall instead of killing.
- (Write-protected + U ⇒ COW in this xv6. Could also use refcount > 1, but a page with refcount 1 may still be write-protected in the parent after the child copied.)
- `usertests` `copyout` test passes **ridiculous pointers** → `walk` panics → check **`va < MAXVA`** first.

## 7. Debugging Advice
- No universal strategy — every page-table bug is different.
- **Print statements** to gather information and test hypotheses.
- **Brainstorm** what could be wrong by reading the code; add prints/panics to rule guesses in or out.
- If it worked half an hour ago: **go back** to the working version and re-apply changes bit by bit.
- Early, broken states produce "complete bananas" errors (e.g. `init` crashing because the shell it forks shares corrupted pages).
