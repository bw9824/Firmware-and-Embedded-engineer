
---

## 1. Why Isolation
- A bug in one app (e.g. your `primes`) must not affect other apps — especially not the shell you'd use to kill it.
- A bug or strange argument passed to the OS must not crash the OS.

### Strawman: no OS (OS as a library)
Apps talk to hardware directly:
- **CPU**: apps must voluntarily give up the CPU (**cooperative scheduling**). One infinite loop → nobody else runs, not even the program that could kill it. → No **enforced multiplexing**.
- **Memory**: all apps share physical memory with no boundary. `echo` writing to address 1000 can overwrite the shell's memory → bugs leak between apps.

→ The main reason to have an OS: **enforce multiplexing and strong memory isolation**. (OS-as-library is only seen in some real-time systems where apps trust each other.)

## 2. The Unix Interface Is Designed for Isolation
The interface **abstracts** hardware so the kernel can sit in between:

| Abstraction | Instead of | Lets the kernel |
|---|---|---|
| **Process** (`fork`) | CPU | Time-multiplex cores among processes |
| **`exec`** memory image (+ `sbrk` to grow) | Physical memory | Control which physical memory each app uses |
| **Files** | Disk blocks | Decide block placement; stop user A from reading user B's data |

- Example: the lab's RISC-V board has 4 cores. With 7 processes, the kernel runs one on a core ~100 ms, then switches — no process can run longer than that.
- An app can't ask "give me physical addresses 1K–2K" — there's no such interface.

## 3. The OS Must Be Defensive
- Apps may be buggy or **malicious** (written by an attacker).
- A bad argument must not crash the kernel (that would be denial of service for everyone).
- An app must not **break out of isolation** and take control of the kernel — then it controls all hardware.
- Hard to get right: Linux still periodically has such bugs.
- Requires a **strong wall** between apps and OS → provided by hardware.

## 4. Hardware Support 1: User Mode / Kernel Mode
- CPU has two modes (RISC-V calls kernel mode **supervisor mode**; also has a third, **machine mode**, mostly ignored in this course).

| Mode | Can execute |
|---|---|
| User | **Unprivileged** instructions: add, sub, branches, jumps, procedure calls... |
| Kernel | + **Privileged** instructions: set page-table register, disable clock interrupts, manipulate protection state |

- The mode is literally **a bit in the processor**. When decoding a privileged opcode in user mode → refuse (like divide-by-zero) → **trap into the kernel**, which may kill the app.
- The instruction that sets the mode bit is itself privileged → user code can't switch itself to kernel mode.
- Privileged instructions are listed in the RISC-V privileged architecture document.

## 5. Hardware Support 2: Page Tables (Virtual Memory)
- Page table maps **virtual addresses → physical addresses**.
- **Each process has its own page table** → it can only access physical memory that appears in its table.
- If the OS gives processes **disjoint** physical memory, a process can't even *form* an address pointing to someone else's memory.
- Each process gets its own address space starting at 0 (`ls` and `echo` both have address 0, mapped to different physical memory). The kernel has its own address space too.

```
| ls: 0 ... 2^n |   | echo: 0 ... 2^n |       user mode
-------------------------------------------------------
|              kernel (own address space)  |   kernel mode
```

## 6. Entering the Kernel: `ecall`
- Boxes are too strict — apps need to call `read`, `write`, `fork`, `exit`.
- RISC-V **`ecall <n>`**: enters the kernel **at one location controlled by the kernel**; `n` = system call number.
- User code never calls the kernel's `fork` function directly:
```
user:   fork()  → stub: ecall with SYS_fork
kernel: syscall() (syscall.c)  reads the number from a register,
        dispatches to sys_fork()
```
- These user-side wrappers are called **system call stubs**.
- The kernel can check anything it wants in the syscall implementation — e.g. `write` must verify the buffer address really belongs to the app, so the kernel isn't tricked into touching other memory.
- To take back the CPU from an infinite loop: kernel programs a **timer**; when it fires, control transfers to the kernel, which can switch processes.

> 💡 Why C? It gives low-level control of hardware (you can cast anything to anything, easy to program e.g. a timer chip). Linux avoids C++ largely by preference.

## 7. Kernel = Trusted Computing Base (TCB)
- The kernel **must be correct**: a bug may be turned into an **exploit** by an attacker to break isolation or take over.
- Must treat user processes as potentially malicious → **security mindset**.
- Hard with a huge kernel; widely used OSes regularly have security bugs.
- → Question: **what should run in kernel mode?**

## 8. Monolithic Kernel vs Microkernel
**Monolithic** (most Unix, Linux, xv6): the whole OS runs in kernel mode.
- ➖ Lots of code in kernel → more bugs (roughly a few bugs per few thousand lines) → more chance of a security bug.
- ➕ **Tight integration** between subsystems (FS, VM, processes) → great performance (e.g. FS and VM share a page cache).

**Microkernel**: run as little as possible in kernel mode.
- Kernel keeps only IPC/message passing, minimal VM (page tables), CPU multiplexing.
- File server, parts of VM etc. run as **normal user processes**.
- Shell calling `exec` → message to kernel → kernel forwards to file server → reply back → 2× the user/kernel crossings vs monolithic.
- ➕ Small kernel → hopefully fewer bugs.
- ➖ Performance: more crossings, and tight integration (sharing a page cache) is harder.

In practice: desktops and data centers mostly monolithic (Linux performance, history); embedded often microkernels (Minix, seL4). Rewriting Linux into a microkernel would be huge work for little incentive.

## 9. xv6 Code Organization and Build
- `kernel/` — all kernel files, compiled into **one binary `kernel`** (monolithic) that runs in kernel mode.
- `user/` — programs that run in user mode.
- `mkfs/` — builds an initial file system image (`fs.img`).

Build:
```
proc.c → gcc → proc.s (RISC-V asm) → assembler → proc.o
pipe.c → ...                                   → pipe.o
all .o → linker (loader) → kernel
```
- Makefile also produces **`kernel.asm`**: the whole kernel disassembled, with each instruction's binary encoding — useful for debugging.
- First kernel instruction is at **`0x80000000`** (address agreed with QEMU; `kernel.ld` tells the linker to place the kernel there).

## 10. QEMU
- Command line passes: `-kernel kernel`, memory size (`-m`), number of cores, the disk image `fs.img`.
- Think of QEMU as **a real RISC-V board** (like a SiFive board with 4 cores, L2 cache, DRAM, UART connected to keyboard/display, Ethernet, PCIe).
- Internally QEMU is a C program running an **infinite loop**: fetch an instruction → decode opcode → execute in software, keeping register state (x0, x1, ...) in C variables. Emulates **privileged instructions** too.
- Uses multiple host threads → emulated cores really run **in parallel**.

## 11. Boot Walkthrough (gdb)
1. Execution starts at `_entry` (`entry.S`) at 0x80000000 — **machine mode**, no paging, no isolation.
2. xv6 jumps to **supervisor mode** as quickly as possible → `main()`.
3. `main()` initializes, in order (order matters for some):
   - `consoleinit` (now can print "xv6 booting")
   - page allocator, virtual memory (kernel page table, turn on paging)
   - process table, trap code (user/kernel transition)
   - PLIC (interrupt controller, for disk and console)
   - buffer cache, inode cache, file system, disk
4. `userinit()` creates the **first process**.

## 12. The First System Call
- `userinit` uses a tiny program **`initcode`** (assembly, statically included in the kernel) because the file system can't run programs yet:
```asm
la a0, init        # "/init"
la a1, argv
li a7, SYS_exec    # syscall number in a7
ecall
```
- `ecall` → kernel → **`syscall()`**: `num = p->trapframe->a7` → **7 = `SYS_exec`** (from `kernel/syscall.h`).
- `syscalls[num]()` — an **array of function pointers** indexed by syscall number → `sys_exec()` (in `sysfile.c`).
- `sys_exec` **copies arguments from user space into kernel space** (path = "/init").
- `/init` opens the console, `dup`s it to fds 0/1/2, **forks**, and the child **execs `sh`** → the shell runs.
