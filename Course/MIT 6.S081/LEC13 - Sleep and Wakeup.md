# LEC13 - Sleep and Wakeup

## Big Picture
- **Coordination**: a thread needs to **wait for a specific event** (pipe not empty, disk read done, child exited). xv6's tool: **`sleep` / `wakeup`**.
- Central danger: the **lost wakeup** problem → why `sleep` takes a **lock** argument.
- Also: how threads **exit** and how **kill** works safely.

---

## 1. Two Rules From Thread Switching
**Rule 1: hold `p->lock` across `swtch`.**
- Process acquires its own lock, sets its state, calls `sched`/`swtch`; the **scheduler thread releases** it.
- Prevents another core's scheduler from running this process while it's still running on its stack (two cores, one stack → crash). Works because all cores share one memory system.

**Rule 2: hold NO other lock when calling `swtch`.**
- One core: P1 acquires lock L (e.g. disk/UART) and switches away. P2 runs, tries to acquire L → **spins forever**.
- Timer interrupt can't save us: `acquire` **turns interrupts off before spinning** (needed to avoid interrupt-handler deadlocks), so P2 never yields back to P1 → **deadlock**.
- Same kind of deadlock possible on multicore with several locks. `sched()` checks that no other locks are held.

## 2. Why Coordination
- Locks: threads **don't have to think** about each other.
- Coordination: threads **explicitly interact** — wait for an event caused by I/O or another thread.
- **Busy waiting** (`while (buffer empty) ;`) is OK only if the event is guaranteed within ~a microsecond (some device hardware). Otherwise it wastes CPU → want to **give up the CPU and get it back when the event happens**.
- Many coordination primitives exist; xv6 (like many Unixes) uses **sleep/wakeup**.

## 3. Example: UART Write (modified driver for the demo)
```c
void uartwrite(char buf[], int n) {
  acquire(&uart_tx_lock);
  for (int i = 0; i < n; i++) {
    while (tx_done == 0)
      sleep(&tx_chan, &uart_tx_lock);   // wait for "ready for next char"
    WriteReg(THR, buf[i]);
    tx_done = 0;
  }
  release(&uart_tx_lock);
}

void uartintr(void) {
  acquire(&uart_tx_lock);
  if (ReadReg(LSR) & LSR_TX_IDLE) {
    tx_done = 1;
    wakeup(&tx_chan);
  }
  release(&uart_tx_lock);
}
```
- UART accepts one char at a time and is slow (~1000 chars/s → ~1 ms between chars = a million cycles). Instead of spinning, sleep; UART interrupts when done.
- **Sleep channel** (`&tx_chan`): any 64-bit value; `wakeup` with the same value wakes those sleepers. Sleep/wakeup don't know what the event is → **flexible**.
- `tx_done`: flag from interrupt routine to writer ("previous char finished").
- (A real driver hands the UART 16 chars at a time; fast devices take many bytes per interrupt.)

## 4. The Lost Wakeup Problem
Imagine a simpler `broken_sleep(chan)`:
```c
void broken_sleep(void *chan) {
  p->state = SLEEPING;
  p->chan = chan;
  sched();
}
void wakeup(void *chan) {
  for each p in proc table:
    if (p->state == SLEEPING && p->chan == chan)
      p->state = RUNNABLE;
}
```
Where to put the lock in `uartwrite`?
- Hold it across the whole loop → interrupt routine can never acquire it to set `tx_done` → **deadlock**.
- Release before sleeping, reacquire after:
```c
while (tx_done == 0) {
  release(&uart_tx_lock);
  // <-- interrupt on another core: sets tx_done = 1, wakeup() finds nobody sleeping!
  broken_sleep(&tx_chan);   // sleeps forever
  acquire(&uart_tx_lock);
}
```
- **Demo**: `init` prints a few chars, then **hangs**. Typing a key un-sticks it (UART's single interrupt routine for input also calls `wakeup`) — by luck. `cat README` stalls every few characters.
- Need to close the window between **releasing the condition lock** and **marking the process SLEEPING**.

## 5. The Fix: sleep(chan, lock)
Rules:
1. There's a **condition** (`tx_done == 1`) protected by a **condition lock** (`uart_tx_lock`).
2. Caller **holds the condition lock** while checking the condition and when calling `sleep`, and **passes the lock** to `sleep`.
3. **`wakeup` must be called with the condition lock held.**

`sleep` makes "release condition lock + mark asleep" **atomic w.r.t. wakeup**:
```c
void sleep(void *chan, struct spinlock *lk) {
  struct proc *p = myproc();
  acquire(&p->lock);      // FIRST take p->lock
  release(lk);            // THEN release condition lock
  p->chan = chan;
  p->state = SLEEPING;
  sched();                // scheduler releases p->lock after switch
  p->chan = 0;
  release(&p->lock);
  acquire(lk);            // reacquire condition lock before returning
}

void wakeup(void *chan) {
  for (p = proc; p < &proc[NPROC]; p++) {
    if (p != myproc()) {
      acquire(&p->lock);
      if (p->state == SLEEPING && p->chan == chan)
        p->state = RUNNABLE;
      release(&p->lock);
    }
  }
}
```
Why it works: from checking the condition until the scheduler has switched away, the sleeper **always holds either the condition lock or `p->lock`**. `wakeup` needs the condition lock (to be called) and `p->lock` (to inspect the process) → it can only see the process **after** it's fully asleep.

## 6. Always Sleep in a Loop
```c
// pipe.c: piperead
acquire(&pi->lock);
while (pi->nread == pi->nwrite && pi->writeopen) {
  if (myproc()->killed) { release(&pi->lock); return -1; }
  sleep(&pi->nread, &pi->lock);
}
```
- `pipewrite` on another core adds bytes and calls `wakeup(&pi->nread)` — the pipe lock stops it from slipping in between the check and the sleep.
- Several readers, writer writes **1 byte** → wakeup wakes **all** of them. Each `sleep` must reacquire the condition lock before returning, so they return one at a time: the first reads the byte; the rest re-check, see it empty, sleep again.
- → Wakeups aren't precise; **every sleep in xv6 is wrapped in a loop re-checking the condition.**

## 7. Sleep/Wakeup vs Semaphores
- Sleep/wakeup interface is a bit complex (must reveal the lock), but **general** — any condition works.
- **Semaphores** (P/V, up/down counters): simpler interface, lost-wakeup handling hidden inside the implementation — but only fit if you're waiting on a **count**.

## 8. Exiting a Thread
Two problems:
1. Can't unilaterally destroy another thread: it may be running on another core, using its stack, holding locks, halfway through updating kernel data.
2. A thread that exits can't free resources **it still needs to execute** (its kernel stack, its proc slot).

### exit()
- Close open files (may involve file-system work), release current directory.
- **Reparent children to `init`** (pid 1) — every exiting process must be waited for.
- Wake up the parent (possibly sleeping in `wait`).
- State = **ZOMBIE**, call `sched()` — never runs again, but not fully freed.
- (Messy code handles a parent and child exiting **concurrently**. Order of setting ZOMBIE vs waking parent doesn't matter: the child holds its `p->lock` until the scheduler releases it, and the parent's `wait` needs that lock.)

### wait()
- Scan proc table for children of the caller in state **ZOMBIE**.
- Collect the exit status; call **`freeproc`**: free trapframe, user page table, ... → state **UNUSED** (slot reusable by `fork`). (Kernel stacks aren't freed in xv6 because of the guard-page layout.)
- **Key trick**: the **parent** frees the resources the child can't free while running.
- `wait` isn't just a convenience — **every process must be waited for**. `init` calls `wait` in a loop for orphans. `init` must **never exit** (exit panics if it's init).

## 9. kill()
- Can't stop the target immediately (might be mid-update in the kernel).
- `kill(pid)` only:
  - sets **`p->killed = 1`**
  - if **SLEEPING** → make it **RUNNABLE** (so it returns from `sleep`)
- Target **checks `killed` at safe points** and calls `exit()` itself:
  - in `usertrap` **before** a syscall and **after** a syscall/interrupt (e.g. a timer interrupt while in user space) — no locks held, nothing half done
- So there may be a **delay** between `kill` and actual exit.
- Sleeping in a syscall:
  - Loops where bailing out is safe check `killed` (e.g. `piperead`, console read — otherwise a process waiting for keyboard input might wait until tomorrow) → return error → `usertrap` sees `killed` → exit.
  - Loops where bailing out is **not** safe don't check (e.g. waiting for a **disk read** in the middle of creating a file) → finish the whole FS operation first.
- xv6 has no permission check; Linux requires the same **user ID** to kill another process.

## 10. Q&A Notes
- Shutting down: the important part is leaving the **on-disk file system consistent** (crash recovery, LEC15); beyond that, depends on what services the machine provides.
