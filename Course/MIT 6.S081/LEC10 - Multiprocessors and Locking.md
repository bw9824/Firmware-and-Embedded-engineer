# LEC10 - Multiprocessors and Locking

## Big Picture
- Apps use multiple cores for performance → the kernel must handle **parallel system calls** → they touch **shared data structures** (proc table, ticks, buffer cache...).
- **Locks** are needed for **correctness** (avoid races), but they **serialize** → limit performance. A fact of life.
- Locks also bring **deadlock**, **modularity** and **performance** problems.
- Implementation details: **atomic swap**, **interrupts off**, **memory fences**.

---

## 1. Why Multicore
- Since the early 2000s **clock frequency plateaued** → single-thread performance plateaued.
- Transistor count kept growing → spent on **more cores**.
- More performance = exploit multiple cores → OS-intensive apps (servers) need the **kernel** to scale on multiple cores too.

## 2. Race Condition Demo
Remove `acquire`/`release` in `kfree`:
```c
void kfree(void *pa) {
  struct run *r = (struct run*)pa;
  // acquire(&kmem.lock);
  r->next = kmem.freelist;
  kmem.freelist = r;
  // release(&kmem.lock);
}
```
- xv6 still boots — races **may or may not** happen.
- `usertests`: a **panic**, and at the end "**lost some free pages**".

What happens (2 CPUs, one shared memory):
```
CPU0: r0->next = freelist     CPU1: r1->next = freelist   (both point to old head)
CPU1: freelist = r1
CPU0: freelist = r0           → r1 is LOST
```
- With more CPUs, others could even briefly see an intermediate list → weirder outcomes.

## 3. The Lock Abstraction
```c
struct spinlock lk;
acquire(&lk);
  ... critical section ...
release(&lk);
```
- **At most one holder** at any time; others wait until `release`.
- Code between = **critical section**: executes **atomically** (all together, never interleaved).
- One **big kernel lock** → every syscall serialized → no parallelism. So kernels like xv6 have **many locks**; syscalls using different locks run in parallel.
- **Nothing enforces** where you put acquire/release — it's up to the programmer.

## 4. When to Lock
**Conservative rule**: if **two processes access a shared data structure** and **at least one writes** → use a lock.
- Too strict sometimes: **lock-free programming** exists (trickier; end of semester).
- Too loose sometimes: `printf` uses a lock to keep a whole string's output together — no shared data structure involved.

### Why locking can't be automatic
Idea: one lock per object, acquired automatically on each operation. Counter-example — `rename("d1/x", "d2/y")`:
```
lock d1; erase x; unlock d1     ← here the file appears not to exist!
lock d2; add y;   unlock d2
```
Correct: **lock d1 and d2 first**, then erase + add, then release both. An operation may need **several locks at once**, chosen by the operation, not by the objects.
(Acquiring all locks up front → effectively a big kernel lock.)

## 5. Three Ways to Think About Locks
1. Locks help **avoid lost updates** (like the lost `kfree`)
2. Locks make **multi-step operations atomic** (critical section)
3. Locks **help maintain invariants** of a data structure: invariant holds when no one holds the lock; may be temporarily broken inside the critical section; restored before `release`.
   - Free list invariant: all free pages on one linked list, head points to first. Temporarily violated mid-`kfree`.

## 6. Problem 1: Deadlock
- Trivial: `acquire(l)` twice by the same holder → waits forever (xv6 **panics** on this).
- **Deadly embrace**:
```
CPU1: rename(d1/x, d2/y)    acquire(d1) ... acquire(d2)  ← waits
CPU2: rename(d2/a, d1/b)    acquire(d2) ... acquire(d1)  ← waits
```
- **Fix: global lock order** — all operations acquire locks in the same order (e.g. lower directory number first).
- Order only needs to be global within a set of locks that are ever held together; independent lock sets have independent orders.

## 7. Problem 2: Locks vs Modularity
- If module M1 calls `f()` in M2, M1 must know **which locks `f` acquires** to respect the global order.
- → M2's **internal** locks leak into M1 → **abstraction violation**, harder modular design.

## 8. Problem 3: Locks vs Performance
- Performance needs **splitting** data structures and locks (per directory? per inode? per process?). The best split isn't obvious.
- Re-splitting = re-check invariants + rewrite code → **lots of work**.
- **Recipe**: start **coarse-grained**, **measure** with real apps. Speed-up OK → done. Otherwise find the **contended** lock and redesign only that part. Avoid unnecessary redesign (costly, complicates reasoning).

## 9. Case Study: UART Lock
- UART has **one lock** (coarse) protecting the **transmit buffer + read/write indices**.
- Roles of that lock:
  - Two processes in `uartputc` don't write the **same slot**
  - Protects the invariant: between r and w = chars to send; the rest = free slots; chars in flight aren't overwritten
  - Ensures **one writer** to the hardware register **THR** (`uartstart` is called with the lock held)
- The **interrupt handler** (`uartintr`) also acquires the lock → the bottom half may run **concurrently on another CPU** with the top half.

## 10. Implementing a Lock
### Broken version
```c
void acquire(struct lock *l) {
  for (;;) {
    if (l->locked == 0) {   // A
      l->locked = 1;        // B
      return;
    }
  }
}
```
Two CPUs both reach A, both see 0, both execute B → **both hold the lock**.

### Fix: atomic hardware instruction (test-and-set)
RISC-V **`amoswap.w r2, r1, (addr)`** — conceptually:
```
lock addr
  tmp  = *addr
  *addr = r1
  r2   = tmp
unlock addr
```
- Most processors have such an instruction. The ISA only specifies behavior; implementation depends on the memory system:
  - single memory controller → it serializes
  - shared bus → **bus arbiter** does both ops atomically
  - caches → **cache coherence protocol** keeps the line in one cache across both ops
- RISC-V also has atomic AND, OR, MAX, MIN, ...

### xv6 spinlock
```c
struct spinlock {
  uint locked;
  char *name;       // debugging
  struct cpu *cpu;  // holder (debugging, e.g. detect double acquire)
};

void acquire(struct spinlock *lk) {
  push_off();                    // disable interrupts
  if (holding(lk)) panic("acquire");
  while (__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;                            // spin: old value 1 = someone holds it
  __sync_synchronize();          // memory fence
  lk->cpu = mycpu();
}

void release(struct spinlock *lk) {
  lk->cpu = 0;
  __sync_synchronize();
  __sync_lock_release(&lk->locked);  // atomic store of 0 (amoswap)
  pop_off();                     // maybe re-enable interrupts
}
```
- `__sync_lock_test_and_set` = C standard atomic → compiles to **`amoswap`** (see `kernel.asm`).
- Lock free (0) → swap writes 1, returns 0 → got it. Held (1) → writes 1 (no change), returns 1 → spin.
- **Why not a plain store of 0 in release?** A store isn't necessarily atomic in hardware: with cache lines bigger than an int, a store = load cache line + update (two micro-ops). Use the guaranteed-atomic instruction instead of reasoning about hardware.

### Why `acquire` turns interrupts off
- `uartputc` holds the UART lock → UART finishes a char → **interrupt on the same CPU** → `uartintr` tries to acquire the same lock → **deadlock** (xv6: panic).
- Spinlocks must handle two kinds of concurrency:
  - between **different CPUs** → the atomic swap
  - between code and an **interrupt on the same CPU** → interrupts off
- Interrupts re-enabled only at the end of `release`.

### Memory ordering → fences
```
locked = 1
x = x + 1      ← could be moved after locked = 0!
locked = 0
```
- **Compiler and CPU reorder** loads/stores when it doesn't change a *sequential* result (lock and x are unrelated) — disastrous for concurrency: the critical section escapes the lock.
- **`__sync_synchronize()`** = **memory fence**: no load/store may move across it. One in acquire, one in release → critical section stays between them.
- (More fine-grained interfaces exist, e.g. acquire/release-ordered atomics; each architecture has a **memory model** the compiler follows.)

## 11. Takeaways
- Locks are needed for correctness but **limit performance** and **complicate programming**.
- Avoid when possible:
  - **Don't share** data if you don't have to
  - Start **coarse-grained**, move to fine-grained only when **measurements** show contention
  - Use a **race detector** to find missing/misplaced locks
- Single CPU with multiple threads: still need atomic critical sections → old uniprocessor kernels just turned **interrupts off/on** instead of real locks.
