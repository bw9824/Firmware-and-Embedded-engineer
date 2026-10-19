# LEC06 - Isolation and System Call Entry/Exit

## Big Picture
- A **trap** = transition from user code into the kernel. Happens on:
  - **system calls**
  - **faults** (page fault, divide by zero)
  - **device interrupts**
- Details matter for **isolation/security** and **performance** (some programs trap a lot).
- Lecture = gdb trace of the shell's very first `write("$ ")` going into the kernel and back.

---

## 1. Hardware State at a Trap
| State | Notes |
|---|---|
| 32 user registers | Incl. `sp` (stack pointer is a general register) |
| PC | |
| Mode | user / supervisor |
| `satp` | Page table pointer |
| `stvec` | Address of the kernel's trap handler |
| `sepc` | Saved PC during a trap |
| `sscratch` | Scratch register (very important here) |

## 2. What Must Happen to Run Kernel C Code
1. **Save all 32 user registers** (so the user program can resume transparently — especially for unexpected interrupts)
2. **Save the PC**
3. Switch to **supervisor mode**
4. Switch `satp` to the **kernel page table** (user page table has almost no kernel mappings)
5. Point `sp` to a **kernel stack** (C needs a stack)
6. **Jump to kernel C code**

Design constraints:
- **Security**: can't trust anything from user space — registers may hold malicious values. xv6 doesn't even look at them, just saves them.
- **Transparency**: user code shouldn't notice the trap.
- The rest of the kernel (e.g. `write`'s implementation) must also be written carefully.

## 3. What Supervisor Mode Can Actually Do
More limited than you might think — only two extra things:
1. Read/write **control registers**: `satp`, `stvec`, `sepc`, `sscratch`, ...
2. Use **PTEs without `PTE_U`** (user can only use PTEs with `PTE_U` set).

Supervisor mode **cannot** read/write arbitrary physical addresses — it still goes **through the current page table**. This constrains the trap code a lot.

> 💡 Kernel code in `vm.c` can use physical addresses only because the kernel page table **identity-maps** them. Before the trap code switches to the kernel page table, those mappings don't exist.

## 4. Path Overview
```
sh: write()  → ecall
 → uservec        (trampoline.S, asm)    save regs, switch to kernel
 → usertrap()     (trap.c)               figure out why
 → syscall()      (syscall.c)            dispatch by number
 → sys_write()    ...
 → usertrapret()  (trap.c)               prepare return (C part)
 → userret        (trampoline.S, asm)    restore regs, switch back
 → sret           back to user, after ecall
```

## 5. Before ecall
```asm
# user/usys.S — write() is a tiny library stub
write:
    li a7, SYS_write    # 16
    ecall
    ret
```
- Breakpoint at the `ecall` (address 0xde6 from `sh.asm`).
- `a0 = 2` (fd), `a1` = buffer pointer (contains `"$ "`), `a2 = 2` (bytes).
- PC and `sp` are **small addresses** → user address space.
- QEMU monitor (`Ctrl-a c`, `info mem`) shows the shell's page table — only **6 mappings**:
  - text, data, **guard page** (no `U` → user can't use), stack
  - **trapframe** and **trampoline** at the very top — **no `U` flag**
  - **No kernel mappings** at all.
- `A` = accessed bit, `D` = dirty bit — set by hardware; a smarter OS uses them to choose pages to evict to disk (xv6 doesn't).

## 6. What ecall Does — Only 3 Things
1. Mode: user → **supervisor**
2. **`sepc` ← PC** (0xde6 = address of the ecall)
3. **PC ← `stvec`** (kernel set it to the trampoline, 0x3ffffff000)

After ecall: PC is in the **trampoline page**, but **all registers still hold user values** and `satp` is still the **user page table** → the trap code can't use any register until it saves it.

### Why does ecall do so little?
RISC-V gives the OS **maximum flexibility**. Other OSes may want to:
- Handle some syscalls **without switching page tables** (expensive)
- Map user and kernel in **one page table** → never switch
- Save **fewer registers** for some syscalls
- Use **no stack** for simple syscalls

Some machines did all of this in hardware; RISC-V leaves it to software for speed opportunities.

## 7. The Trampoline Page
- ecall doesn't switch page tables → the first kernel instructions must be **mapped in every user page table**.
- Trampoline = page with `uservec`/`userret`, mapped at the **same virtual address (top)** in **every user page table and the kernel page table**.
- Not `PTE_U` → user code can't touch it.
- We can deduce we're in supervisor mode: executing in a non-U page without crashing.
- "Trampoline": you **bounce** on it from user space into kernel space.

## 8. uservec: Saving Registers
**Problem**: need a free register and a place to save 32 registers.
- Can't write to physical memory directly (supervisor still uses page table).
- Can't switch `satp` first: don't know the kernel page table address, and `csrw satp` needs the value in a register.

**Solution, two parts:**
1. **Trapframe page**: each process has one, mapped in its user page table at **0x3fffffe000** (just below the trampoline). `struct trapframe` (proc.h) has 32 register slots plus 5 kernel values:
```c
struct trapframe {
  /*   0 */ uint64 kernel_satp;    // kernel page table
  /*   8 */ uint64 kernel_sp;      // top of this process's kernel stack
  /*  16 */ uint64 kernel_trap;    // address of usertrap()
  /*  24 */ uint64 epc;            // saved user PC
  /*  32 */ uint64 kernel_hartid;  // CPU core number
  /*  40 */ uint64 ra; ...         // 32 user registers
};
```
2. **`sscratch`** holds the trapframe address (set by the kernel before returning to user). First instruction:
```asm
csrrw a0, sscratch, a0   # swap: a0 = trapframe address, sscratch = user a0 (= 2)
```
Then ~30 `sd` instructions store every register into the trapframe.

> 💡 Why not save on the user stack? The user program may **not have a valid stack** (some languages don't use one, or use a weird format, e.g. stack blocks allocated from the heap). The kernel can't assume anything about user memory.

Then:
```asm
ld sp, 8(a0)     # kernel stack (high in memory, with guard page)
ld tp, 32(a0)    # hartid → tp (RISC-V has no direct way to ask "which core am I")
ld t0, 16(a0)    # address of usertrap
ld t1, 0(a0)     # kernel satp value
csrw satp, t1    # switch to kernel page table
sfence.vma
jr t0            # jump to usertrap()
```
- After `csrw satp`, `info mem` shows the **big kernel page table**.
- **Why no crash after switching page tables?** The trampoline is mapped at the **same VA** in both tables, so the next instruction is still there.

## 9. usertrap() (C code)
Handles **all** traps from user space (syscalls, exceptions, interrupts).
```c
w_stvec((uint64)kernelvec);       // traps while in kernel go to kernelvec
struct proc *p = myproc();        // uses tp (hartid) → per-CPU array
p->trapframe->epc = r_sepc();     // save user PC
if (r_scause() == 8) {            // 8 = system call
    if (p->killed) exit(-1);
    p->trapframe->epc += 4;       // resume AFTER the ecall
    intr_on();                    // allow interrupts during syscalls
    syscall();
}
...
usertrapret();
```
- `stvec` changed because **traps from kernel mode** take a different, simpler path (already on kernel page table/stack).
- `sepc` must be saved in memory: while in the kernel we might **switch to another process**, which could trap and overwrite `sepc`.
- `epc += 4`: `sepc` points at the ecall; without +4 we'd re-execute it.
- The trap hardware **turns interrupts off**; xv6 turns them back on so interrupts are served faster during long syscalls.

## 10. syscall()
```c
int num = p->trapframe->a7;              // 16 = SYS_write
p->trapframe->a0 = syscalls[num]();      // table of function pointers
```
- `sys_write` gets its arguments from the **trapframe** (`a0`, `a1`, `a2`).
- Return value stored into **trapframe `a0`** → becomes `a0` in user space (C convention: return value in `a0`). Here: **2** bytes written.

## 11. usertrapret() (C code)
Sets up everything for the **next** trap and for returning:
```c
intr_off();                              // about to point stvec at user handler
w_stvec(TRAMPOLINE + (uservec - trampoline));
p->trapframe->kernel_satp   = r_satp();
p->trapframe->kernel_sp     = p->kstack + PGSIZE;
p->trapframe->kernel_trap   = (uint64)usertrap;
p->trapframe->kernel_hartid = r_tp();
// sstatus: clear SPP (sret → user mode), set SPIE (interrupts on after sret)
w_sepc(p->trapframe->epc);               // sret jumps to sepc
uint64 satp = MAKE_SATP(p->pagetable);
uint64 fn = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);   // a0 = trapframe, a1 = user satp
```
- Interrupts off: once `stvec` points at the user handler, an interrupt while still in the kernel would go to the wrong handler.
- The page-table switch must happen **in the trampoline** (only code mapped in both tables).

## 12. userret (asm)
```asm
csrw satp, a1          # switch to user page table (trampoline still mapped)
sfence.vma
ld t0, 112(a0)         # saved user a0 (now = return value 2)
csrw sscratch, t0
# ~31 ld instructions: restore user registers from trapframe
csrrw a0, sscratch, a0 # a0 = 2 (return value), sscratch = trapframe address
sret
```
- **`sret`**: switch to **user mode**, **PC ← `sepc`**, **re-enable interrupts**.
- Back at user address 0xdea (the `ret` after `ecall`) with `a0 = 2`.
- `sscratch` keeps the trapframe pointer for the next trap.

> 💡 The very first entry into user space also goes through this path (machine boots in the kernel; `sret` is the only way to get to user mode) — that's how `sscratch` and `stvec` get set initially.

## 13. Takeaways
- Syscalls look like function calls, but the user/kernel transition is **far more complex**:
  - **Isolation**: kernel can't trust anything in user space.
  - **Simple, fast hardware**: RISC-V does the minimum.
- Real OS/CPU designers care a lot about trap **speed**. Think about:
  - Could hardware/software be redesigned to make this simpler or faster?
  - Could a malicious program abuse any of this to break isolation?

> 💡 Many OSes offer **memory-mapped file access** to avoid read/write syscall overhead — the mmap lab implements a version.
