# LEC22 - Meltdown

## Big Picture
- Kernel security so far = **isolation**: user/supervisor mode, page tables, defensive syscall code.
- **Meltdown (2018)**: user code could read **kernel memory** despite page-table protections — a surprising break of a basic assumption.
- A **micro-architectural attack**: exploits hidden CPU implementation details (often only guessed at). Meltdown is fixable and now fixed, but raised fears of more such attacks.
- Two ingredients: **speculative execution** + **CPU cache timing**.

---

## 1. The Precondition
- At the time, operating systems mapped the **whole kernel into every process's page table**, with the user-access bit (`PTE_U`) **clear**.
- A kernel virtual address was therefore *present* in the page table while in user space, but using it from user mode causes a **page fault**.
- Why map the kernel in: a system call didn't need to **switch page tables** (switching flushes the TLB and virtually-indexed caches, slowing later code). This made syscalls faster — and Meltdown relies on this habit.
- (xv6 does NOT do this — its user page tables map only user pages + trampoline + trapframe. So xv6 isn't vulnerable.)
- Attacker needs the **kernel virtual address** of the target. Realistic because: attackers have time (try every address, study kernels). Kernels already use **KASLR** (load the kernel at a random address) to resist address guessing, but this is assumed defeatable given patience.

## 2. The Mechanism (why it works despite the fault)
If CPUs behaved exactly as the manual says, the attack would simply fault on the kernel load and stop. It works because of two real implementation behaviors.

### Speculative / out-of-order execution (a performance trick)
- A load from RAM takes **hundreds of cycles**; the CPU doesn't idle. It uses **branch prediction** and runs **ahead**, executing later instructions **speculatively** before it knows whether they should run.
- Results go into **shadow (temporary) registers**. If the speculation was correct → committed; if wrong → **discarded** as if never executed.
- **Architectural** behavior = what the manual advertises. **Micro-architectural** = what the chip actually does internally; intended to be **transparent** (same visible results).
- **Faults are reported at retirement.** An instruction **retires** only after it finishes and every earlier instruction has retired. A permission-violating load doesn't fault immediately — it faults when it **retires**, possibly hundreds of cycles later.
- **The Intel flaw**: during speculation, a load of a kernel address (page-table entry exists but no permission) **still returned the real data** to dependent speculative instructions. The fault was deferred to retirement, which then rolled back the registers.
- **AMD** CPUs (same ISA) were **not** vulnerable — they enforce the permission check even for the speculative load, so forbidden data never reaches a register. Recent Intel CPUs adopted this, apparently with no performance loss (the check is a few gates; the fault still had to wait for retirement anyway).

### Caches are not truly transparent
- **L1 data cache** (~64 KB, a few cycles), **L2** (bigger, physically indexed, ~dozen+ cycles), RAM (~hundreds of cycles). An L1 entry is believed to also hold a copy of the page-table **permission** bits.
- Because OSes didn't switch page tables on syscalls, the **L1 held both user and kernel data** while in user space (the permissions decided a fault, not the presence of data).
- Micro-architecture is invisible **architecturally** but **visible through timing**: a cached load returns in a few cycles, an uncached one in ~100× that. With a cycle-accurate timer you can tell the difference.

## 3. Flush + Reload (a covert channel, not itself an exploit)
- Answers "did some code touch memory address X?" — but only for memory **you're allowed to access**.
- **Flush** the cache line for X (Intel's `clflush`, or by loading enough other data to evict it).
- Run the code of interest.
- **Reload** X and **time** it (Intel's cycle-accurate `rdtsc`): a **fast** load ⇒ X was cached ⇒ the code touched it; a **slow** load ⇒ it didn't.

## 4. How Meltdown Combines Them (conceptually)
- Use a user buffer whose cache lines you can probe with flush+reload, with the two candidate lines placed **far apart** (a whole page) so the hardware **prefetcher** doesn't blur them.
- Delay retirement of the illegal kernel load (e.g. by making an earlier long-running instruction not retire for a while), giving speculation time to run.
- During that window, the CPU speculatively loads one **bit** of kernel data and uses it to index which of the two buffer lines gets pulled into the cache. Registers are rolled back when the fault finally fires, but the **cache state change persists** (it's "hidden" micro-architectural state).
- Recover from the fault (register a page-fault handler, or other techniques), then **flush+reload** the two lines to learn which was cached → the leaked bit.
- Repeat to scan memory. Reading **one bit at a time** is faster than a byte at a time: a bit needs ~2 probes, a byte needs 256. Since the kernel usually maps **all physical memory**, you can potentially read **all of RAM** (other processes' memory).

## 5. Threat Model
- **Time-sharing machine**: log in alongside others, scan kernel memory for someone's password sitting in I/O/network buffers.
- **Cloud** (AWS): peer into other customers' VMs/containers on the same host — the big real-world worry.
- **Browser**: untrusted JavaScript / WebAssembly from websites might run it (sticking point was getting nanosecond timing).

## 6. Why It's Unreliable
- The paper's own runs show many positions where extraction **failed** even after many retries (sometimes only ~10 bytes/s).
- Likely: it works reliably when the kernel data is already in the **L1 cache**; for data not in L1, the speculative load may or may not do all the work to fetch from RAM — an effective race — so it occasionally works. The exact conditions aren't fully understood.

## 7. Fixes
**Software — KAISER / KPTI (deployed within weeks):**
- **Don't map the kernel in user page tables** (except a tiny trampoline) — switch page tables on syscalls, exactly like **xv6**.
- Then the kernel virtual address is **meaningless** in user space (no translation, not in the TLB) → the CPU can't speculatively load its data → attack fails.
- Cost: page-table switch per syscall/interrupt flushes the TLB and virtually-indexed L1 → slower syscalls. **PCID** (tag TLB entries by address space) avoids full flushes. Real-world impact on typical workloads ~**5%** — acceptable. Linux enables the mitigation based on what the CPU reports.

**Hardware:**
- Check permissions **early** and never forward data to speculative instructions without permission (AMD always did; modern Intel does). Likely cheap; probably skipped originally just to save a few gates on a critical path with no architectural downside.

## 8. Bigger Lessons
- Hidden performance optimizations that are "architecturally transparent" are **not** transparent through **timing** → a leakage channel.
- Decades of micro-architectural tricks now have to be re-examined for security; many similar attacks followed (every recent security conference has sessions on speculation-based attacks).
- **Spectre** (companion paper): instead of reading forbidden data yourself, **train the shared branch predictor** so a *victim* program speculatively executes instructions of your choosing, loading secrets into the cache; then flush+reload shared memory to sense them. You don't need to access the secret's address yourself — just steer the victim's speculation. Harder to fix generally than Meltdown.
- Sharing (CPU caches, disk/network, even EM emissions) creates side channels in general; isolation as an idea is strained by this whole class of attacks.
- Responsible disclosure: researchers notify CPU and OS vendors **before** publishing (now required for acceptance).

## 9. Cache Hierarchy Notes (Q&A)
- Each core: own **L1** (small, fast, **virtually** indexed) and usually own **L2**; often a shared **L3**. L2/L3 are **physically** indexed (usable regardless of virtual address; survive page-table switches). L1 is virtually indexed → logically must be flushed on a page-table switch.
- The **TLB** is typically looked up in parallel with the L1; on an L1 miss you use the physical address for L2/L3. A TLB miss = page-table load, which can be cached in L2 (not L1, which is virtual).
