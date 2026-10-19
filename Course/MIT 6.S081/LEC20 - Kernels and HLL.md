# LEC20 - Kernels and HLL

## Big Picture
- Question students keep asking: **in what language should you write a kernel?** Hotly debated, few facts.
- Paper: "The benefits and costs of writing a POSIX kernel in a high-level language" — **Biscuit**, a Unix-like monolithic kernel in **Go** (garbage collected), compared against **Linux**.
- No crisp answer, but data: HLL kernel is **feasible**, prevents many memory-safety exploits, costs roughly **5–15%** performance.

---

## 1. Why C
- **Full control** over memory allocation and freeing.
- Almost **no implicit code** — you can imagine the RISC-V instructions behind each line.
- **Direct memory access** (PTE bits, device registers).
- **Few dependencies**, no big runtime — xv6 boots with a few lines of assembly, then C.
- Windows, Linux, BSDs are all C.

**Downside: hard to write secure code.**
- **Buffer overruns** (write past an array, below the stack)
- **Use-after-free**; with threads sharing memory, hard to know when memory can be freed
- 2017: **~40 Linux kernel CVEs** letting an attacker take full control, from memory-safety bugs.
- Students hit use-after-free in the COW lab too.

## 2. High-Level Language Trade-offs
**Benefits:**
- **Memory/type safety** → such bugs become a panic or can't be written at all
- **Garbage collection** → freeing is easy
- Good for **concurrency**; good **abstractions** (interfaces...) → modular code

**Costs ("HLL tax"):**
- Bounds checks, nil-pointer checks, more expensive casts
- **GC cycles**
- Perceived incompatibilities: no direct memory access, no handwritten assembly (needed for context switch, boot), language's concurrency model may not match kernel patterns (e.g. one thread passing a lock to another during scheduling)

**Goal**: measure trade-offs (safety, programmability, performance) in a **high-performance** kernel. xv6 wouldn't do — it's slow anyway (quadratic/linear-search algorithms). Prior HLL kernels explored new OS designs, didn't do head-to-head comparisons. Building a production-grade kernel is hard for a small team → build something with Linux's important aspects, optimize to the same ballpark, compare carefully.

## 3. Methodology
- Biscuit provides a subset of Linux syscalls with the **same arguments/conventions** → run the **same apps** (e.g. **NGINX**), same syscall traces, on both kernels.
- **Why Go?**
  - **Statically compiled** (not interpreted like Python), good compiler → high performance
  - Designed for **systems programming**: easy to call assembly/foreign code, good concurrency support
  - Has a **garbage collector** — the main thing to evaluate
- **Rust** wasn't mature then. Today it might be chosen — but Rust **sidesteps** the question by needing no GC; the point was to measure GC's cost. A Rust Linux probably wouldn't beat C, roughly the same ballpark.

## 4. Biscuit Design
- Classic **monolithic** kernel, like xv6 but more complete and faster.
- User programs (in C) are **multi-threaded**; each user thread has a **kernel thread** = a **goroutine** (Go's threads, scheduled by the Go runtime, which also provides sleep/wakeup-like condition variables).
- **Go runtime runs on bare hardware**: boots first; a small **shim** tricks it into thinking there's an OS underneath (e.g. gives it memory for its heap).
- Features: multicore with fine-grained locking; journaling FS (like ext3); VM system with **VMAs** and `mmap`; full **TCP/IP** stack; high-performance drivers (**10 Gb NIC**, sophisticated disk driver).
- **58 syscalls**, **~28,000 lines** (xv6: ~20 syscalls, <10K lines).
- Each user process has its own page table; syscalls work like xv6 (args in registers, `SYSENTER` on x86, trapframe, return).
- Locks + condition variables mostly, **few channels** (channel-heavy FS design performed worse).

## 5. Implementation Challenges
1. **Run Go runtime on bare metal** with ~zero modifications (so Go upgrades just work — upgraded many times). Go is mostly OS-agnostic; emulate the few OS features it needs to start.
2. **Goroutines for different user processes** need different page tables, but the Go scheduler is unmodified and can't switch page tables → like xv6, **switch page tables on kernel entry/exit**; use `copyin`/`copyout` software walks.
3. **Device interrupts**: Go has no notion of interrupts or "interrupts off while holding a lock". → Interrupt handler does almost **nothing**: no locks, no allocation; just set a flag and **wake a dedicated goroutine** that does the real work.
4. **Heap exhaustion** — the big surprise (below).

## 6. Heap Exhaustion
- Kernel allocates objects on a heap (sockets, file descriptors, `struct proc`...). (xv6 statically allocates everything.)
- Many heavy processes → heap fills up; nobody frees yet. What does the kernel do when it needs one more object?
- Every kernel faces this. For Biscuit it's serious: **Go's `new` can't fail**.

**Strawman solutions:**
1. **Panic** (xv6's bcache does this) — not a real solution.
2. **Wait in the allocator** until memory frees up → **deadlock** risk: you may hold a lock that the process who'd free memory needs (with a big kernel lock, obvious; with fine-grained locks, still easy).
3. **Fail the allocation** (return null) and **bail out** → must undo memory already allocated and partial multi-step disk operations → very hard to get right.
- Linux uses both 2 and 3 and has trouble with them (search "too small to fail").

**Biscuit's solution: reserve first.**
```
syscall entry:  reserve(max memory this syscall could need)   // wait here if needed
                ... run syscall: allocations can never fail ...
syscall exit:   return reservation to the pool
```
- At syscall start, **no locks/resources are held** → waiting is safe (no deadlock). While waiting, kernel can **evict caches** or **kill a process** to free memory.
- No allocation-failure checks or error-handling code anywhere in the kernel.
- **How much to reserve?** Too much (e.g. half of memory) limits concurrency.
  - Go is easy to **statically analyze** (standard packages — the compiler uses them).
  - Compute the syscall's **call graph**, its max depth, sum of live allocations along it (object sizes known).
  - **Escape analysis**: memory allocated in a callee and returned to the caller stays live → add it to the caller.
  - Worst case (conservative). Loops bounded by syscall arguments → **annotate** max bounds; **avoid recursion**; annotate max slice capacities (slices double when growing).
  - A few days of work by hand → a reasonably tight bound per syscall.

## 7. Performance Optimizations
- Same tricks as Linux: **large pages** for kernel text (fewer TLB misses), **per-CPU NIC transmit queues**, **RCU** read-lock-free directory cache.
- Lesson: **Go didn't stand in the way** of these optimizations. Lots of work regardless of language.

## 8. Benefits
**Did Biscuit avoid expensive Go features?** Static analysis counted feature use per 1000 lines vs the **Go runtime** and **Moby** (big Go projects): allocations, maps, slices, channels (little), **multi-value return** (most loved), closures, defer, goroutines, interfaces, type assertions, many packages. Similar usage → no cheating.

**Simpler code (qualitative):**
- **GC**: `exit` doesn't need to carefully free every structure (address space, VMAs...) — the collector does it.
- Multi-value returns, closures, **maps** (no linear searches like xv6).

**Concurrency example — who frees a shared object?**
- Two threads share a buffer; who frees it when both are done? In C: reference counts + locks/atomics → expensive at high core counts.
- **Lock-free readers** (common in Linux):
```
get():  return atomic_load(&head)      // no lock
pop():  lock; h = head; atomic_store(&head, h->next); unlock; free(h)?  ← unsafe!
```
  A concurrent reader may still hold `h` → **use-after-free**. Linux's fix: **RCU** — defer freeing until provably safe; but RCU critical sections have rules (e.g. can't sleep/schedule) → error-prone.
- With GC: **non-issue**, freed only when unreachable.

**CVEs**: of the Linux CVEs examined, ~40 memory-safety bugs (use-after-free, double free, out of bounds): **8 disappear** in Go (GC), **32 become panics** (bounds checks) instead of exploits. Logic bugs stay logic bugs.

## 9. Costs
Setup: real hardware (4 cores, 2.8 GHz, 16 GB, hyperthreading off). Apps: **web server (NGINX), key-value store (Redis), mail server benchmark** — kernel-intensive (most time in kernel).

**Biscuit vs Linux 4.9** (Linux features Biscuit lacks disabled where possible, e.g. page table isolation):
- Biscuit always slower: roughly **10–15%**. Not apples-to-apples, but **same ballpark** (not 2–10× off).

**Where the cycles go:**
| Category | Share |
|---|---|
| GC | ~3% (with the GC actually running) |
| Function **prologues** (check if goroutine stack must grow) | highest — Go devs think it can be lowered |
| Write barriers (GC tracking pointers) | very little |
| Safety checks (bounds, nil) | ~2–3% |

**GC cost depends on live data**: GC traces all **live** objects.
- 640 MB live data (2M vnodes). Free memory only **0.5×** live → **~34%** GC overhead. Free memory **2×** live → **~9%**.
- → Keep GC overhead < 10%: physical memory ≈ **3× the live heap**.

**Pauses**: Go's GC is **concurrent** with short stop-the-world (to enable write barriers) + **incremental** work in `new`.
- Max single pause **150 µs** (NGINX: marking a large TCP connection table).
- Max total pause in one HTTP request **582 µs**; only **0.3%** of requests delayed > 100 µs.
- Fine compared to tail-latency targets (ms to tens of ms). Each Go upgrade had a better GC.

**Identical code paths in C vs Go** (painstaking): **pipe ping-pong** kernel path (syscall entry, fd layer, pipe, copyin/out, scheduler wakeup): ~1.2K vs 1.8K assembly lines, same top-10 hot instructions, no allocation/GC → **Go ~15% slower**; extra **prologue + safety-check** instructions (~16 more) account for it.

## 10. Conclusions
- Performance **paramount** (can't give up ~15%) or **minimize memory** → **C**.
- **Safety/security** important → **HLL**. Often performance is merely important, not paramount → HLL is reasonable.
- A programming language is a programming language — it **didn't stand in the way** of building a kernel.
- 6.S081 stays in C: the course's purpose is to see **everything** between the CPU and the syscall interface; Go hides too much (e.g. threads). Students also liked C for understanding what really runs on the CPU — while noting bugs like returning pointers to dead stack objects, writing past string ends, `strncpy` vs `safestrcpy`, missing maps.

## 11. Q&A Notes
- **Runtime**: Go's runtime provides threads, scheduler, hash tables, GC (needs a heap) → the **shim** supplies what it would ask an OS for. C has a tiny runtime (printf, string functions); xv6's "shim" is just setting up a stack before `main`.
- Biscuit has ~**1,500 lines of assembly** (boot, getting ready for the Go runtime); early Go code is written carefully not to allocate.
- Developed mostly in **QEMU**; real hardware needed different boot code. Debugging early boot: write characters synchronously to the UART.
- **Goroutines are cheap**: stacks start small and **grow dynamically** (the prologue check); runtime multiplexes many goroutines on a few kernel threads ("M threads") — user-level scheduling. C/C++ thread libraries can do similar.
