# LEC24 - Q&A

## Big Picture
- Final Q&A: the **net lab** (e1000 driver) and **mmap**, plus what classes to take next.
- OS lessons: how a driver coordinates with hardware via **DMA rings** (producer/consumer, ownership), why **no lock** is needed in `receive`, why the **while loop**, what **mbufs** are, and what **mmap** buys you over read/write/lseek.

---

## 1. Hardware Picture
- QEMU emulates a whole **board**: RISC-V processor (4 cores, L1/L2 caches) + RAM + devices on a bus; the **e1000 NIC** hangs off a (slower) **PCIe** bus.
- NIC has a **controller** with **memory-mapped I/O registers**: the driver reads/writes them with ordinary load/store to physical addresses.
- Packets and **two rings** (TX, RX) live in **RAM**. The driver tells the NIC the ring addresses (via registers); the NIC **DMAs** packet data between RAM and the wire **without the CPU**.
- Both the NIC and the CPU manipulate the rings → need a coordination protocol.

## 2. Software Structure
```
user:  nettest → write() → network stack (IP/UDP/ARP)
                              ↓ e1000_transmit (top half)    ← manipulates TX ring
kernel driver: ───────────────────────────────────────────
                              ↑ e1000_recv (bottom half, interrupt) → net_rx → stack
```
- Driver = **`e1000_transmit`** + **`e1000_recv`**. `recv` runs in the **interrupt handler** (bottom half); `transmit` is called by kernel threads on behalf of processes (top half).
- **Concurrency**:
  - Many senders can be in `transmit` at once → **needs a lock** (the e1000 lock).
  - Only **one** `recv` runs at a time (further interrupts are blocked until the current one is handled) → `recv` shares no data with a concurrent `recv`.
- `transmit` and `recv` operate on **separate** rings → the driver doesn't share data structures between them.

## 3. Why No Lock in receive (and why a lock causes a panic)
- `recv` has no concurrent peer and shares nothing → no lock needed.
- But `recv` can call **up into the stack** and back **down**: an **ARP request** → `e1000_recv` → `net_rx` → `net_rx_arp` → **`e1000_transmit`** (to send the reply), which **acquires the e1000 lock**.
- If you locked in `recv` too, `transmit` would try to re-acquire the **same lock already held** → **deadlock → xv6 panics** (re-acquire by the same CPU).
- Fixes (any is fine): **no lock in recv** (staff solution), **recursive/re-entrant locks**, or **two separate locks**.
- A lock *would* be needed if top and bottom halves **shared a data structure** (like the console driver's shared queue).

> 💡 Typically a bottom half does minimal work (grab packet, enqueue) and **never calls into the top half** — a separate top-half thread drains the queue. This driver calls up for simplicity.

## 4. DMA Rings
```
TX ring (in RAM):  [desc][desc][desc]...  (fixed size, wraps = "ring")
                      ↑head(hardware)   ↑tail(software)
```
- **Producer/consumer** coordination:
  - **TX**: software owns the **tail**, adds a descriptor at tail+1, advances tail. Hardware owns the **head**, sends the packet there, advances head. Descriptors between head and tail are **owned by the NIC** — software must not touch them.
  - **RX**: hardware fills descriptors and sets the **DD (descriptor done) bit** in the status field; software checks the tail, consumes DD-set descriptors, advances tail.
- Head/tail are **control registers** on the NIC.
- **Why a ring, not one register** (like the UART)? To absorb **bursts** — the NIC is very fast (gigabits/s); queue up many packets so it stays busy, and buffer incoming bursts for the OS to process.
- **Queue full** → packets are **dropped** (one cause of packet loss; TCP may retransmit).

## 5. Descriptors
- **Defined by the hardware** — the driver can't change their layout.
- **RX descriptor**: an **address** (where the NIC should DMA the received packet in RAM) + a **status** field (NIC sets **DD** when a packet has arrived). Shaded fields filled by hardware, white by software.
- **TX descriptor**: an **address** (where the data to send is) + a **command** field:
  - **EOP** (end of packet): this is the last descriptor of the packet
  - **RS** (report status): set a done bit when transmitted

## 6. Code: transmit
```c
acquire(&e1000_lock);
idx = regs[E1000_TDT];             // tail
if (!(tx_ring[idx].status & E1000_TXD_STAT_DD)) { release; return -1; }  // ring full
if (tx_mbufs[idx]) mbuffree(tx_mbufs[idx]);   // free the old mbuf in this slot
tx_mbufs[idx] = m;
tx_ring[idx].addr   = (uint64)m->head;
tx_ring[idx].length = m->len;
tx_ring[idx].status = 0;
tx_ring[idx].cmd    = E1000_TXD_CMD_RS | E1000_TXD_CMD_EOP;
__sync_synchronize();              // memory fence
regs[E1000_TDT] = (idx + 1) % TX_RING_SIZE;   // advance tail → NIC reads the descriptor
release(&e1000_lock);
```
- **`__sync_synchronize()`**: a **memory fence** so all the descriptor writes are **materialized in RAM** (not stuck in L1/L2, not reordered) **before** the NIC reads them after the tail update — otherwise the NIC reads stale fields.

## 7. mbufs
- **OS abstraction** for holding packet data; nothing to do with the NIC. Separate from the hardware rings.
- Needed because a received packet is handed to the stack and may sit on a **socket's queue** until a process calls `read` — something must hold the bytes meanwhile. Each socket can have a **list of mbufs**.
- An mbuf has a backing buffer (`buf`) and a `head` pointer into it (where the packet starts). The list operations (`push`/`pop`) are in `net.c`; if you don't use them you treat the mbuf as a byte array.
- Standard structure in real kernels (Linux has a similar one).

## 8. Code: receive — Why the while loop
```c
while (1) {
  idx = (regs[E1000_RDT] + 1) % RX_RING_SIZE;
  if (!(rx_ring[idx].status & E1000_RXD_STAT_DD)) break;   // no (more) packet
  rx_mbufs[idx]->len = rx_ring[idx].length;
  net_rx(rx_mbufs[idx]);                 // up-call into the stack
  rx_mbufs[idx] = mbufalloc(0);          // fresh buffer for the NIC
  rx_ring[idx].addr = (uint64)rx_mbufs[idx]->head;
  rx_ring[idx].status = 0;
  regs[E1000_RDT] = idx;                 // give the slot back to the NIC
}
```
- **Why loop?** A **burst** of packets generates **one interrupt** (the NIC can't raise another while one is pending, and interrupts may be off during a locked section). All the burst's packets sit in the ring. Without the loop you'd handle only the first and leave the rest stuck until the next interrupt — which may not come. (A test sending 10 parallel pings gets 10 replies in one burst → one interrupt.)
- Must only read descriptors the **software owns** (DD set, between tail and head) — scanning the whole ring and reading arbitrary descriptors is dangerous (the NIC may be refilling them).

## 9. mmap vs read/write/lseek
Problem: update a struct in the **middle** of a file.
```c
fd = open(f, ...);
read(fd, &s, 16);        // offset: 0 → 16
s.field = 1;
write(fd, &s, 16);       // NO offset arg → writes at offset 16, the WRONG place!
```
- `write` uses the fd's **implicit offset**, which `read` already advanced. Need **`lseek(fd, 0, ...)`** to reset it first → clunky (open, read, seek, write).

**With mmap:**
```c
fd = open(f, O_RDWR);
struct h *h = mmap(0, len, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
h[0].field = 1;          // just a memory write
munmap(h, len);
```
- The file is mapped into memory; treat it as an **array of bytes/structs** and update arbitrary locations directly with pointers — much more convenient for in-file data structures (**databases** use this a lot).
- read/write is better for **stream** I/O (stdin → stdout); mmap is better for **random in-place updates** of a file.

## 10. Q&A Note
- **How does the NIC know the previous interrupt is handled?** The driver re-enables interrupts (writes the interrupt-enable register) at the end of `recv`; if the DD condition still holds when interrupts are re-enabled, another interrupt fires — easy to miss that one instruction.

## 11. What Next (systems classes)
- OS follow-ons (grad OS / projects), computer architecture, compilers, **networking**, **databases**, **distributed systems**, **security**, **systems verification**.
- Read systems papers (conferences are public and recorded), follow **LWN.net** for Linux, and **keep hacking** — extend the labs or build your own projects. Big fans of learning by doing.
