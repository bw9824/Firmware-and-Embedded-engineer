# Embedded Systems Protocols Sheet

| Feature / Protocol | UART | I2C | SPI |
| --- | --- | --- | --- |
| Full Name | Universal Asynchronous Receiver-Transmitter | Inter-Integrated Circuit | Serial Peripheral Interface |
| Clock Synchronicity | Asynchronous**<br>(No dedicated clock line) | Synchronous(Shared `SCL` clock line) | Synchronous (Shared `SCK` clock line) |
| Transmission Mode | Full-Duplex | Half-Duplex | Full-Duplex |
| Physical Pins Needed | 3 Pins: `TX`, `RX`, `GND` | 2 Pins: `SDA`, `SCL` | 4+ Pins: `MOSI`, `MISO`, `SCK`, + 1 `CS` per slave |
| Bus Topology | Point-to-Point (1-to-1 strictly) | Multi-Master / Multi-Slave | Single-Master / Multi-Slave |
| Device Targeting | None (Direct peer connection) | Software 7-bit or 10-bit addressing | Hardware Chip Select (CS) pin gating |
| Electrical Output Type | Push-Pull | Open-Drain / Open-Collector(Requires external Pull-up resistors) | Push-Pull |
| Max Practical Speed | Slowest (Usually up to 115.2 kbps) | Medium (100k–400kbps, high-speed modes up to 3.4Mbps) | Fastest (Blazing fast, up to tens of MHz, e.g., 50MHz) |
| Key Pros | Minimal wiring required;<br>Simple hardware interface. | Incredible scalability; Connect up to 127 devices on 2 lines. | Maximum data throughput; No communication protocol overhead. |
| Key Cons | Limited to 2 devices;<br>Baud rate mismatch breaks data. | Slow speed due to RC rise delay; Complex start/stop software handling. | Wastes GPIO pins for `CS` lines; No native hardware handshake/ACK. |
| Typical Use Cases | Debug Console (`printf`), GPS/Bluetooth modules. | Low-speed sensors (IMUs, temperature), EEPROMs, RTCs (Real-Time Clocks). | High-bandwidth components (LCD screens, SD cards, external Flash). |
| Advanced Hard-core Mechanism | Baud Rate Mismatch Tolerances: Clocks must be within ±2% to prevent bit framing errors. | Arbitration: Wired-AND allows "0" to win bus collisions natively. Clock Stretching: Slave pulls `SCL` LOW to pause the Master. | SPI Modes (0-3): Configured via CPOL (clock idle state) and CPHA (sampling clock edge). |
| Step 1: Physical Layer Debugging | Verify `TX`/`RX` are cross-connected; Measure bit-width on an oscilloscope to check baud rate. | CRITICAL: Check if Pull-up Resistors are physically present. Verify if `SDA`/`SCL` lines are swapped. | Ensure the target device's `CS` line drops LOW during transmissions. Check for signal reflections on high-speed lines. |
| Step 2: Protocol Layer Debugging | Perform a Loopback Test by shorting `TX` & `RX` directly on the MCU to isolate software. | Connect a Logic Analyzer to check for ACK/NACK responses.<br>Verify 7-bit vs 8-bit slave address shifting. | Match CPOL and CPHA settings between Master and Slave precisely. Check for SPI clock phase drift. |
