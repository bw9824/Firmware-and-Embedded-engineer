# LEC18 - OS Organization

## Big Picture
- Question: **what should a kernel do at all?** Unix/xv6's set of abstractions isn't the only answer (and programmers' preferences differ).
- **Monolithic kernels** (Unix, Linux, xv6) vs **microkernels** (only IPC + threads in the kernel).
- Paper: "The Performance of µ-Kernel-Based Systems" — **Linux running as a user-level server on L4** is only a few % slower than native Linux → microkernels can't be dismissed on performance alone.

---

## 1. Monolithic Kernels: Why
- Kernel = **one big program** with **powerful abstractions** (file system, file descriptors, address spaces) instead of raw hardware.
- Advantages of big abstractions:
  - **Portability**: files/dirs work on any disk, SSD, network FS; address spaces instead of raw MMU access
  - **Hide complexity**: simple `read`/`write` interface, very complex code behind it
  - **Resource management & sharing** (free memory, free disk blocks) → simpler programs, **robustness**, **security** (programs can't grab each other's disk blocks)
- **Integration**: subsystems can look into each other's data. E.g. `exec` touches both the file system (load binary) and virtual memory (set up address space) — easy in one program.
- All kernel code runs with **full hardware privilege** → convenient for kernel developers.

## 2. Monolithic Kernels: Why Not
- **Big and complex** (Linux: hundreds of thousands to millions of lines, lots of interconnections) → **bugs**, including exploitable **security bugs**.
- Grow every desirable feature → **general purpose** (phones, laptops, servers, routers) → maybe **not optimal** for any one use (e.g. your web server doesn't need sound card support). Hard to optimize a code path that must handle a thousand cases. Writing 1 byte through a pipe involves buffering, locking, sleep/wakeup, scheduling, context switch...
- **Design decisions baked in**, apps can't change them:
  - Can only `wait` for your own children, not grandchildren or unrelated processes
  - Can only change your own address space, not another process's
  - A database knows how to lay out B-trees on disk best; the file system doesn't
- **Extensibility** (1990s topic): programs couldn't change the kernel on the fly.

## 3. Microkernel Idea
- **Tiny kernel**: only **IPC** (inter-process communication) + **threads/tasks** (+ minimal address space support).
- Everything else = **user-level server processes**:
```
user:    vi    cc    window system  |  FS server   disk driver   net stack   pager
         ───────────── IPC ──────────────────────────────────────────────────────
kernel:  µkernel (tasks, threads, address spaces, IPC)
```
- `vi` reads a file → IPC to FS server → IPC to disk driver → driver talks to hardware → reply → reply.
- "Microkernel" is a general approach, not one artifact; many different projects.
- Today: **L4** variants in many **cell phones** (radio microcontrollers) and Apple's **secure enclave** processor; embedded/specialized systems. The **user-level services + IPC** idea also spread into mainstream OSes (e.g. **macOS**).

## 4. Hoped-For Benefits
- **Aesthetics**: smaller, more focused, not a grab bag.
- **Security**: fewer lines → fewer bugs; small enough to **formally verify** (**seL4**, a descendant of L4, is proven correct).
- **Speed**: small code easier to optimize; don't pay for unused features.
- **Flexibility**: fewer baked-in decisions.
- **Modularity**: OS split into separate services; user-level code easier to **modify/customize**.
- **Robustness**: a crashing user-level service (e.g. a **driver** — most kernel bugs are in drivers) can be **restarted** instead of rebooting.
- Run **multiple OS "personalities"** (e.g. Unix servers) on one microkernel.
(These were hopes, not guaranteed consequences.)

**Challenges:**
- What's the **minimal syscall API** that's still powerful enough to build `fork`, `exec`, COW, mmap... in a kernel that knows nothing about files?
- Must **build the rest of the OS** as user-level servers.
- Lots of IPC → **IPC must be very fast**.
- Monolithic kernels gain performance from **integration**; splitting things up may lose optimizations.

## 5. L4 (as of the paper)
- **7 system calls** (Linux: ~350; xv6: 21). ~**13,000 lines** (xv6 kernel: ~6–7K; Linux 20–30× bigger).
- Abstractions:
  - **Tasks** (≈ process, address space) with **multiple threads** each (structure programs, use multicore)
  - **Address spaces**: map pages in your own or (with permission) another task's address space — via special IPC messages
  - **IPC**: send bytes to a thread identified by ID
- Creating a program: `thread_create` (new task, no memory) → special IPC to **map pages** you prepared (code/data) into it → special IPC with start **PC and SP**.
- **Devices**: privileged tasks can map device registers into their address space; L4 turns **interrupts into IPC messages** to the driver task. L4 knows nothing else about devices.
- **External pagers**: a task's **page faults become IPC messages** to its designated **pager task**, which can allocate memory, map it (special IPC), and resume the thread → lazy allocation, COW fork, mmap'd files all in **user space**. (In Linux you'd have to modify the kernel to add COW fork.)
- No file system, no fork/exec, no pipes, no drivers, no networking in the kernel.
- L4 does **thread scheduling/switching** (timer interrupts, saved registers per thread, switch page tables) — like xv6.

## 6. Slow IPC (pipe-like design, early microkernels such as Mach)
```
P1: send(P2, msg)     → copy msg into kernel buffer, return    (asynchronous, buffered)
P1: recv(&reply)      → sleep until reply appears
    ... scheduler, context switch ...
P2: recv(&msg)        → copy msg out of kernel buffer
P2: send(P1, reply)   → copy into kernel buffer
    ... eventually P1 scheduled, copy reply out
```
- Per request/response: **4 syscalls**, **8 user/kernel crossings**, message copies in and out of kernel, sleep, full scheduler run + context switch.
- Each crossing switches page tables → disturbs CPU caches/**TLB**.

## 7. Fast IPC in L4 ("Improving IPC by Kernel Design") → ~20× faster
| Technique | Idea |
|---|---|
| **Synchronous** | `send` waits for the receiver to be in `recv` (and vice versa). If P2 is already waiting, P1's send **jumps straight into P2** as if returning from P2's `recv` — no scheduler, no general context switch |
| **Unbuffered** | Both addresses known → copy **directly user → user**, no kernel buffer |
| **Small messages in registers** | Kernel path preserves designated registers → **zero copy** |
| **Big messages by page mapping** | Grant/map a page instead of copying |
| **Combined syscalls** | Client: `call` = send + receive. Server: `sendrecv` = reply + wait for next request → halves the crossings |
- Servers sit in a loop, normally blocked in the kernel waiting for the next request → the fast path applies. The reply follows the same path in reverse, returning from the client's `call`.
- Made people take microkernels seriously again.

## 8. L4Linux: Compatibility
- To be adopted on workstations/servers, a microkernel must run **existing apps** (my editor, my compiler) → must be **compatible with Unix/Linux**.
- Rewriting Linux as many servers = vast work → instead run **the Linux kernel as a user-level L4 task**. (A kernel is just a program — xv6 runs in QEMU, which is user space.)
- Modified Linux's low-level parts (page table updates, processor registers) to use L4 syscalls/IPC; most of Linux unchanged → got its FS, networking, drivers for free.
- **Each Linux process = separate L4 task**. Linux changes a process's page table by sending IPC to L4.
- Process syscalls (`fork`, `read`...) go through a small **library** that turns them into **IPC to the Linux server**, which normally waits in `recv`.
- Linux server is **one L4 thread**; inside, Linux switches among its own per-process kernel threads (like xv6) — but **which user process runs is decided by L4**, so Linux may be handling vi's syscall while L4 runs the shell. Why not one L4 thread per Linux kernel thread? Single-core hardware then, and that Linux version had no multiprocessor support (no spinlocks).
- Downside: Linux **lost control of scheduling** (priorities, fairness) — L4 schedules processes.
- Page-table complication ("dual space mistake"): native Linux mapped kernel + user in **one page table** (fast syscalls, no switch). L4Linux tried to have each process's mappings visible in the Linux server while handling its syscall, but L4 always switches to the task's own page table → they made a kernel-sharing task per process; complicated, didn't work out well.

## 9. Results
- Context: earlier measurements of **Mach** in the same configuration were **much slower** than Unix → many concluded microkernels were hopeless. This paper is the **rebuttal**.
- `getpid`: native Linux **1.7 µs**, L4Linux **~4 µs** — ~2× (two IPC crossings vs one syscall), about the minimum possible.
- **AIM** benchmark (files, processes, lots of syscalls): L4Linux only **a few % slower**. Real programs do work beyond syscalls, so IPC cost matters less. (Paper also has tricks like tagged TLBs/small spaces to avoid page-table switch costs.)

## 10. Where It Ended Up (20 years later)
- L4 widely used in **embedded** systems (no Unix compatibility needed).
- Never caught on for workstations/servers — not because of a flaw, but it wasn't **dramatically better** enough to make people switch.
- Lasting influence:
  - More flexible **VM interfaces** → things like `mmap` in Linux
  - Running an OS on top of another layer → **virtual machine monitors** (cloud)
  - Extensibility → Linux **loadable kernel modules**
  - IPC / client-server services → **macOS**
- Hybrids exist: Mach 2.5 had a full Unix inside the microkernel; macOS is a full kernel with good IPC; Google's Fuchsia uses some of these ideas.
