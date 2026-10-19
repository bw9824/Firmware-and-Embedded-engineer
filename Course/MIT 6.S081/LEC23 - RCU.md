# LEC23 - RCU

## Big Picture
- Goal: **good multi-core performance for shared kernel data that is read much more than written.**
- The kernel is a parallel program; locks give correctness but **prevent parallelism**.
- **RCU (Read-Copy-Update)** (Linux): readers run with **no locks and no writes to shared memory**; writers do extra work. A big success for read-heavy data.

---

## 1. The Problem
- Many processes making independent syscalls *should* run in parallel, but the kernel has **shared data** (memory, disk cache, inode cache, scheduler lists).
- Example: a **singly linked list**, mostly **read** (scanned), occasionally **written** (change content, insert, delete).
- xv6 uses a spin lock → **readers also lock** (to avoid seeing a half-updated list). But exclusive locks let only **one reader at a time** → no parallelism even with no writers.

## 2. Read-Write Locks (and why they still don't scale)
```c
r_lock(l); ... read ...; r_unlock(l);     // many readers OR
w_lock(l); ... write ...; w_unlock(l);    // one writer, never both
```
- Simplified Linux implementation: a count `n` — `0` = free, `-1` = writer, `>0` = that many readers.
- `r_lock` uses **compare-and-swap** (atomic "if `n == x`, set to `x+1`, else fail") in a loop to increment safely against concurrent writers/readers.
- **Writer starvation**: with readers present (`n > 0`), a writer's CAS (expects `0`) keeps failing → may never acquire.
- **The real killer — cache coherence**: each core caches `n` in its L1. Each CAS that changes `n` must **invalidate** every other core's copy (messages over the interconnect).
  - `n` cores all calling `r_lock`: ~`n` trips through the loop, each costing ~`n` invalidation messages → **O(n²)** total.
  - A read-only access that could've been satisfied from cache in a few cycles becomes an expensive **write** to shared data costing hundreds/thousands of cycles.
- **Key insight**: any **write** to data cached by other cores forces cross-core communication; reads of unmodified data are cheap (local cache). We want readers that **never write anything**, including lock bookkeeping.

## 3. What Goes Wrong With No Reader Locks
Reading a list while a writer modifies it (writers still lock against each other):
1. **Change content in place** → reader sees a mix of old and new bytes.
2. **Insert** → if `head` points to the new element before it's initialized, reader follows a garbage next pointer.
3. **Delete + free** → reader holding a pointer to the freed element sees it reused/overwritten.

RCU fixes all three by making **writers** more careful, so readers need no locks.

## 4. RCU Idea 1: Copy, Don't Modify in Place
- Writers **never modify a list element's content in place**. To "update" element E2:
  - Allocate a **new** E2′, fully initialize it (content + next pointer to E3)
  - With a **single committing write**, switch E1's next pointer from E2 to E2′
- Pointer writes are **atomic** → a reader sees either the old or the new fully-valid element, never a half-update.
- Works for structures updatable with **one committing pointer write**: **lists, trees**. Not **doubly-linked lists** (two pointers to each node, can't swap atomically). A tree update builds a new subtree and swaps a head pointer, sharing the unmodified parts.

## 5. RCU Idea 2: Memory Barriers
- Compilers and CPUs **reorder** memory operations → "initialize E2′, then set E1→E2′" isn't reliable without barriers.
- **Writer**: barrier **before** the committing write → E2′ is fully initialized before it's linked (`rcu_assign_pointer`).
- **Reader**: load the pointer, **barrier**, then dereference → the dependent reads happen after loading the pointer, so they see initialized content (`rcu_dereference`).

## 6. RCU Idea 3: Grace Periods Before Freeing
- After swapping E1→E2′, some reader may still be using the old E2. Must not free it yet.
- **Not** reference counts (readers would write → defeats the whole point). **Not** a garbage collector (Linux isn't GC'd).
- **The rule**: readers may **not hold an RCU pointer across a context switch** (can't yield the CPU inside an RCU critical section — like a spin-lock critical section).
- **Writer waits a grace period**: until **every core has context-switched at least once** → no reader can still hold a pointer to the old element → safe to free.
- Implementation (paper's simple version): the writer briefly runs on **every core** (forcing a context switch on each) via **`synchronize_rcu()`**, then frees.
```c
// writer
acquire(&lock);
old = head;
e = alloc(); e->data = ...; e->next = old->next;
rcu_assign_pointer(head, e);   // barrier + atomic pointer write
release(&lock);
synchronize_rcu();             // wait for a grace period
free(old);
```
- `synchronize_rcu` may take a **fraction of a millisecond** — acceptable because RCU writes are rare.
- **`call_rcu(ptr, free_fn)`**: returns instantly, defers the free via background bookkeeping. Risk: the deferred-free list can grow huge → out of memory → discouraged.

## 7. Reader Code
```c
rcu_read_lock();                       // just sets a "no context switch" flag — nearly free
for (e = rcu_dereference(head); e; e = rcu_dereference(e->next))
    ... use e ...
rcu_read_unlock();                     // clears the flag
```
- `rcu_read_lock/unlock` do **almost nothing** (a nested-counter flag so a timer interrupt won't context-switch).
- `rcu_dereference` = load pointer + memory barrier (a few dozen cycles) → much cheaper than a lock.
- **Constraint**: must **not return** an RCU-protected element or a pointer into it outside the critical section (could context-switch while holding it). Returning a **copy** of the data is fine.

## 8. Performance
- **Readers**: essentially **zero** synchronization overhead (just the flag + one barrier). Scale perfectly.
- **Writers**: slower — still lock among themselves, plus the potentially long `synchronize_rcu`.
- Net win depends heavily on the **read/write mix** → always **measure** against real workloads.

## 9. Limitations
- Only helps when **reads ≫ writes**.
- Readers can't **sleep/yield** holding RCU data → sometimes must re-lookup after a sleep → awkward code.
- Best for **single-committing-write** structures (lists, trees), not arbitrary ones (sequence locks allow in-place updates but are more complex and narrower).
- Readers may see **stale data** for an unbounded time (a reader that grabbed a pointer just before a replace may use the old value much later). Usually fine (operations were concurrent anyway), occasionally surprising.

## 10. Write-Heavy Data (aside)
RCU is for read-heavy data. For write-heavy data, the main technique is **avoid sharing — restructure per-core**:
- **Per-CPU free lists** (the lock lab's `kalloc`): each core has its own list; touch another's only when yours is empty.
- Per-core scheduler run queues; **per-core statistics counters** (writes cheap/local, reads lock and sum all of them).

## 11. Takeaways
- RCU is a **giant Linux success** — read-mostly data (e.g. cached file blocks) is everywhere, so speeding up reads is broadly useful.
- It's "magic" because it eliminates **locking and writing for readers** entirely.
- The key enabler is the **grace-period** deferred free — RCU is as much a specialized **garbage-collection** technique as a synchronization one.

> 💡 NUMA note: "one shared RAM" is the logical model; physically, multi-socket machines attach RAM to each CPU chip with a fast interconnect, and accessing another chip's RAM sends a message — a lot of engineering maps the shared-memory model onto real hardware.
