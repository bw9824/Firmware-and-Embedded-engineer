# LEC05 - Calling Conventions and Stack Frames RISC-V

## Big Picture
- A practical lecture: RISC-V assembly, the **calling convention**, and the **stack**.
- Needed for the traps lab: you'll work closely with **trapframes**, the **stack**, and assembly.
- **gdb** is the main tool for inspecting all of this.

---

## 1. From C to the Processor
```
C → compiler → assembly (.s) → assembler → object/binary (.o) → processor
```
- Processors don't understand C — only the **binary encoding** of their **ISA** (instruction set architecture). Each instruction has an opcode.
- Assembly has no structured control flow (no loops); functions are just **labels**.
- Any compiled language (C++, ...) ends up as the same assembly.
- `kernel/kernel.asm` = the whole xv6 kernel in assembly, each instruction with its address.

## 2. RISC-V vs x86-64
| | RISC-V | x86-64 |
|---|---|---|
| Type | **RISC** — reduced instruction set | **CISC** — complex instruction set |
| Size | 2 manuals (~240 + ~135 pages) | 3 books, >15,000 instructions (added ~3/month since the 70s) |
| Instructions | Simple, each does one small thing, few cycles | Many do several operations at once |
| Openness | **Open source** ISA (from a UC Berkeley research project) | Proprietary (Intel, AMD) |
| Why big | — | **Backwards compatibility** (never deprecate), SIMD instructions |

- Neither is fundamentally better; it's a design trade-off.
- RISC is everywhere: **ARM** (Qualcomm Snapdragon in Android phones, Apple chips) is also RISC.
- RISC-V = **base integer ISA** + optional **extensions** (e.g. F = single-precision floating point). The compiler only emits instructions for extensions the processor supports.

## 3. Reading Assembly
```asm
.section .text      # this is code (text segment)
.global sum_to      # can be called from other files (declared in defs.h)
sum_to:
    mv t0, a0       # t0 <- n
    li a0, 0        # a0 <- 0 (sum)
loop:
    add a0, a0, t0
    addi t0, t0, -1
    bnez t0, loop
    ret
```
- Compilers **optimize** a lot → real compiled code may look different; in gdb you may see "variable **optimized out**" (e.g. kept only in a register).
- Kernel addresses look like `0x80...` → search them in `kernel.asm` to find the exact instruction.

## 4. Registers
- Assembly operates on **registers**, not memory. Pattern: **load** into register → **operate** → **store** back.
- Registers are the fastest storage → use memory only when out of registers.
- Always refer to them by **ABI name**.

| ABI name | Use | Saver |
|---|---|---|
| `ra` | Return address | Caller |
| `sp` | Stack pointer | Callee |
| `gp`, `tp` | Global / thread pointer | — |
| `t0`–`t6` | Temporaries | Caller |
| `s0`/`fp` | Saved register / **frame pointer** | Callee |
| `s1`–`s11` | Saved registers | Callee |
| `a0`–`a1` | Function args / **return values** | Caller |
| `a2`–`a7` | Function args | Caller |

- **Caller-saved**: NOT preserved across a call — the called function may overwrite them. If the caller needs them afterwards, it saves them.
  - e.g. `ra` must be caller-saved: every function call overwrites it.
- **Callee-saved**: preserved — the called function must restore them before returning.
- More than 8 args → extra args go on the stack.
- All registers are 64 bits; smaller types are sign- or zero-extended.
- A value twice the pointer size (128 bits) is returned in the pair **`a0` + `a1`**.

> 💡 Why are `s1` and `s2`–`s11` not adjacent? Likely because of **compressed (16-bit) instructions**, which can only use registers x8–x15 (`s0`, `s1`, `a0`–`a5`). Normal RISC-V instructions are 32 bits.

## 5. The Stack
```
high addresses
 +-----------------+
 | caller's frame  |
 +-----------------+ <- fp (s0): top of current frame
 | return address  |    fp - 8
 | previous fp     |    fp - 16
 | saved registers |
 | local variables |
 +-----------------+ <- sp: bottom of stack
low addresses   (stack grows DOWN)
```
- Each function call creates its own **stack frame** by moving `sp` (subtraction = grow).
- Frames hold saved registers, local variables, extra arguments; sizes differ per function.
- Always at **fixed offsets from `fp`**: the **return address** and the **previous frame pointer**.
- Previous fp lets you walk back: on return, restore fp → points to the caller's frame.
- The compiler generates frames following the calling convention:
```
prologue   (make frame, save ra / registers)
body
epilogue   (restore, remove frame, ret)
```

## 6. Leaf vs Non-Leaf Functions
- **Leaf function**: calls no other function → no need to save `ra` or caller-saved registers (e.g. `sum_to`).
- **Non-leaf** (`sum_then_double` calls `sum_to`):
```asm
sum_then_double:
    addi sp, sp, -16    # prologue: make frame
    sd   ra, 0(sp)      # save return address
    call sum_to
    li   t0, 2
    mul  a0, a0, t0
    ld   ra, 0(sp)      # epilogue: restore ra
    addi sp, sp, 16
    ret
```
- **Demo: remove prologue/epilogue** → `call sum_to` overwrites `ra` with `sum_then_double+4` → `ret` jumps back into `sum_then_double` → **infinite loop**. Seen in gdb by watching `ra` in `layout reg`.
- Why `-16`? Frame needs 8-byte slots (64-bit registers); compilers normally use 16 to hold `ra` + previous `fp`.

## 7. Structs in Memory
- A struct is a **contiguous** region of memory; fields laid out in order, like an array whose elements can have different types (with possible alignment padding).
- Usually passed to functions **by reference** (pointer).
- Example: `struct person { int id; int age; }` → in gdb `p *p` shows `{id = 1215, age = 22}`; `x/2x p` shows the raw memory.

## 8. gdb Cheat Sheet
| Command | What it does |
|---|---|
| `tui enable` | Text UI with source window |
| `layout asm` / `layout reg` / `layout split` / `layout source` | Show assembly / registers / C+asm / C |
| `focus reg` | Choose which window arrow keys scroll |
| `b sum_to`, `b *0x80...` | Breakpoint |
| `b sum_to if i == 5` | **Conditional** breakpoint |
| `watch i` | **Watchpoint** — stop when value changes |
| `info break` | List breakpoints (and hit counts) |
| `delete` | Delete all breakpoints |
| `info reg` (`i r`) | Registers |
| `info frame` (`i frame`) | Current frame: level, saved pc, caller frame, args, saved registers |
| `bt` | **Backtrace**: whole call stack (e.g. `usertrap → syscall → sys_demo → demo4 → dummymain`) |
| `frame 3` | Switch to another frame to inspect it |
| `info args`, `info locals` | Arguments, local variables |
| `p *argv@argc` | Print array elements |
| `apropos tui` | Search gdb help |

> 💡 The traps lab asks you to write your own **backtrace** by walking the frame-pointer chain.

> 💡 **tmux**: `Ctrl-b c` new window, `Ctrl-b n/p` next/previous, `Ctrl-b %` / `Ctrl-b "` split, `Ctrl-b o` switch pane — handy to run QEMU and gdb side by side.
