# Embedded Systems Protocols Sheet

Interview prep for UART, I2C/SMBus, and SPI, plus the neighbouring interfaces interviewers like to
compare them with. Start with the one-page table, then use the deep dives for follow-up questions.

**Contents**

1. [Quick comparison](#1-quick-comparison)
2. [UART deep dive](#2-uart-deep-dive)
3. [I2C deep dive](#3-i2c-deep-dive)
4. [SMBus vs I2C](#4-smbus-vs-i2c)
5. [SPI deep dive](#5-spi-deep-dive)
6. [Other interfaces you'll be compared against](#6-other-interfaces-youll-be-compared-against)
7. [Where these show up in an NVMe SSD](#7-where-these-show-up-in-an-nvme-ssd)
8. [Debugging playbook](#8-debugging-playbook)
9. [Interview questions with short answers](#9-interview-questions-with-short-answers)

---

## 1. Quick comparison

| Feature / Protocol | UART | I2C | SPI |
| --- | --- | --- | --- |
| Full Name | Universal Asynchronous Receiver-Transmitter | Inter-Integrated Circuit | Serial Peripheral Interface |
| Clock Synchronicity | **Asynchronous** (no clock line; both sides agree on a baud rate in advance) | **Synchronous** (shared `SCL`, driven by the controller) | **Synchronous** (shared `SCK`, driven by the controller) |
| Transmission Mode | Full-Duplex | Half-Duplex | Full-Duplex |
| Physical Pins Needed | `TX`, `RX`, `GND` (+ optional `RTS`/`CTS` for flow control) | `SDA`, `SCL` (+ `GND`) | `SCK`, `MOSI`, `MISO` + one `CS` per target (+ `GND`) |
| Bus Topology | Point-to-point (1-to-1) at TTL level; RS-485 allows multi-drop | Multi-controller / multi-target | Single controller / multi-target |
| Device Targeting | None (direct link) | Address in the first byte: 7-bit (or 10-bit) | Hardware Chip Select (`CS`) line per device |
| Electrical Output Type | Push-pull | **Open-drain** (needs external pull-up resistors) | Push-pull |
| Typical Speed | Commonly 9600–115200 baud for consoles; hardware UARTs usually reach 1–3 Mbaud+ | Standard 100 kbit/s, Fast 400 kbit/s, Fast-mode Plus 1 Mbit/s, High-speed 3.4 Mbit/s | Fastest: tens of MHz (e.g. 50 MHz); Quad SPI flash ~100+ MHz × 4 data lines |
| Flow Control / ACK | None built in (optional RTS/CTS or XON/XOFF) | ACK/NACK after every byte | None — the controller can't tell if a target is even present |
| Key Pros | Minimal wiring; no shared clock; the universal debug console | Only 2 wires for many devices; built-in ACK; multi-controller arbitration | Highest throughput; simple shift-register hardware; no addressing overhead |
| Key Cons | Two devices only (at TTL level); both ends must agree on baud and frame format | Slower (rise time limited by pull-ups and bus capacitance); a stuck target can hang the whole bus | One `CS` pin per device; no ACK; no standard beyond the clock modes |
| Practical Device Limit | 2 | Limited by **400 pF bus capacitance** and address collisions — 112 usable 7-bit addresses (16 of 128 are reserved) | Limited by `CS` pins and `MISO` loading/fan-out |
| Typical Use Cases | Debug console (`printf`), GPS/Bluetooth modules, bootloader/recovery links | Sensors (IMU, temperature), EEPROMs, RTCs, PMICs, board management (SMBus) | SPI NOR flash, displays, ADC/DAC, SD cards (SPI mode), radio transceivers |
| Signature Mechanism | Start-bit synchronization + 16× oversampling; baud mismatch tolerance (≈ ±2–3% combined in practice) | Wired-AND **arbitration** (a `0` wins) and **clock stretching** (target holds `SCL` low) | **SPI modes 0–3** from CPOL (clock idle level) × CPHA (sampling edge) |

---

## 2. UART deep dive

### Frame format

```
idle  start  D0 D1 D2 D3 D4 D5 D6 D7  [parity]  stop  idle
 1      0    ←—— LSB first ——→            P       1     1
```

- Line **idles high**. A **start bit** (low) marks the start of every character.
- **5–9 data bits, LSB first**, optional **parity** (even/odd), **1, 1.5, or 2 stop bits** (high).
- **8N1** = 8 data bits, no parity, 1 stop bit → **10 bits on the wire per byte**.
  - 115200 baud × 8/10 = **11.52 KB/s** of payload.
  - One bit at 115200 baud lasts **8.68 µs** — measure it on a scope to verify the real baud rate.

### How the receiver stays in sync without a clock

The receiver samples the line at typically **16× the baud rate**. It detects the falling edge of the
start bit, waits half a bit to land in the middle, then samples each following bit once per bit
time (often as a majority vote of 3 samples around the center).

Because it re-synchronizes on **every start bit**, clock error only accumulates across a single
frame. By the stop bit (~9.5 bit times later for 8N1), the sampling point must still fall inside
the bit: that gives a theoretical **combined** tolerance of roughly ±5%, and a practical design
budget of about **±2–3% combined** between the two sides. Internal RC oscillators (often ±1–2% or
worse over temperature) are the classic cause of "works on my bench, garbage in the chamber".

### Receive errors

| Error | Meaning |
|---|---|
| **Framing error** | Stop bit sampled low — baud mismatch, noise, or a break |
| **Overrun** | A new byte arrived before software read the previous one — ISR too slow, or no DMA |
| **Parity error** | Parity bit didn't match (only if parity is enabled) |
| **Break** | Line held low for longer than a full frame — sometimes used deliberately as a signal |

### Electrical levels: same protocol, different wires

| Level | Signaling | Notes |
|---|---|---|
| **TTL/CMOS UART** | 0 V / 3.3 V (or 5 V), idle high | What MCU pins speak. Connecting 5 V TX to a 3.3 V-only RX can damage it. |
| **RS-232** | Logic 1 = −3 to −15 V, logic 0 = +3 to +15 V (**inverted**) | Needs a transceiver (e.g. MAX3232). Never connect directly to an MCU pin. |
| **RS-485** | Differential pair (A/B), usually half-duplex | Multi-drop (32 unit loads standard), long cables (~1200 m at low rates), 120 Ω termination at both ends. |

### Firmware patterns

- **Polling** → fine for a debug console, loses bytes under load.
- **Interrupt per byte** + **ring buffer** → standard; ISR only moves the byte into the buffer.
- **DMA + idle-line detection** → best for high baud and variable-length packets: DMA fills a
  buffer, and the *idle line* interrupt says "the packet ended".
- **Flow control**: **RTS/CTS** (hardware) stops the sender before the receiver overruns; XON/XOFF
  (software) sends 0x11/0x13 in-band, which breaks binary data.

---

## 3. I2C deep dive

### Why open-drain and pull-ups

Every device can only **pull a line low** or **let go**; resistors pull it high. That gives you:

- **Wired-AND**: if *anyone* drives low, the line is low. No two drivers ever fight (no short
  circuit), which is what makes multi-controller buses safe.
- **Arbitration**: two controllers start together; each checks `SDA` after sending each bit. The
  one that sent `1` but sees `0` has lost and backs off — non-destructively, the winner's message
  is intact.
- **Clock stretching**: a slow target holds `SCL` low to make the controller wait.

The cost: the rising edge is an **RC curve** (pull-up × bus capacitance), which is what limits speed.

### Bus conditions and one transaction

- **Data changes only while `SCL` is low.** A change while `SCL` is high is a control condition:
  - **START**: `SDA` falls while `SCL` is high.
  - **STOP**: `SDA` rises while `SCL` is high.
- Each byte is **MSB first**, followed by a **9th clock** for **ACK** (receiver pulls `SDA` low) or
  **NACK** (leaves it high).

**Reading a register — the classic interview sequence:**

```
S | ADDR+W | A | REG | A | Sr | ADDR+R | A | DATA0 | A | DATA1 | N | P
   controller →         ←target ACKs        target →  controller ACKs, then NACKs the last byte
```

- `S` START, `Sr` **repeated START**, `P` STOP, `A` ACK, `N` NACK.
- **Why a repeated START instead of STOP + START?** The controller keeps ownership of the bus
  between "set the register pointer" and "read it" — another controller can't jump in — and many
  devices require it to keep the register pointer.
- **Why NACK the last byte?** It tells the target to stop driving `SDA`, so the controller can
  issue STOP.

### Addressing

- **7-bit** addresses give 128 values; **0x00–0x07 and 0x78–0x7F are reserved** (general call,
  10-bit prefix `11110xx`, Hs-mode code, …) → **112 usable**.
- **"8-bit address" confusion**: a datasheet may say `0xA0`/`0xA1` (address byte including the R/W
  bit). The 7-bit address is `0xA0 >> 1 = 0x50`. Linux `i2c-tools` and most drivers want the
  **7-bit** form. Mismatching the two is the #1 "device doesn't ACK" cause.
- **Same address on two devices?** Use the address-select pins, an **I2C mux** (e.g. TCA9548A), a
  second bus, or a part with a programmable address.

### Speed modes

| Mode | Max rate | Max rise time (tr) | Notes |
|---|---|---|---|
| Standard (Sm) | 100 kbit/s | 1000 ns | |
| Fast (Fm) | 400 kbit/s | 300 ns | |
| Fast-mode Plus (Fm+) | 1 Mbit/s | 120 ns | Stronger drivers (20 mA sink) |
| High-speed (Hs) | 3.4 Mbit/s | — | Entered with a special controller code; current-source pull-ups |
| Ultra-Fast (UFm) | 5 Mbit/s | — | **Unidirectional, push-pull, no ACK** — effectively a different bus |

### Sizing the pull-up resistor

```
Rp(min) = (VDD − VOL(max)) / IOL         VOL(max) = 0.4 V, IOL = 3 mA (Sm/Fm)
Rp(max) = tr / (0.8473 × Cb)             tr = rise-time limit above, Cb = bus capacitance
```

Worked example, 3.3 V, Fast mode, Cb = 100 pF:

- Rp(min) = (3.3 − 0.4) / 3 mA ≈ **967 Ω**
- Rp(max) = 300 ns / (0.8473 × 100 pF) ≈ **3.5 kΩ**
- → choose something like **2.2 kΩ**.

Rules of thumb: ~4.7 kΩ for 100 kHz, ~2.2 kΩ for 400 kHz. Too strong a pull-up → devices can't pull
the line below VOL; too weak → slow edges that fail timing at speed and pick up noise.

> **Mixed voltages**: because nothing drives high, a simple MOSFET level shifter (e.g. BSS138
> circuit) lets a 1.8 V controller talk to a 3.3 V target.

### Failure modes interviewers ask about

- **Bus stuck low.** A target was mid-read when the controller reset; it's still holding `SDA`
  low, waiting for clocks. **Recovery**: switch `SCL` to GPIO and toggle it up to **9 times** until
  the target releases `SDA`, then send a STOP. Good drivers do this automatically at init.
- **Clock stretching vs. a controller that ignores it.** Some controllers (or bit-banged drivers)
  don't support stretching, and read garbage from slow targets.
- **NACK on the address**: wrong address (7/8-bit), device unpowered or in reset, wrong bus, or busy
  — e.g. an EEPROM during its internal write cycle (it NACKs until done, so firmware uses **ACK
  polling**).

---

## 4. SMBus vs I2C

SMBus (System Management Bus) is a stricter profile of I2C used for board and system management —
batteries, sensors, PCIe and SSD management. Interviewers like this one for storage roles.

| | I2C | SMBus |
|---|---|---|
| Clock range | DC to 100 kHz / 400 kHz / 1 MHz … | **10 kHz minimum**; 100 kHz classically (SMBus 3.0 adds 400 kHz and 1 MHz) |
| Timeout | **None** — a controller may hold `SCL` low forever | **Timeout ≈ 25–35 ms**: a device seeing `SCL` low that long resets its interface — this is what prevents a permanently hung bus |
| Logic levels | Relative to VDD (0.3 / 0.7 × VDD) | Fixed thresholds (independent of supply) |
| Address ACK | A device may NACK its address when busy | A device must **always ACK its own address** |
| Error checking | None | Optional **PEC** — a CRC-8 byte at the end of a transaction |
| Protocol | Raw bytes | Defined commands: Quick Command, Send/Receive Byte, Read/Write Byte/Word, Block Read/Write, Process Call |
| Extras | — | **SMBALERT#** interrupt line, **ARP** (dynamic address assignment) |

In practice most I2C devices work on an SMBus and vice versa, as long as the clock stays within
both ranges and nobody depends on the timeout behavior.

---

## 5. SPI deep dive

### The four modes

| Mode | CPOL | CPHA | Clock idles | Data sampled on |
|---|---|---|---|---|
| **0** | 0 | 0 | Low | Rising (leading) edge |
| 1 | 0 | 1 | Low | Falling (trailing) edge |
| 2 | 1 | 0 | High | Falling (leading) edge |
| **3** | 1 | 1 | High | Rising (trailing) edge |

Modes **0 and 3** are by far the most common; most SPI NOR flash supports both. A mode mismatch
typically shows as data **consistently shifted by one bit** — e.g. an expected `0xEF` read back as
`0x77` or `0xDE` — rather than random garbage.

### Other things that must match

- **Bit order**: MSB first is the norm, but some devices are LSB first.
- **Word size**: 8-bit is standard; some ADCs/DACs use 12/16/24-bit frames.
- **`CS` polarity and framing**: usually active-low. Many devices treat **`CS` going high as "end
  of command"** — e.g. SPI NOR needs `WREN` (0x06) as its *own* `CS` frame before Page Program.
  Holding `CS` low across two commands is a classic bug.
- **`MISO` must be tri-stated** by every target whose `CS` is high. One target that doesn't release
  `MISO` corrupts reads from all the others.

### Variants

| Variant | What changes |
|---|---|
| **3-wire SPI** | One bidirectional data line instead of MOSI + MISO (half-duplex) |
| **Daisy-chain** | Each target's `MISO` feeds the next one's `MOSI`, all sharing one `CS` (shift registers, LED drivers) |
| **Dual / Quad SPI (QSPI)** | 2 or 4 data lines. Notation `1-1-4` = command on 1 line, address on 1, data on 4; `1-4-4`, `4-4-4` widen more phases. Often supports **XIP** (execute-in-place, flash memory-mapped into the CPU's address space) |
| **Octal SPI / xSPI** | 8 data lines, often DDR — JEDEC JESD251 |

> **Naming**: MOSI/MISO are being replaced by **SDO/SDI**, **COPI/CIPO**, or **PICO/POCI**; `CS`
> is also called `SS` or `nCS`. Same signals.

### Speed limits are about timing, not protocol

At high `SCK`, the round trip — controller drives `SCK` → target's clock-to-output delay → `MISO`
back across the board — has to arrive before the controller's sampling edge. When writes work but
**reads fail only at high speed**, that round trip is the first suspect. Many controllers have a
**sample-delay** setting for exactly this. Also: long `SCK` traces ring, and ringing can
**double-clock** a target — a 22–33 Ω series resistor at the driver usually fixes it.

### SPI NOR command set worth knowing

| Opcode | Command | Why you care |
|---|---|---|
| `0x9F` | Read JEDEC ID | First bring-up check: manufacturer + device ID |
| `0x05` | Read Status Register | Poll the **WIP** (write-in-progress) bit after program/erase |
| `0x06` | Write Enable (WREN) | Required before every program/erase — its own `CS` frame |
| `0x03` / `0x0B` | Read / Fast Read | `0x0B` adds dummy cycles to allow higher clock rates |
| `0x02` | Page Program | Writes up to one page (typically 256 bytes); wrapping past a page boundary is a classic bug |
| `0x20` / `0xD8` | Sector erase (4 KB) / Block erase (64 KB) | Flash bits only go 1 → 0 by programming; erase sets them back to 1 |
| `0x5A` | Read SFDP | Discoverable parameter tables describing the chip's capabilities |

---

## 6. Other interfaces you'll be compared against

| Interface | Wires | Key idea | Typical speed | Where |
|---|---|---|---|---|
| **CAN / CAN FD** | Differential `CAN_H`/`CAN_L`, 120 Ω at each end | Multi-controller; **bitwise arbitration on the message ID** — dominant `0` wins, the same idea as I2C. CRC, ACK slot, error counters, bus-off | Classical ≤ 1 Mbit/s (8-byte payload); FD data phase ~5–8 Mbit/s (64-byte payload) | Automotive, industrial |
| **I3C** (MIPI) | `SDA`, `SCL` | Successor to I2C: push-pull data for speed, **dynamic addressing**, **in-band interrupts**; legacy I2C devices can share the bus | 12.5 MHz SDR, more with HDR modes | Phones, sensors, newer DDR5/server management |
| **USB** | Differential `D+`/`D−` (plus SuperSpeed pairs) | Host-centric, enumeration and descriptors, defined device classes | 12 Mbit/s FS, 480 Mbit/s HS, 5–20+ Gbit/s SS | Peripherals |
| **JTAG** | `TCK`, `TMS`, `TDI`, `TDO` (+ `TRST`) | TAP state machine; boundary scan and CPU debug; devices daisy-chain | ~1–50 MHz | Debug, board test |
| **SWD** | `SWDIO`, `SWCLK` | ARM's 2-wire alternative to JTAG for Cortex debug | ~1–50 MHz | Arm MCU debug |
| **PCIe** | Differential TX/RX pairs per lane + `REFCLK` | Point-to-point, packet-based (TLP/DLLP), link training (LTSSM), 128b/130b encoding from Gen3 | Gen4 16 GT/s and Gen5 32 GT/s **per lane** → x4 ≈ 7.9 GB/s (Gen4) / ≈ 15.8 GB/s (Gen5) per direction | SSDs, GPUs, NICs |

> **The framing that ties it together:** UART, I2C, and SPI are *low-speed, single-ended* links for
> control and configuration. PCIe and USB are *high-speed, differential, packetized* links for
> data. A real product uses both: the data path is PCIe, and the dozen small parts around it are
> configured over I2C/SPI and debugged over UART.

---

## 7. Where these show up in an NVMe SSD

This is the section that connects the theory to SSD firmware experience. The mapping below is
typical of NVMe SSD designs; details vary by controller and product.

| Interface | Typical role in an SSD |
|---|---|
| **PCIe** | The host data path — NVMe commands, DMA of user data |
| **UART** | Firmware debug console and log output during bring-up; often a vendor recovery/download path |
| **SPI** | **SPI NOR** holding boot code or configuration on many designs (others boot straight from NAND) |
| **I2C / SMBus** | **NVMe-MI** out-of-band management: a server's BMC reads temperature, health, and identity without going through the host OS. The NVMe Basic Management Command lives at SMBus address `0x6A` (7-bit). Also the FRU/VPD EEPROM, temperature sensors, and the PMIC |
| **JTAG / SWD** | Debugging the controller's CPU cores during bring-up and failure analysis |

> **My story (fill in before the interview):** one concrete case where an interface mattered on
> the PCIe Gen4/Gen5 controller projects — e.g. a bug found via the UART log, an SMBus/NVMe-MI
> requirement from a customer, or a boot-flash issue. What was the symptom, how did you narrow it
> down, what was the fix?

---

## 8. Debugging playbook

Work from the bottom layer up: **power and wiring → electrical signal → protocol → firmware**.

| Step | UART | I2C | SPI |
|---|---|---|---|
| **1. Physical** | `TX`↔`RX` crossed; **common GND**; voltage levels match (3.3 V vs 5 V vs RS-232) | **Pull-ups physically present** and sized right; `SDA`/`SCL` not swapped; device powered and out of reset | `CS` actually goes low during the transfer; `SCK` present at the target pin; `MISO` not driven by another target |
| **2. Electrical** | Measure one bit width on a scope (115200 → 8.68 µs) to verify the real baud | Scope the **rise time** against the mode's limit; check low level is below 0.4 V | Look for **ringing** on `SCK`; check signal integrity at the target, not the controller |
| **3. Protocol** | **Loopback test**: short `TX`→`RX` on the MCU — echo works ⇒ the problem is outside the MCU | **Logic analyzer** with I2C decoder: is the address ACKed? Verify 7-bit vs 8-bit address; scan the bus (`i2cdetect -y <bus>` on Linux) | Match **CPOL/CPHA**, bit order, word size; read the JEDEC ID (`0x9F`) as a known-answer test |
| **4. Firmware** | Overrun errors → ISR too slow, add DMA/ring buffer; framing errors → clock source | Bus stuck low → 9-clock recovery; timeouts on every transaction; check clock-stretching support | Reads fail only at speed → lower `SCK` / tune sample delay; check `CS` framing between commands |

**Tools**: oscilloscope (edges and levels), logic analyzer with protocol decoders (Saleae,
sigrok/PulseView), USB-UART adapters (FTDI, CP2102), Linux `i2c-tools`, `flashrom` for SPI NOR.

---

## 9. Interview questions with short answers

**Why does I2C need pull-up resistors?**
Its outputs are open-drain: devices can only pull low. The resistors provide the high level. That
design is what allows wired-AND arbitration between controllers and clock stretching, with no risk
of two drivers fighting.

**Why is SPI faster than I2C?**
Push-pull drivers give sharp edges instead of RC rise times; no address byte; no ACK bit per byte;
and full-duplex transfer. The trade-off is more pins and no built-in error detection.

**Walk me through reading a register over I2C.**
START, address+W, register index, repeated START, address+R, read bytes (ACK each except the last,
NACK the last), STOP. The repeated START keeps bus ownership and the device's register pointer.

**Your I2C bus is stuck low. What happened and what do you do?**
Usually a target was interrupted mid-transfer and is still holding `SDA` low. Toggle `SCL` up to 9
times until it releases, then issue STOP. Prevent it by doing that recovery at driver init.

**How does a UART receiver work without a clock?**
It re-synchronizes on every start bit, then samples at the expected mid-point of each bit using
16× oversampling. Clock mismatch accumulates over one frame; beyond about ±2–3% combined, the last
bits get sampled in the wrong place and you see framing errors.

**What's the real throughput of 115200 baud?**
8N1 uses 10 bits per byte, so 11,520 bytes/s.

**SPI reads return data that's consistently off by one bit. Cause?**
Mode mismatch — the controller samples on the other edge than the target updates on. Check CPOL and
CPHA against the target's datasheet timing diagram.

**Two sensors share the same I2C address. Options?**
Address-select pins, an I2C mux, a second I2C bus, or a part with a programmable address.

**How do you choose between UART, I2C, and SPI for a new part?**
Usually the part decides. Otherwise: SPI for bandwidth (flash, displays, fast ADCs); I2C when many
slow devices must share few pins and you want ACKs; UART for a point-to-point link to another
processor or module, or a human-readable console.

**What's the difference between SMBus and I2C?**
SMBus is a stricter profile: a 10 kHz minimum clock and a ~25–35 ms timeout (so the bus can't hang
forever), fixed logic thresholds, mandatory address ACK, optional PEC CRC, and defined command
formats. It's what server management (and NVMe-MI on SSDs) uses.

**When would you use DMA for UART?**
High baud rates or bursty traffic where per-byte interrupts would overrun or eat CPU. DMA fills a
buffer, and an idle-line interrupt tells you a packet ended.

**Why is CAN arbitration "non-destructive"?**
Like I2C, the bus is wired-AND: a dominant `0` overrides a recessive `1`. A node that sends `1` and
reads `0` stops transmitting; the winner's frame continues unharmed, so no bandwidth is wasted on
collisions.
