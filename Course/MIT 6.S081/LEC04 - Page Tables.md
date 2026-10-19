# LEC04 - Page Tables

## Big Picture
- Goal: **memory isolation** — each app (and the kernel) gets its **own address space**.
- Mechanism: **page tables**, walked by hardware (the **MMU**).
- Page tables are a **level of indirection fully controlled by the OS** → enables many tricks later (page faults, COW...).
- This lecture: mechanisms + xv6's kernel address space setup.

---

## 1. The Problem
- All programs (kernel, shell, cat) live in one physical memory (DRAM).
- If `cat` has a bug and stores `7` to address 1000, which belongs to the shell → shell's memory is corrupted.

**Address spaces**: give every program (including the kernel) its own address space starting at 0.
- `cat` writing to 1000 writes **its own** 1000.
- `cat` can't even **name** an address belonging to the shell.
- Challenge: **multiplex** many address spaces onto one physical memory.

> 💡 Physical memory can run out: `kalloc` keeps a list of free pages and returns null when empty; the OS should propagate the error to the app gracefully.

## 2. Paging Hardware: MMU + satp
```
CPU --virtual address--> MMU --physical address--> memory
                          |
                     satp → page table (stored in RAM)
```
- Once the MMU is on, **every address** in an instruction is a **virtual address**.
- The page table (map) is stored **in memory**; the MMU just walks it.
- **`satp`** register holds the **physical address** of the root of the map.
- **Each process has its own map**; on a process switch the kernel switches `satp`.
- Writing `satp` is **privileged** — otherwise a user app could install its own map and break isolation. The kernel keeps each process's page-table pointer in its `struct proc`; there's one `satp` per core.

## 3. Pages and Address Format
- One entry per address would need 2^64 entries → impossible. → **Translate per page**: RISC-V page = **4 KB = 4096 bytes** (common on almost all processors).
- Virtual address = **index** (which page) + **offset** (byte within page, 12 bits since 2^12 = 4096).

**RISC-V (Sv39) on our processor:**
```
Virtual  (64 bits): | unused (25) | index (27) | offset (12) |   → 2^39 = 512 GB address space
Physical (56 bits): | PPN (44)                 | offset (12) |
```
- Top 25 bits unused → room for bigger address spaces in future processors.
- 56-bit physical addresses chosen by hardware designers (technology trend prediction + margin). Physical memory may be bigger than one virtual address space, and vice versa.

## 4. Three-Level Page Table
A flat table with 2^27 entries per process would still fill memory. Real RISC-V structure:
```
VA index (27 bits) = | L2 (9) | L1 (9) | L0 (9) |

satp → [top directory] --L2--> [middle directory] --L1--> [bottom directory] --L0--> PPN + offset
```
- Each directory = **one 4 KB page** of **512 PTEs** (4096 / 8 bytes).
- Each PTE in a directory holds the **physical page number** of the next level.
- **Advantage**: unused parts of the address space need **no directories at all**. An address space with only page 0 needs just **3 directories** (one per level), instead of 2^27 entries.
- Directories store **physical** page numbers (and `satp` holds a physical address) — otherwise translation would depend on another translation (recursive).
- Next directory address = **44-bit PPN + 12 zero bits** → every directory is **page aligned**.
- Translation fails (invalid entry) → hardware refuses → **page fault** (handled by the OS, later lecture).

## 5. Page Table Entry (64 bits)
```
| reserved (10) | PPN (44) | flags (10) |
```
| Flag | Meaning |
|---|---|
| **V** | Valid — entry can be used for translation |
| **R** | Can read |
| **W** | Can write |
| **X** | Can execute instructions |
| **U** | Accessible from user mode |

(Other flag bits matter less for now.)

## 6. TLB
- Naively every memory access = **3 extra memory reads** (one per level) → expensive.
- **TLB (Translation Lookaside Buffer)**: cache of recent VA → PA translations. First access walks the table; later ones hit the TLB.
- Implementation details are hidden in the processor. What the OS must know: when **switching page tables**, tell the CPU to **flush** stale TLB entries → RISC-V instruction **`sfence.vma`**.
- The 3-level walk is done **in hardware**. Each core has its own MMU and TLB.
- CPU caches: some indexed by virtual address (before MMU), some by physical (after MMU).

> 💡 xv6 also has a **software** walk (`walk()`): needed to **build** page tables, and to translate user pointers in the kernel (`copyin`/`copyout` use the user page table to find the physical address).

## 7. Physical Address Layout (decided by the board)
- Determined by **board designers**, not the OS (documented in the SiFive board manual's memory map).
- A **demultiplexer / memory controller** inside the chip routes each physical address to DRAM or a device.

| Physical address | What |
|---|---|
| 0x0 | Unused |
| 0x1000 | **Boot ROM** — runs first at power on, then jumps to 0x80000000 |
| 0x02000000 | **CLINT** (interrupt-related) |
| 0x0C000000 | **PLIC** (interrupt controller) |
| 0x10000000 | **UART0** (console / display) |
| 0x10001000 | **VIRTIO disk** |
| 0x80000000 | **DRAM** starts (KERNBASE); xv6 uses up to 128 MB (PHYSTOP) |

- Addresses below 0x80000000 are **devices**: load/store there talks to the device = **memory-mapped I/O**.

## 8. xv6 Kernel Virtual Address Space
- At boot, paging is off. `kvminit` builds the kernel's page table.
- Mostly **identity mapping**: VA == PA (keeps xv6 simple).
- Exceptions:
  - **Kernel stacks** mapped **high**, each with an **unmapped guard page** below (PTE invalid). Stack overflow → page fault → immediate panic, instead of silent corruption. Guard page uses **no physical memory**.
  - So each kernel stack page is **mapped twice** (high address + its identity address). Page tables allow one-to-one, one-to-many, many-to-one mappings.
  - **Trampoline** page at the top (next lecture).
- Permissions:
  - Kernel **text: R-X** (can't write code → catch bugs early)
  - Kernel **data: RW-** (can't execute data)
- Free memory (between end of kernel and PHYSTOP) holds user processes' page tables, text, data. Running out → `fork`/`exec` return error.
- Every process has a kernel stack.

## 9. Code Walkthrough: kvminit
```c
kernel_pagetable = (pagetable_t) kalloc();   // top-level directory
memset(kernel_pagetable, 0, PGSIZE);         // all PTEs invalid
kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W); // map devices one by one
kvmmap(VIRTIO0, ...); kvmmap(CLINT, ...); kvmmap(PLIC, ...);
kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);  // text
kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W); // data + free RAM
kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
```
- Constants come from **`memlayout.h`** (the manual's memory map as C constants).
- `vmprint` after mapping UART0 (0x10000000):
  - top-level entry **0** → middle-level entry **128** → bottom entry → PA 0x10000000, flags R W V
  - check: `0x10000000 >> 12 >> 9` = 0x80 = **128** ✓

## 10. kvminithart: Turning Paging On
```c
w_satp(MAKE_SATP(kernel_pagetable));
sfence_vma();
```
- A **dramatic moment**: before writing `satp`, addresses are physical; the **very next instruction's PC** is translated through the new page table.
- It works only because the kernel is **identity mapped** → the next PC translates to the same physical address.
- Bad page table → overwrite kernel data, or page fault → kernel panic. Page-table bugs show up as **bizarre crashes** that need detective work.

## 11. walk()
```c
pte_t *walk(pagetable_t pagetable, uint64 va, int alloc)
```
- Software version of the hardware walk: goes level 2 → 1 → 0.
- If an intermediate directory is missing and `alloc` is set → allocate a zeroed page for it and continue.
- Returns a pointer to the **bottom-level PTE**; the kernel can then write a PA + flags into it.
- `kvmmap`/`mappages` = call `walk` for each page in the range and fill in the PTE.
- Works after paging is on because of the identity mapping (page-table pages' physical addresses are also valid virtual addresses).

## 12. User Address Spaces
- Also go from 0 to MAXVA, each with its own page-table tree allocated from free memory.
- When the kernel runs a process, it loads that process's root PA into `satp`.
- Processes are usually much smaller than the full virtual space.
- Many processes map identical kernel parts — xv6 doesn't share those, but a real OS would.
