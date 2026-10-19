# LEC21 - Networking

## Big Picture
- Network structure, packet formats (Ethernet, ARP, IP, UDP) and how a typical kernel **network stack** is organized — background for the **net lab** (e1000 driver).
- Paper: **receive livelock** (Mogul & Ramakrishnan) — under overload, an interrupt-driven router's output **drops toward zero**; fix by switching to **polling**.

---

## 1. Network Structure
```
host ─┐                                  ┌─ host
host ─┼─ LAN (MIT) ── router ── router ── LAN (Stanford) ─┼─ host
host ─┘          (Internet backbone)                └─ host
```
- **LAN** (local area network): hosts that can see each other's packets (cable, Ethernet switch, Wi-Fi). Software hides the differences at a low level. Limited to ~a few hundred hosts (e.g. broadcasts).
- Larger Internet = many LANs connected by **routers**, a router network with long-distance links. **Routing**: hosts at MIT reach hosts at Stanford hop by hop.
- **Ethernet** protocols: local communication. **IP**: long-distance routing, layered on top.

## 2. Ethernet
```
| dest MAC (48) | src MAC (48) | type (16) | payload ... |
```
- Frame = Ethernet's word for packet. Hardware also adds start/end-of-frame signals (special electrical symbols, invisible to software).
- **48-bit address** per NIC: high 24 bits = **manufacturer**, low 24 = unique number (often sequential). Unique, but **say nothing about location** → only useful on the same LAN.
- `type`: which higher-level protocol handles the payload (**0x0800 = IP**, **0x0806 = ARP**).
- `ff:ff:ff:ff:ff:ff` = **broadcast** to all hosts on the LAN.
- Lab's `kernel/net.h` defines structs for each header; software uses them to **parse** and **format** packets.
- **`tcpdump`** shows packets with a hex dump — try it; you'll need it for the lab.

## 3. ARP (Address Resolution Protocol)
- IP uses **32-bit IP addresses**, which have structure: high bits ≈ **network number** (routers use them), low bits = host on its LAN.
- On the final LAN, need IP → Ethernet address:
  - Broadcast an **ARP request**: "whoever owns IP X, send me your Ethernet address"
  - Owner sends an **ARP reply**
- ARP packet sits in the Ethernet payload (type 0x0806): sender/target IP and Ethernet addresses, request/reply code. Redundant Ethernet addresses because ARP was designed to work on non-Ethernet networks too.
- In the lab: the host (10.0.2.2) ARPs for xv6's IP (10.0.2.15); xv6 replies.

## 4. Nested Headers
```
| Ethernet | IP | UDP | DNS | ...
```
- Sending: each layer **prepends** its header (DNS → UDP → IP → Ethernet).
- Receiving: each layer **validates**, **strips** its header, uses a type/protocol field to pick the next layer.

## 5. IP
```
| ... | len | ... | protocol (8) | checksum (16) | src IP (32) | dst IP (32) |   (20 bytes)
```
- Ethernet header is replaced at each hop; the **IP header stays** from original source to final destination → global significance.
- **Destination IP**: high bits guide routers (MIT long had network **18.x.x.x**; routers worldwide had a table entry for 18).
- **Protocol** field: e.g. **17 (0x11) = UDP**, 6 = TCP.
- **Checksum**: detect corruption → discard.

## 6. UDP (and TCP)
- IP gets a packet to a **host**; need to pick the **application** → **ports**.
```
| src port (16) | dst port (16) | length | checksum | payload |
```
- **TCP**: complex — sequence numbers, retransmission, in-order delivery without gaps (used by the web).
- **UDP**: simple **best effort**, no error correction.
- **Sockets API**: a process asks to receive packets for a port → gets a **file descriptor**; arriving packets appear on it.
- **Well-known ports** (DNS = **53**); clients use a random **source port** → server copies it into the reply's dest port → the right app gets the reply.
- Sending to a host on the same LAN → ARP + send directly; elsewhere → send to a **router** on the LAN.
- **Max packet size**: Ethernet 1500 B (modern up to ~9000). Limits: checksums/ECC reliably cover only so many bits; routers/hosts need buffers for max size. IP's 16-bit length caps at **64 KB**.
- End of lab: xv6 sends a DNS query to Google's DNS server and gets the reply.

## 7. Kernel Network Stack (typical)
```
apps (browser, DNS server)
──────────────────────────
sockets layer   fd ↔ port tables, per-socket queues of received packets
UDP | TCP       transport: UDP ~stateless; TCP keeps protocol control blocks (seq #s, unacked data)
IP  | ARP
NIC driver(s)
──────────────────────────
NIC hardware → network
```
- Receive: packet moves **up**, headers stripped. Send: moves **down**, headers added. Design follows the packet nesting.
- **Packet buffers** used throughout, with **queues** between layers; one buffer allocator (**mbufs** in the lab).

## 8. Control Flow: Independent Actors + Queues
```
NIC ──RX interrupt──► [input queue] ──► IP thread ──► UDP ──► socket queue ──► app
                                         └─(router: forward)─► [output queue] ──► NIC (TX interrupt)
```
- **RX interrupt routine**: does the minimum — copy packet off the NIC, **append to a queue**, return. (NIC memory is small; RAM is huge → avoid NIC overflow in bursts.)
- **IP thread** (or similar): processes queued packets; delivers up to sockets, or — if the host is a **router** — appends to another NIC's **output queue**. (Home Wi-Fi/cable routers usually run **Linux's stack**.)
- **TX interrupt**: NIC finished sending, ready for more.
- These are **separately scheduled**; on a uniprocessor interrupts have absolute priority over the thread; apps are yet another independently scheduled entity.
- **Queues** everywhere:
  - absorb **temporary bursts**
  - keep the output NIC **busy** (100% utilization)
  - let parts be structured **independently** (IP thread doesn't need to know about interrupts or apps)
- Multiple NICs: routers (cable + Wi-Fi), servers talking to the Internet on one network and a private database network on another.

## 9. How NICs Deliver Packets
**Old (the paper's NIC)**: NIC buffers packets in **its own memory**, interrupts; driver loops copying **byte/word by byte** into host memory. Made sense 30 years ago; today CPU ↔ device-on-bus interactions are **very slow**.

**Modern (e1000, the lab)**: **DMA** directly into host memory.
- Driver allocates packet buffers (e.g. 16 × 1500 B) and an array of pointers to them = **DMA ring**; tells the NIC the ring's address.
- **RX ring**: on arrival, NIC fetches the next buffer pointer from RAM, writes the packet there, advances its index (wraps around).
- **TX ring**: driver puts packets to send; NIC sends them.
- Lab job: write the driver for these rings.
- Modern NICs: still DMA rings, plus **many queues** (NIC chooses a queue per packet by some header field — e.g. per **VM** at Amazon), some **TCP processing** offloaded (checksums).
- DMA details: address translation hardware between device and RAM may let NICs use virtual addresses; **cache coherence**: if the latest data is in the CPU cache, NIC DMA reads/writes go to the cache.
- The lab stack is minimal (no TCP, no multiple NICs, no performance tuning).

## 10. Receive Livelock (the paper)
**Graph**: router with 2 NICs; x = input packet rate, y = output (forwarded) rate.
- **Goes up (y = x)**: every packet in is forwarded out.
- **Stops at ~5000 pkt/s**: some resource is saturated. Likely **CPU**: ≈ **200 µs** per packet. Not the network: 10 Mb/s Ethernet ≈ 10–15K small pkt/s.
- **Ideal**: stay flat at 5000, dropping the excess.
- **Actual**: **goes down toward zero**!

**Why?** Each extra input packet triggers an **interrupt**, which has **priority** over everything and is **expensive** (CPU copies the packet off the NIC). As input grows, interrupts steal more CPU from the forwarding thread → eventually **100%** of CPU in the interrupt routine, **0** packets forwarded.
- Packets are lost at the **input queue**: it's full (forwarding thread never runs), so the interrupt routine copies each packet in and then **discards** it — wasted work.
- = **interrupt livelock**. General phenomenon: a pipeline of tasks where an earlier stage gets priority and the input rate can't be controlled. Other resources can livelock too (e.g. NIC DMA consuming RAM bandwidth).

## 11. The Fix: Interrupts → Polling Under Load
```
interrupt handler:
    disable this NIC's interrupts
    wake up network thread
    return                     // does NOT copy packets

network thread:
    loop:
        for each NIC: pull up to 5 packets (quota), process them fully
        if no NIC had packets:
            enable interrupts; sleep
```
- **High load**: thread **polls** with interrupts off → interrupts never steal time. Excess packets are dropped **inside the NIC** (its queue is full) — **no CPU wasted** on them.
- **Low load**: interrupts enabled → woken immediately when a packet arrives.
- Result: output rises to 5000 and stays **flat** — no livelock.
- Quota (5 packets) gives **fairness** among NICs and avoids starving output.
- NIC needs only enough buffering for a quota — buffering is for **transient bursts**, not persistent overload.
- **Other stages can livelock too**: if a local app doesn't read its socket fast enough, the network thread could take 100% CPU and the app 0%. Paper's fix: when a socket queue gets long, **stop pulling packets** from the NIC so the app can run.
- Lesson: **don't spend resources on a packet that will be dropped later**.
