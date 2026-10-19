# LEC19 - Virtual Machines

## Big Picture
- **Virtual machine** = simulation of a computer accurate enough to **run an operating system**.
- Three parts:
  1. **Trap-and-emulate** — how to build a VMM in software (on RISC-V)
  2. **Hardware support** — Intel **VT-x** (+ EPT)
  3. Paper: **Dune** — uses VT-x for ordinary processes, not VMs

---

## 1. Setup and Terms
```
guest:  [ Linux guest kernel | vi  cc ]   [ Windows guest | ... ]
          guest supervisor     guest user
host:   VMM (virtual machine monitor)  — runs in real supervisor mode
        hardware
```
- **VMM** takes the place of the OS kernel; simulates computers for **guest** OSes.
- **Guest supervisor mode / guest user mode** = the VMM's emulation of those modes.
- Classic goal: boot **unmodified** Linux/Windows; the guest can't tell it's virtualized.

## 2. Why VMs
- **Consolidation**: many low-load servers (name server, security server...) on one machine.
- **Cloud** (AWS): rent VMs, not physical machines; decide placement on the fly, pack several customers onto one machine.
- **Kernel development**: xv6 in QEMU; easier debugging (gdb).
- Tricks from the extra layer of indirection: **checkpoint** a whole OS + processes (reliability, debugging, cloning), **migrate** a running guest to another machine.
- Old idea (1960s), now pervasive. A different view of OS topics (memory allocation, scheduling, protection); much OS design/research action has moved **down into VMMs**.

## 3. Goals
- Guest **can't tell** real vs virtual → any OS runs, whatever hardware tricks it uses.
- Guest **can't escape**: only its allotted memory, no devices without permission. Cloud customers supply their own (possibly hacked) OS.
- VMs give **stricter isolation than processes** (processes can kill each other, share files, pipes).
- In practice blurred for performance: Linux and common VMMs have **co-evolved**; Linux may knowingly talk to the VMM (e.g. fast device access) — a controlled exception.

## 4. Option 1: Pure Software Interpretation
- Read each guest instruction, decode, apply its effect to registers simulated in software (like QEMU without acceleration).
- Works, but **orders of magnitude slower** (dozens of host instructions per guest instruction) → not used for production/cloud.

## 5. Option 2: Trap-and-Emulate
- Run guest instructions **directly on the real CPU** (same ISA).
- Can't run the guest kernel in real supervisor mode (it could write `satp` and access any memory). Can't run it as a plain user process either (privileged instructions would crash it).
- → Run the **guest kernel in user mode**. VMM runs in **supervisor mode** (booted as the kernel, or as a **loadable kernel module** in Linux — like Dune).
- VMM loads guest kernel into memory, sets up a page table so guest memory looks like it starts at 0, `sret`s into the guest's first instruction **in user mode**.
- Ordinary instructions (add, load/store, function calls) run at **full speed**.
- **Privileged instruction → trap into the VMM**. RISC-V designers ensured **every** supervisor-only instruction traps in user mode.

**Virtual state** per guest (VMM software variables):
- virtual `stvec`, `sepc`, `scause`, `satp`, `sstatus`, ...
- virtual **mode** (guest supervisor vs guest user)
- virtual **hart number**
- (a separate state structure per guest, like xv6's `struct proc`; a sophisticated VMM can give one guest several cores)

**Emulation examples:**
- Guest reads `sepc` (`csrr a0, sepc`) → trap → VMM decodes it → puts **virtual sepc** into the saved `a0` in its trapframe → `sret` to the next instruction. Guest notices nothing.
- Why not let the guest use the real registers? The VMM needs them. E.g. on a guest syscall, the real `scause` says "illegal instruction / ecall from user"; the guest kernel must see "system call".
- Guest user code executes a privileged instruction → VMM sees virtual mode = user → must deliver a fault to the guest kernel, not emulate.
- Guest kernel executes **`sret`** (privileged → traps) → VMM sets virtual mode = user, sets **real `sepc` = virtual sepc**, switches page tables, real `sret` into guest user code.
- Guest user does **`ecall`** → traps to VMM → VMM simulates a trap into the guest kernel: virtual `sepc` = PC, virtual mode = supervisor, virtual `scause` = syscall, real `sepc` = virtual `stvec` → `sret` into the guest kernel's trap handler.
- Cost: OSes that execute many privileged instructions → many expensive traps. Still widely used for years and liked.

## 6. Trap-and-Emulate: Page Tables
- Guest writes `satp` → trap. VMM must **not** let it use its page table directly (could map any physical memory).
- Two mappings:
  - **Guest page table**: guest virtual → **guest physical** (guest thinks it has e.g. 32 GB starting at 0)
  - **VMM map**: guest physical → **host physical** (real pages allocated to this guest, not contiguous)
- VMM builds a **shadow page table**: guest virtual → host physical (combine both), puts **that** in the real `satp`.
- Guest can only ever reach host pages the VMM allocated to it → **isolation**.
- Guest modifying PTEs directly: RISC-V only promises the MMU sees PTE changes after **`sfence.vma`** — privileged → traps → VMM rescans the guest page table, updates the shadow table, does a real `sfence.vma`.
- Guest's "direct mapping" is direct only in the virtual world.
- VMM keeps shadow tables for non-running guests and updates incrementally; maintaining shadow page tables is a lot of work.

## 7. Trap-and-Emulate: Devices
| Strategy | How | Notes |
|---|---|---|
| **Emulation** | Pretend to be a real, common device. Leave its memory-mapped registers **unmapped** → every access traps → VMM simulates the device | Works for unmodified OSes; **slow** (trap per register access). QEMU's UART for xv6 |
| **Virtual (paravirtual) device** | Interface designed for VMs: command **queues in memory**, few/no MMIO register accesses | Fast; guest needs a driver for it. xv6's **`virtio_disk.c`** — QEMU applies commands to `fs.img`. A standard across VMMs |
| **Pass-through** | Real hardware supports multiple guests (modern **NICs** act as several independent NICs) | Highest performance; guest drivers usually know |
- Most of the difficulty of building a VMM today is in **device emulation/drivers** good enough for real guest OSes.

## 8. Hardware Support: Intel VT-x
Motivation: VMs are pervasive; trap-and-emulate has **many expensive traps**; **x86 was hard to virtualize** (possible, but difficult); make VMMs **faster and easier** to build. ~10 years old, very widely used.

- Virtual state moves **into hardware**: in guest mode the CPU has a **separate set of control registers**. The guest kernel executes privileged instructions **without trapping**, on its own copies; hardware adds checks so it can't escape.
- Terms: guest mode = **non-root**, host mode = **root**.
- **VMCS** (VM control structure) in VMM memory: configuration flags + initial guest register values.
- Instructions: **`VMLAUNCH`** (start a guest from a VMCS), **`VMRESUME`**, **`VMCALL`** (guest intentionally exits to VMM).
- **Device/timer interrupts** force an exit to the VMM → guests can't hog the CPU; VMM time-shares guests.
- Each core has its own root + non-root registers and EPT → different guests on different cores; VMM keeps a per-guest struct.

### EPT (Extended Page Table)
- Guest can load its own `cr3` (x86's `satp`) freely — **no trap**.
- VMM sets up an **EPT** per guest.
- MMU translates **guest virtual → guest physical** (guest's table) → **host physical** (EPT), on every access, in hardware.
- VMM still controls which physical memory the guest uses; **no shadow page tables** needed.
- Cost: TLB miss can need many more memory references (each guest table level walks the multi-level EPT) — caching usually hides it.
- AWS VMs: VT-x + clever device schemes → close to bare-metal speed.

## 9. Dune
- Use VT-x hardware for **ordinary Linux processes**, not VMs. **Dune** = loadable kernel module in Linux.
- A process can switch into **Dune mode**: still a process (Linux syscalls etc.), but isolated by the VT-x machinery instead of only a page table. Runs in **non-root supervisor mode**:
  - Has its **own `cr3` → its own page table** (required)
  - Has its own **supervisor/user mode** split
  - Can't escape: Dune's **EPT** only maps the process's own memory. No extra powers over other processes or the kernel — just more flexible control of its own memory.
- (Dune address space limited by physical address width, which is smaller than virtual — e.g. 36 vs 48 bits.)

### Use 1: Sandboxing untrusted plugins
- Browser (process supervisor mode) runs an untrusted **plugin in guest user mode** with its **own page table** exposing only some pages.
- Plugin's "system calls" trap into the **browser**, not Linux → browser decides what to allow.
- Achievable otherwise in Linux, but Dune is elegant and efficient.
- `fork` from a Dune process yields a non-Dune process — no security hole: the plugin can't call fork itself (its syscalls go to the browser), and re-entering the sandbox needs privileged instructions.

### Use 2: Faster garbage collection
- Concurrent GC traces objects while the program runs; the program may modify an already-traced object.
- With Dune: GC sets up its own page table and reads the **dirty (D) bits** directly with ordinary loads → finds modified pages → rescans those. (Dirty bits are slow/hard to get on ordinary Linux.)
- To terminate: one full pass, then **freeze** other threads, rescan dirty pages, finish, resume.
- Significant speedups for GC-heavy programs.
- Can't combine with sandboxing: guest user mode can't have its own `cr3`.
- Requires VT-x CPU + Dune module; a research project, would need adoption to be relied on.
