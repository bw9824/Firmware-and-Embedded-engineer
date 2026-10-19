# LEC09 - Interrupts

## Big Picture
- **Interrupt** = hardware wants attention (packet arrived, key pressed). Software saves its work, handles it, resumes.
- Uses the **same trap mechanism** as syscalls and page faults, but differs in 3 ways:
  1. **Asynchronous** — the handler may have nothing to do with the currently running process
  2. **Concurrency** — device and CPU run truly in parallel
  3. **Programming devices** — each device has its own (often unclear) manual
- Case study: how `$ ` gets printed and how typing `ls` gets read.

---

## 0. Aside: Memory on a Real Machine (`top` on Athena)
- Almost all memory is **in use** — a large part by the **buffer cache**. An OS doesn't leave memory idle → allocating memory usually means **evicting** something (not cheap).
- **Resident memory** (actually in physical memory) is much smaller than the **virtual address space** → demand paging, shared libraries at work.
- 100+ users, ~950 processes, up 249 days.

## 1. Hardware: Where Interrupts Come From
```
devices (UART, Ethernet, disk...) ──53 lines──► PLIC ──► cores
```
- External devices on the board have interrupt lines into the CPU.
- **PLIC** (Platform-Level Interrupt Controller) routes interrupts to cores:
  - Programmable by the kernel: which core(s), **priorities**
  - If no core can take it (interrupts disabled), PLIC **holds** it
  - A core **claims** an interrupt (so no other core gets it), handles it, then tells PLIC it's **complete**
- PLIC is hardware circuitry, no code.

## 2. Software: Device Drivers
```
   top half    ← read()/write() from processes / rest of kernel
      ↕  queue/buffer
   bottom half ← interrupt handler
```
- **Driver** = code that manages a device; in xv6 all drivers are in the kernel (e.g. `uart.c`).
- **Bottom half** = interrupt handler; runs **in no particular process's context** → can't use `copyin`/`copyout` (current page table may belong to another process).
- **Top half** = called by processes (`read`/`write`); interacts with user processes.
- A **queue** **decouples** them and lets the device run in parallel with the CPU.
- Drivers often make up **more code than the core kernel** (one per device, many devices).

## 3. Programming Devices: Memory-Mapped I/O
- Devices appear at physical addresses chosen by the board maker (SiFive memory map: CLINT 0x2000000, PLIC 0xC000000, UART0...; QEMU's "virt" board places UART elsewhere).
- Normal **load/store** to those addresses read/write the device's **control registers** — with **side effects** (e.g. a store sends a byte).

**UART 16550** (what QEMU emulates for keyboard + console):
| Register | Meaning |
|---|---|
| 0 | Load: **RHR** (receive holding — the received byte); Store: **THR** (transmit holding — byte to send) |
| 1 | **IER** — interrupt enable (receive-ready, transmit-holding-empty) |
| ... | LSR (line status), baud rate, ... |

- UART serializes a byte onto a **transmit line** bit by bit; the other side reassembles it.
- After sending a byte, it raises an **interrupt** ("give me the next one"). It has a ~16-byte FIFO, but the protocol still applies.
- In QEMU: transmit line → console (display); keyboard → receive line.

## 4. RISC-V Interrupt Registers
| Register | Role |
|---|---|
| **SIE** (supervisor interrupt enable) | Separate bits: external (E), software (S), timer (T) |
| **SSTATUS.SIE** | **One bit to enable/disable all interrupts** on this core |
| **SIP** | Which interrupts are pending |
| **SCAUSE** | Why we trapped |
| **STVEC** | Where to jump on trap/interrupt |
Each core has its own copy.

## 5. Setup in xv6
1. **`start.c`** (machine mode): delegate interrupts/exceptions to supervisor mode; set SIE bits (external, software, timer); program timer (timer interrupts arrive in M-mode).
2. **`consoleinit` → `uartinit`**: disable UART interrupts → set baud rate → 8-bit chars → reset FIFOs → **enable transmit + receive interrupts**.
3. **`plicinit`**: tell PLIC to accept **UART (IRQ 10)** and VIRTIO disk interrupts.
4. **`plicinithart`** (on **every core**): "this core wants UART and VIRTIO interrupts" (priority ignored).
5. **`scheduler` → `intr_on()`**: sets **SSTATUS.SIE** → this core can now actually be interrupted.

## 6. Printing `$ ` (output path)
- `init`: `mknod("console")` → `open` gives **fd 0**, `dup` → fd 1, 2. Then fork + exec `sh`.
- To the shell, the console is **just a file** (Unix: device = file).
- `fprintf(2, "$ ")` → one **`write` syscall per character**.
```
sys_write → filewrite()          (file type == device → device's write)
 → consolewrite()                copyin the one char from user
 → uartputc(c)
 → uartstart()
```
**`uartputc`**: transmit buffer of **32 chars**, circular, with **write index** (producer) and **read index** (consumer):
- Empty: `w == r`. Full: `w + 1 == r` (mod 32) → **sleep** (shell put aside, other processes run).
- Otherwise put char at `w`, advance `w`, call `uartstart`.

**`uartstart`**: "kick" the device:
- If UART is still busy sending → return.
- Else take char at `r`, write it into **THR**.
- Syscall returns; shell continues (e.g. calls `read`) while the UART sends in parallel.

## 7. What Hardware Does on an Interrupt
If SSTATUS.SIE is set and PLIC routes an interrupt to this core:
1. **Clear SIE** (no nested interrupts until re-enabled)
2. **`sepc` ← pc**
3. Save current mode
4. Mode ← **supervisor**
5. **pc ← `stvec`** (`uservec` if in user mode, `kernelvec` if in kernel)
→ `usertrap()` (same path as syscalls).

## 8. Handling It
```
usertrap() → devintr()
  scause says external interrupt
  irq = plic_claim()           // which device? → 10 = UART
  if (irq == UART0_IRQ) uartintr();
  plic_complete(irq);
```
**`uartintr`**:
- Read RHR: if a char arrived → `consoleintr(c)`. Here none (we're transmitting) → `-1`.
- Call **`uartstart`** → send the next buffered char (the space — the shell's 2nd `write` likely ran concurrently).

## 9. Interrupts and Concurrency
1. **Device and CPU run in parallel** → **producer/consumer** parallelism via the buffer.
2. **An interrupt can stop the current program** between any two instructions:
   - User code: fine — state is saved and restored.
   - **Kernel code** can be interrupted too → sometimes the kernel must **disable interrupts** to make a code sequence atomic.
3. **Top and bottom half may run in parallel on different cores** (one core in `uartputc`, another in `uartintr`, both touching the buffer) → protect with **locks** (next lecture).

**Producer/consumer buffer**:
```
[ $ ][ ␣ ][   ][   ] ...  (32 slots, circular)
  ↑r       ↑w
producer (uartputc) advances w; stops when w+1 == r (full)
consumer (interrupt → uartstart) advances r; stops when r == w (empty)
```

## 10. Reading `ls` (input path)
- Shell calls `read` → `fileread` → **`consoleread`**: input buffer of **128 chars**. Here the **shell is the consumer**, the **keyboard the producer**. Buffer empty → **sleep**.
- User types `l` → UART receives it → interrupt → PLIC → some core → `devintr` → `uartintr` reads RHR = `'l'` → **`consoleintr('l')`**:
  - Special keys: `Ctrl-P` (print process list), `Ctrl-H` / `Ctrl-U` (backspace / kill line)
  - **Echo** the char back to the console (so you see what you type)
  - Put it in the input buffer
  - On **newline**: **wakeup** the process sleeping in `consoleread`
- Again: buffer decouples producer and consumer so each runs at its own speed; one blocks only when the buffer is full/empty.

## 11. Evolution: Interrupts vs Polling
- Early Unix: interrupts were **relatively fast** → hardware could be simple; the CPU did the work.
- Today: interrupts are **slow relative to the CPU** (save/restore registers, entry/exit). Devices do much more work themselves before interrupting.
- Example: gigabit Ethernet with minimum-size (64 B) packets → ~**1.5 million packets/s** → less than **1 µs** of instructions per packet if one interrupt each. Not enough.
- **Polling**: CPU repeatedly reads the device's status register.
  - Wastes CPU cycles when nothing arrives (couldn't run other processes meanwhile)
  - But for a **very fast device**, avoids interrupt entry/exit cost
- Rule: **slow device → interrupts**; **fast device → polling**. Sophisticated drivers (high-performance NICs) **switch dynamically** between the two.

## 12. Q&A Notes
- One UART shared by all cores → one buffer → **lock** serializes `uartputc`/`uartintr` from different cores.
- `sleep(chan, lock)` **releases the lock** while sleeping and re-acquires it on wakeup; `wakeup(chan)` wakes processes sleeping on that address. ("Conditional synchronization")
- `printf` from multiple processes interleaves: locking is per `putc`, so a whole `printf` is **not atomic**.
- **Timer interrupts**: CLINT interrupts go to **machine mode** → `timervec` (asm) reprograms CLINT for the next tick, then raises a **supervisor software interrupt** → after `mret`, kernel takes it via `kernelvec`/`usertrap` and treats it as a timer tick. (Just how this chip works — delegating timer interrupts directly to S-mode isn't possible here.)
