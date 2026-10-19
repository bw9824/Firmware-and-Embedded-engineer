# LEC11 - Thread Switching

## Big Picture
- Why multiple tasks: **time sharing** (many users/processes), **simpler program structure** (e.g. primes with processes), **parallel speed-up** on multicore.
- **Thread** = one **serial execution**; its state = **PC + registers + stack**.
- xv6 switches between processes **indirectly**: user → kernel thread → **scheduler thread** → another kernel thread → user.
- Core mechanism: `swtch()` saves/restores **14 callee-saved registers**.

---

## 1. Threads
- A threading system **interleaves** many threads. Two strategies:
  1. **Multiple CPUs**, one thread each
  2. Each CPU **switches** among threads (needed when threads ≫ CPUs) — this lecture
- xv6 does both: threads run on all cores, each core switches.
- **Shared memory?**
  - xv6 **kernel threads** (one per process, runs its syscalls) **share kernel memory** → need locks.
  - xv6 **user processes**: one thread each, no memory sharing.
  - Linux allows **multiple threads per process sharing memory**.
- Alternatives to threads: **event-driven programming**, **state machines** — more efficient, but threads are the most **programmer-friendly**.

## 2. Challenges
1. **Scheduling**: how to switch, which thread next (xv6 has a scheduler per core)
2. **What to save/restore**, and where
3. **Compute-bound threads** that never give up the CPU

## 3. Compute-Bound Threads: Timer Interrupts
- Per-core hardware generates **periodic interrupts** (e.g. every 10 ms) → kernel gains control even if user code loops forever.
- Kernel handler **yields** the CPU to the scheduler.
- **Preemptive scheduling**: the timer forcibly takes the CPU; the kernel then does a voluntary yield on the process's behalf. (Opposite: voluntary/cooperative scheduling.)

## 4. Thread States
| State | Meaning |
|---|---|
| **RUNNING** | Executing on a CPU now (PC/registers are in CPU hardware) |
| **RUNNABLE** | Not running, but wants to as soon as possible — registers **saved in memory** |
| **SLEEPING** | Waiting for an I/O event (next lecture) |

Timer interrupt + yield: RUNNING → RUNNABLE (save CPU registers to memory). Scheduler picks it later: copy saved registers back into a CPU.

## 5. xv6 Picture (simplified)
```
user:    cc        ls        sh          (user thread: user stack, PC, registers)
         | trap: save user regs in trapframe
kernel:  cc's kernel thread (own kernel stack)
         | switch: save kernel regs in context
         ls's kernel thread → restore ls's context → finish syscall → restore trapframe → ls in user space
```
- Never a direct user→user switch. Always: trap into kernel → kernel thread of old process → kernel thread of new process → return to user.

## 6. Full Picture (with scheduler threads)
```
P1 user ──timer──► P1 kernel thread ──swtch──► scheduler(CPU0) ──swtch──► P2 kernel thread ──► P2 user
                    saves p1->context            loops proc table            restores p2->context
```
- A kernel thread can only `swtch` to **its CPU's scheduler thread**, never directly to another process.
- Scheduler finishes putting P1 aside, scans the process table for a RUNNABLE process (could be P1 again), switches to it.
- **Where is state saved?**
  - User registers → **trapframe** (per process)
  - Kernel thread registers → **`p->context`** (per process; one thread executes in one place → one context)
  - Scheduler registers → **`cpu->context`** (in `struct cpu`, one per core; scheduler has no process)
  - Trapframe holds what's needed for entering/leaving the kernel; context holds what's needed for kernel ↔ scheduler switches.
- Each scheduler has its **own stack**, set up at boot (`start.s`/`entry.S`: the CPU's boot stack).
- `yield` is called **by the kernel**, not by user code: on **timer interrupts**, and when a syscall must **wait for I/O** (`sleep`). The diagram is the same in both cases.

**Facts to keep in mind:**
- A core runs **one thing at a time**: a user thread, a kernel thread, or its scheduler thread. Switching creates the illusion of many.
- A thread runs on **at most one core**; otherwise its state is saved.
- Saved contexts are **always produced by a call to `swtch`** → restoring one = returning from an earlier `swtch` call.
- "Context switch" usually means switching kernel thread ↔ scheduler thread (sometimes used for the whole process-to-process dance, or user ↔ kernel).
- In xv6 a process is either executing in user space, in the kernel, or not at all — call it "two threads" (user + kernel) that never run at the same time.

## 7. struct proc (relevant fields)
```c
struct proc {
  struct spinlock lock;        // protects state (and more)
  enum procstate state;        // RUNNING, RUNNABLE, SLEEPING, ...
  uint64 kstack;               // kernel stack
  struct trapframe *trapframe; // saved user registers
  struct context context;      // saved kernel registers (swtch)
  ...
};
```
- Different kernel threads are distinguished by their **kernel stacks** and by **`myproc()`**: uses `tp` (hartid) to index the per-CPU array that the scheduler fills with the running process.

## 8. Demo: spin.c on One CPU
- Two compute-bound processes (fork, then infinite loops printing `/` or `\` occasionally); xv6 with **one core**. Output alternates in bursts → timer-driven switching.
- Break in `devintr` timer case (`which_dev == 2`) → back in `usertrap` about to call `yield()`.
- `p->name` = "spin", pid 3. `trapframe->epc = 0x62` → an `add` in spin's loop (from `spin.asm`).

### yield()
```c
void yield(void) {
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```
- Lock held: state says RUNNABLE but we're **still running on its stack** → no other core's scheduler may pick it up (two cores on one stack = disaster).

### sched()
- Mostly **sanity checks** + panics (historically one of the most bug-prone parts of xv6), then:
```c
swtch(&p->context, &mycpu()->context);
```
- `cpus[0].context.ra` → points into `scheduler` (found with `x/i` in gdb).

### swtch (kernel/swtch.S)
```asm
swtch:          # a0 = old context, a1 = new context
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0) ... sd s11, ...
    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1) ... ld s11, ...
    ret
```
- **Why no PC?** It's predictably inside `swtch`. What matters is **`ra`** — where `swtch` was called from, i.e. where to continue.
- **Why only 14 registers** (`ra`, `sp`, `s0`–`s11`)? `swtch` is called like a normal C function → the compiler already saved **caller-saved** registers on the stack. Only **callee-saved** ones need saving.
- Before: `sp` = process's kernel stack (high memory), `ra` → `sched`.
- After loads: `sp` → `stack0` (the CPU's **boot stack**, where scheduler runs), `ra` → `scheduler`.
- `ret` → now in the **scheduler thread**, returning from the `swtch` the scheduler called long ago.

### scheduler()
```c
for (;;) {
  intr_on();
  for (p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if (p->state == RUNNABLE) {
      p->state = RUNNING;
      c->proc = p;
      swtch(&c->context, &p->context);
      c->proc = 0;        // back here when p yields
    }
    release(&p->lock);
  }
}
```
- After returning from `swtch`: `c->proc = 0` (forget the process), **release `p->lock`** (acquired in `yield`). Now another core could run that process — safe, since its registers are saved and we're off its stack.
- Finds pid 4 (the other spin), sets RUNNING, `swtch` → `ra` points into **`sched`** → returns into pid 4's old `yield` → `usertrap` → back to user space. (Backtrace shows `usertrap → yield → sched`.)
- If it had stopped in a syscall (waiting for I/O), `ra` would lead back into `sleep` / syscall code instead.

## 9. Why `p->lock` Is Held Across swtch
Lock is acquired by the yielding thread and released by the scheduler (and vice versa when starting a process).
1. **Make yielding atomic** w.r.t. other cores: set RUNNABLE + save registers + stop using its stack. Until all three are done, no other scheduler can see it.
2. **Make starting a process atomic**: holding a spinlock = **interrupts off**. Otherwise a timer interrupt mid-switch (state RUNNING but registers not yet restored) would save garbage registers into the context, overwriting the real ones.

## 10. Q&A Notes
- Only **CPU registers** need saving: everything else (stack, heap) is in memory and stays untouched. Registers are saved because the new thread reuses them.
- `swtch` is **architecture-specific** (x86 might also save floating-point state; xv6 kernel doesn't use FP).
- Written in **assembly** because C can't name `ra` or `sp`.
- User code always runs with interrupts on → preemption is guaranteed. The kernel turns interrupts off only temporarily (e.g. while holding locks). Broken timer hardware → buy a new computer.
- Most switches are **not** timer-driven: syscalls waiting for something (and `exit`) give up the CPU.
- **Linux threads**: roughly, each thread is like a process that **shares the address space** with others; scheduling then works much like xv6.
- **First switch into a new process**: `allocproc` fakes the context — **`ra = forkret`**, **`sp` = top of its kernel stack**. `forkret` releases the scheduler's lock and "returns" via `usertrapret` (trapframe also faked: `epc` 0 and user `sp` for the first process; `fork` copies the parent's).
- `forkret` also initializes the **file system** the first time (reads superblock, replays log) — needs a process context to wait for disk I/O.
