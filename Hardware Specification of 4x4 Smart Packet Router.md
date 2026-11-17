# Hardware Specification of 4x4 Smart Packet Router (`router_4x4_top`)

## 1. Overview & Architecture

### 1.1 Purpose & System Context

The `router_4x4_top` is a **4-port packet switch** commonly found in Network-on-Chip (NoC) interconnects and network interface controllers. Its job is to accept variable-length byte packets arriving simultaneously on four independent input channels (`in_0` through `in_3`), inspect each packet's header to determine its destination and required processing, resolve traffic conflicts when multiple inputs target the same output, and transmit valid packets out of the four output channels (`out_0` through `out_3`).

The hardware is divided into two distinct functional planes:

   1. **The Data Plane (Streaming Interfaces):** Four input ports and four output ports operating on an 8-bit (`1 byte/cycle`) `valid`/`ready` handshake protocol. All ports operate synchronously on the same system clock (`clk`).
   2. **The Control & Telemetry Plane (APB3 Slave Interface):** A memory-mapped register interface (`psel`, `penable`, `pwrite`, `paddr`, `pwdata`, `prdata`) that allows a host CPU (or UVM testbench) to enable/disable ports, select arbitration algorithms, load encryption keys, read traffic/error statistics, and service interrupts (`irq`).

---

### 1.2 Internal Block Diagram

While the RTL team has full freedom over internal module partitioning, a clean, modular implementation consists of four primary subsystems as illustrated below:

```text
                        +---------------------------------------+
                        |       APB3 Slave & Register File      |
                        |  (CTRL, KEY, IRQ_EN/STAT, Counters)   |
                        +---+---------------+---------------+---+
                            | Config        | Keys/Mode     | Status/IRQ
                            v               v               ^
  +----------------+    +---------------------------------------+    +----------------+
  | Input Port 0   |--->|                                       |--->| Output Port 0  |
  | (FIFO + Parser |    |        4x4 Crossbar & Arbiter         |    | (Payload ALU + |
  |  + FCS Check)  |    |   - Fixed Priority / Round-Robin      |    |  FCS Gen)      |
  +----------------+    |   - Optional QoS Priority             |    +----------------+
                        |   - Non-Preemptive Packet Lock        |
  +----------------+    |   - Broadcast Synchronization         |    +----------------+
  | Input Port 1   |--->|                                       |--->| Output Port 1  |
  | (FIFO + Parser |    |                                       |    | (Payload ALU + |
  |  + FCS Check)  |    |                                       |    |  FCS Gen)      |
  +----------------+    |                                       |    +----------------+
                        |                                       |
  +----------------+    |                                       |    +----------------+
  | Input Port 2   |--->|                                       |--->| Output Port 2  |
  | (FIFO + Parser |    |                                       |    | (Payload ALU + |
  |  + FCS Check)  |    |                                       |    |  FCS Gen)      |
  +----------------+    |                                       |    +----------------+
                        |                                       |
  +----------------+    |                                       |    +----------------+
  | Input Port 3   |--->|                                       |--->| Output Port 3  |
  | (FIFO + Parser |    |                                       |    | (Payload ALU + |
  |  + FCS Check)  |    |                                       |    |  FCS Gen)      |
  +----------------+    +---------------------------------------+    +----------------+

```

---

### 1.3 Core Architectural Principles

#### A. Store-and-Forward Switching (Why We Buffer Whole Packets)

In packet switching, there are two ways to forward data: *Cut-Through* (sending Byte 0 out before Byte N arrives) and *Store-and-Forward* (waiting until the entire packet arrives before forwarding). **`router_4x4_top` strictly uses Store-and-Forward switching.**

   * **Why?** The Frame Check Sequence (`FCS`) checksum is located at the **very last byte** of the packet (`eop`). The router cannot know if a packet suffered bit corruption or a length error until the final byte is received. If the router started transmitting bytes early, a corrupted packet would leak onto the output port before the error was discovered.
   * **How it works:** Each input port stores incoming bytes in an internal buffer until `eop` is received. Only after the complete packet passes both **length validation** and **checksum (`FCS`) verification** does the input port assert a routing request to the central arbiter. If either check fails, the packet is silently flushed from the input buffer and never reaches the crossbar.

#### B. Non-Preemptive Output Locking (Packet-Level Granularity)

Because the streaming output interface carries raw bytes without packet-ID tags, bytes from two different packets must never be interleaved on the same output port.

   * **How it works:** Suppose Input 0 and Input 1 both have valid packets destined for Output 2. The arbiter grants Output 2 to Input 0. From the clock cycle `out_sop[2]` is transferred until the final byte `out_eop[2]` is transferred, **Output 2 is locked exclusively to Input 0**.
   * Even if a higher-priority packet (`PRIO = 3`) arrives at Input 1 mid-transfer, Input 1 must wait until Input 0 finishes transmitting its entire packet (`eop`). Arbitration for Output 2 only occurs again on the cycle after `out_eop[2]` completes.

#### C. Backpressure & Flow Control (`valid` / `ready`)

Downstream receivers connected to `out_0..3` may pause reception at any time by deasserting `out_ready[j] = 0`.

* When `out_ready[j]` goes low mid-packet, the active input port simply pauses reading from its internal FIFO and holds `out_valid[j]`, `out_data[j]`, `out_sop[j]`, and `out_eop[j]` steady.
* Meanwhile, upstream senders can continue pushing new bytes into the input port until that input port's internal 32-byte buffer fills up. Once the buffer lacks space for another packet, the input port deasserts `in_ready[i] = 0` to stall the upstream sender cleanly without dropping bytes.

---

### 1.4 End-to-End Lifecycle of a Packet

To understand how all modules interact, trace a single packet from arrival to exit:

1. **Stage 1 — Ingress & Buffering:**
An upstream device asserts `in_valid[i] = 1` and `in_sop[i] = 1` with Byte 0 (`HEADER`). As long as `in_ready[i] == 1`, bytes are written sequentially into Input Port `i`'s FIFO on each rising clock edge. Simultaneously, an ingress FSM tracks the byte count and computes a running XOR checksum across all incoming bytes.
2. **Stage 2 — Validation & Filtering:**
When `in_eop[i] == 1` is accepted, the ingress FSM checks two conditions:
   * Did the total received payload byte count match the `LENGTH` field (`1..16`)?
   * Does the running XOR of `HEADER`, `LENGTH`, and `PAYLOAD` equal the final `FCS` byte?
If **either fails**, the buffer pointers discard the packet, `malformed_cnt` increments, and the appropriate interrupt status bit (`len_err` or `fcs_err`) is latched. If **both pass**, the packet is marked *Ready for Arbitration*.


3. **Stage 3 — Destination Resolution & Arbitration:**
The input port decodes the target output port(s) from `HEADER` (`CTRL[0]` loopback override, `OPCODE == 01` broadcast, or `DEST_ID`).
   * If the target output port is disabled (`CTRL_REG.port_en[dest] == 0`), the packet is immediately discarded from the input FIFO, `drop_cnt` increments, and `IRQ_STAT_REG.drop_err` is set.
   * Otherwise, the input port raises a request (`req`) to the target output arbiter. When multiple inputs request the same output, the arbiter selects one winner based on `CTRL_REG.qos_en` and `CTRL_REG.arb_mode` (Fixed Priority or Round-Robin).


4. **Stage 4 — Egress Streaming & On-the-Fly Transformation:**
Once granted, the input FIFO streams bytes across the crossbar to the winning output port's egress stage:
* `HEADER` and `LENGTH` pass through unmodified.
* Each `PAYLOAD` byte passes through a lightweight combinational ALU controlled by `OPCODE`:
    * `00` (Unicast) / `01` (Broadcast): Payload passes unchanged.
    * `10` (XOR-Cipher): `out_byte = in_byte ^ KEY_REG[7:0]`.
    * `11` (Bit-Reverse): `out_byte = {in_byte[0], in_byte[1], ..., in_byte[7]}`.
* Because modifying the payload changes the packet's parity, the egress stage computes a **fresh running XOR checksum** over the outgoing `HEADER`, `LENGTH`, and *transformed* `PAYLOAD` bytes, replacing the final `FCS` byte on the `out_eop` beat.


5. **Stage 5 — Completion & Accounting:**
On the clock edge where `out_valid[j] && out_ready[j] && out_eop[j]` occurs:
   * The output port unlocks and returns to the `IDLE` state, ready for the next arbitration cycle.
   * The corresponding 16-bit packet counter (`PKT_CNT_01` or `PKT_CNT_23`) increments by 1 (saturating at `16'hFFFF`).
   * *(Note for Broadcasts: If a broadcast packet transmits to $K$ enabled output ports simultaneously, each of those $K$ output ports increments its own transmitted packet counter by 1.)*

---
### 1.5 Design Freedom vs. Mandatory Contracts

* **What is flexible (RTL Team's Choice):** You may organize your RTL hierarchy however you prefer (e.g., separate `input_port`, `arbiter`, `crossbar`, `output_port`, and `apb_slave` modules). You may implement the 32-byte input buffer using flip-flop arrays, dual-packet ping-pong buffers, or a circular byte FIFO with read/write/commit pointers.
* **What is strictly fixed (Contract with DV Team):** Module name (`router_4x4_top`), exact port names/widths, APB register addresses/bit-fields, packet byte ordering, error drop conditions, and `valid`/`ready` protocol rules must match this specification 100%.

---

## 2. Top-Level Interface (`router_4x4_top`)

### 2.1 Top-Level Module Declaration

The module name, parameter-free signature, port names, bit widths, and directions below are **strictly frozen**. Locking this boundary allows the DV team to instantiate the DUT in their UVM testbench and compile their environment immediately while the RTL team builds the internal logic.

```systemverilog
module router_4x4_top (
    input  logic        clk,
    input  logic        rst_n,        // Active-low synchronous reset

    // APB3 Configuration & Status Interface (Zero-Wait-State)
    input  logic        psel,
    input  logic        penable,
    input  logic        pwrite,
    input  logic [7:0]  paddr,
    input  logic [31:0] pwdata,
    output logic [31:0] prdata,
    output logic        pready,       // Tied to 1'b1 (single-cycle access)

    // 4x Input Packet Streaming Ports
    input  logic [3:0]  in_valid,
    output logic [3:0]  in_ready,
    input  logic [3:0]  in_sop,       // Start of Packet (asserted on Byte 0)
    input  logic [3:0]  in_eop,       // End of Packet (asserted on last byte)
    input  logic [7:0]  in_data [4],

    // 4x Output Packet Streaming Ports
    output logic [3:0]  out_valid,
    input  logic [3:0]  out_ready,
    output logic [3:0]  out_sop,
    output logic [3:0]  out_eop,
    output logic [7:0]  out_data [4],

    // Interrupt Output
    output logic        irq           // Active-high level interrupt
);

```

> **SystemVerilog Indexing Convention:** All 4-bit control vectors (`in_valid`, `in_ready`, `in_sop`, `in_eop`, `out_valid`, `out_ready`, `out_sop`, `out_eop`) map bit-for-bit to the unpacked byte arrays (`in_data [4]` and `out_data [4]`). For example, **Input Port `i`** (`i` $\in \{0, 1, 2, 3\}$) uses `in_valid[i]`, `in_ready[i]`, `in_sop[i]`, `in_eop[i]`, and `in_data[i][7:0]`.

---

### 2.2 Detailed Port Descriptions

#### A. Global Clock & Reset Signals

| Port Name | Width | Direction | Reset State | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `clk` | 1 | `input` | — | **System Clock.** All internal registers, FIFOs, FSMs, and output ports are triggered on the **rising edge** (`posedge clk`). |
| `rst_n` | 1 | `input` | — | **Active-Low Synchronous Reset.** When `rst_n == 1'b0` on a rising edge of `clk`, all FSMs return to `IDLE`, input FIFOs flush, registers load their default reset values, `out_valid` clears to `4'b0000`, `in_ready` deasserts to `4'b0000`, and `irq` clears to `1'b0`. Normal operation begins on the first `posedge clk` after `rst_n` returns to `1'b1`. |

---

#### B. APB3 Configuration & Status Interface

This interface implements an AMBA APB3 slave with zero wait states, used to program configuration registers and read traffic/error counters.

| Port Name | Width | Direction | Reset State | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `psel` | 1 | `input` | — | **Peripheral Select.** Asserted high (`1'b1`) by the APB master (testbench) to indicate that `router_4x4_top` is selected for a register read or write transfer. |
| `penable` | 1 | `input` | — | **Peripheral Enable.** Deasserted (`1'b0`) during the first cycle of a transfer (**Setup Phase**) and asserted (`1'b1`) during the second cycle (**Access Phase**) to strobe the transaction. |
| `pwrite` | 1 | `input` | — | **Transfer Direction.** `1'b1` indicates an APB **Write** (master writes `pwdata` into the register at `paddr`); `1'b0` indicates an APB **Read** (router drives register contents onto `prdata`). |
| `paddr` | 8 | `input` | — | **Register Byte Address.** Selects the target internal register (`0x00` through `0x18`, word-aligned on 4-byte boundaries). |
| `pwdata` | 32 | `input` | — | **Write Data Bus.** Carries the 32-bit value driven by the APB master during a write transfer (`pwrite == 1'b1`). |
| `prdata` | 32 | `output` | `32'h0` | **Read Data Bus.** Driven by the router during an APB read transfer (`psel && !pwrite`). Must hold valid register data during the Access Phase (`penable == 1'b1`). |
| `pready` | 1 | `output` | `1'b1` | **Slave Ready.** Indicates whether the router can complete the APB transfer immediately. Because all internal registers respond in a single cycle, tie `pready = 1'b1` permanently. |

---

#### C. Input Packet Streaming Ports (4 Independent Channels)

Each input port (`i = 0, 1, 2, 3`) acts as a **streaming slave (sink)** receiving byte-by-byte packets from an external source (or UVM Input Agent Driver).

| Port Name | Width | Direction | Reset State | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `in_valid[3:0]` | 4 | `input` | — | **Input Data Valid.** Driven high (`1'b1`) by the upstream sender when `in_data[i]`, `in_sop[i]`, and `in_eop[i]` hold valid packet information on Port `i`. |
| `in_ready[3:0]` | 4 | `output` | `4'b0000` | **Input Buffer Ready (Backpressure).** Driven high (`1'b1`) by the router when Input Port `i`'s internal FIFO has room to accept incoming bytes. Driven low (`1'b0`) during reset or when the input FIFO is full. |
| `in_sop[3:0]` | 4 | `input` | — | **Input Start of Packet.** Must be asserted high (`1'b1`) simultaneously with `in_valid[i]` strictly on **Byte 0 (`HEADER`)** of every packet, and held low (`1'b0`) for all subsequent bytes of that packet. |
| `in_eop[3:0]` | 4 | `input` | — | **Input End of Packet.** Must be asserted high (`1'b1`) simultaneously with `in_valid[i]` strictly on the **final byte (`FCS`)** of every packet, and held low (`1'b0`) for all earlier bytes. |
| `in_data [4]` | `4 × 8` | `input` | — | **Input Byte Data.** Four independent 8-bit data buses (`in_data[0]` to `in_data[3]`), each carrying one byte of the packet per valid clock beat. |

---

#### D. Output Packet Streaming Ports (4 Independent Channels)

Each output port (`j = 0, 1, 2, 3`) acts as a **streaming master (source)** transmitting routed packets to a downstream receiver (or UVM Output Agent Monitor/Responder).

| Port Name | Width | Direction | Reset State | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `out_valid[3:0]` | 4 | `output` | `4'b0000` | **Output Data Valid.** Driven high (`1'b1`) by the router when Output Port `j` is actively transmitting a valid packet byte on `out_data[j]`. |
| `out_ready[3:0]` | 4 | `input` | — | **Output Receiver Ready (Backpressure).** Driven by the downstream receiver. When `1'b1`, the receiver accepts the byte on `posedge clk`. When `1'b0`, the router must stall and hold all output signals for Port `j` unchanged. |
| `out_sop[3:0]` | 4 | `output` | `4'b0000` | **Output Start of Packet.** Driven high (`1'b1`) by the router strictly during the first beat (`HEADER` byte) of an outgoing packet on Port `j`; low (`1'b0`) otherwise. |
| `out_eop[3:0]` | 4 | `output` | `4'b0000` | **Output End of Packet.** Driven high (`1'b1`) by the router strictly during the final beat (`FCS` byte) of an outgoing packet on Port `j`; low (`1'b0`) otherwise. |
| `out_data [4]` | `4 × 8` | `output` | `'{4{8'h00}}` | **Output Byte Data.** Four independent 8-bit data buses (`out_data[0]` to `out_data[3]`), carrying transformed or unmodified packet bytes out of the switch. |

---

#### E. Interrupt Interface

| Port Name | Width | Direction | Reset State | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `irq` | 1 | `output` | `1'b0` | **Interrupt Request.** Active-high, level-sensitive interrupt output. Asserts (`1'b1`) whenever at least one enabled error flag is active: `irq = |(IRQ_STAT_REG[2:0] & IRQ_EN_REG[2:0])`. Remains high until software/testbench clears the corresponding bit(s) in `IRQ_STAT_REG` via a Write-1-to-Clear (`W1C`) APB write or masks them in `IRQ_EN_REG`. |

---

### 2.3 Streaming Interface Protocol & Timing Rules

Both the input ports (`in_*`) and output ports (`out_*`) strictly obey the standard **AXI-Stream / Avalon-ST style `valid`/`ready` handshake**:

1. **Beat Transfer Rule:** A single byte transfer ("beat") occurs on a rising clock edge **if and only if** `valid == 1'b1` **and** `ready == 1'b1` simultaneously. If `valid == 1'b1` while `ready == 1'b0`, the transfer is **stalled** (no data is consumed or advanced).
2. **Signal Stability Rule (No Retraction):** Once a sender asserts `valid = 1'b1`, it **must not** deassert `valid` or change `data`, `sop`, or `eop` until a clock edge occurs where `ready == 1'b1`. Violating this rule is a protocol error (and will trigger UVM SVA assertion failures).
3. **Packet Framing Rule:**
* `sop` and `eop` are only meaningful when `valid == 1'b1`.
* Every packet must start with `sop == 1'b1` (on Byte 0) and end with `eop == 1'b1` (on the last byte).
* `sop` and `eop` must **never** be asserted on the same cycle (`!(valid && sop && eop)`), because the minimum legal packet size is 4 bytes (1 Header + 1 Length + 1 Payload + 1 FCS).


4. **Back-to-Back Transfers:** After an `eop` beat completes (`valid && ready && eop`), a new packet may begin immediately on the very next clock cycle (`sop == 1'b1`) with zero idle cycles required between packets.

#### Example Streaming Timing Diagram (4-Byte Packet with 1 Stall Cycle)

The waveform below illustrates a minimum-length 4-byte packet (`HDR`, `LEN=1`, `PAY0`, `FCS`) where the receiver applies backpressure (`ready = 0`) during Cycle 3:

```text
Cycle:       |  1  |  2  |  3  |  4  |  5  |  6  |
              _   _   _   _   _   _   _   _   _   _
clk        __| |_| |_| |_| |_| |_| |_| |_| |_| |_| |_
              ___________________
valid      __|                   |___________________
              ___________     ___
ready      __|           |___|   |___________________
              ___
sop        __|   |___________________________________
                              _______
eop        __________________|       |_______________

data[7:0]  --<HDR><LEN><PAY0><PAY0><FCS>-------------
                       ^       ^     ^
                       |       |     +-- Beat 4: FCS transferred (eop=1)
                       |       +-------- Beat 3: PAY0 transferred (ready=1)
                       +---------------- STALL! ready=0; sender holds PAY0 & eop=0

```

---

### 2.4 APB3 Bus Protocol & Timing Rules

Because beginners frequently latch APB writes one cycle too early (during the Setup Phase), pay close attention to the two-phase APB3 protocol:

1. **Idle Phase (`psel == 0`, `penable == 0`):** No register access is occurring.
2. **Setup Phase (`psel == 1`, `penable == 0`):** The APB master places `paddr`, `pwrite`, and (for writes) `pwdata` onto the bus. **Do not modify internal registers during this cycle.**
3. **Access Phase (`psel == 1`, `penable == 1`, `pready == 1`):**
* **On a Write (`pwrite == 1`):** Sample `pwdata` and update the register selected by `paddr` on the rising edge of `clk` at the end of this cycle.
* **On a Read (`pwrite == 0`):** Drive the contents of the register selected by `paddr` onto `prdata` so the master samples valid data on the rising edge of `clk` at the end of this cycle.



#### Example APB3 Write Followed by APB3 Read

```text
Cycle:       |  1 (Setup) | 2 (Access) |  3 (Setup) | 4 (Access) |
              _   _   _   _   _   _   _   _   _   _   _   _   _   _
clk        __| |_| |_| |_| |_| |_| |_| |_| |_| |_| |_| |_| |_| |_
              _________________________   _________________________
psel       __|                         |_|                         |_
                          _____________               _____________
penable    ______________|             |_____________|             |_
              _________________________
pwrite     __|                         |_____________________________

paddr      --<   0x04 (KEY_REG)        >--<   0x04 (KEY_REG)        >

pwdata     --<      32'h0000_00A5      >-----------------------------

prdata     -------------------------------------------< 32'h0000_00A5 >
                                       ^                           ^
                                       |                           +-- Read sampled
                                       +-- Write committed to KEY_REG

```

## 3. Packet Frame Format & Operations

Every packet consists of **4 to 19 bytes**:

| Byte Offset | Field | Bit Width | Description |
| --- | --- | --- | --- |
| **Byte 0** | **HEADER** | `[7:0]` | `[7:6]` **DEST_ID**: Target output port (`0` to `3`).|
|`[5:4]` **OPCODE**: Packet operation (see table below).|
|`[3:2]` **PRIO**: Priority level (`0` = Lowest, `3` = Highest).|
|`[1:0]` **CTRL**: `CTRL[0] == 1` enables **Loopback** (overrides `DEST_ID` and routes back to the source port ID). `CTRL[1]` is reserved (`0`). |
| **Byte 1** | **LENGTH** | `[7:0]` | Number of payload bytes (`N`). Valid range: **`1` to `16**`. |
| **Bytes 2 .. N+1** | **PAYLOAD** | `N × 8` | Packet data bytes (`Payload[0]` to `Payload[N-1]`). |
| **Byte N+2** | **FCS** | `[7:0]` | **Frame Check Sequence:** Bitwise XOR of all preceding bytes (`HEADER ^ LENGTH ^ PAYLOAD[0] ^ ... ^ PAYLOAD[N-1]`). |

### Supported Operations (`OPCODE`)

| `OPCODE` | Operation Name | Phase | Hardware Behavior |
| --- | --- | --- | --- |
| `2'b00` | **Unicast** | Phase 1 | Routes packet unmodified to the port specified by `DEST_ID` (or source port if `CTRL[0] == 1`). |
| `2'b01` | **Broadcast** | Phase 1 | Replicates and routes packet unmodified to **all currently enabled** output ports (`CTRL_REG.port_en[i] == 1`). |
| `2'b10` | **XOR-Cipher** | Phase 2 | Routes to `DEST_ID`, XORing every **Payload** byte with `KEY_REG[7:0]`. Recomputes `FCS` at the output to match the transformed payload. |
| `2'b11` | **Bit-Reverse** | Phase 2 | Routes to `DEST_ID`, reversing the bit order of every **Payload** byte (`data[0..7] -> data[7..0]`). Recomputes `FCS` at the output. |

---

## 4. APB Register Map (`0x00` – `0x18`)

* **Access Types:** `RW` = Read/Write, `RO` = Read-Only, `W1C` = Write-1-to-Clear.
* Reads from unmapped addresses return `32'h0000_0000`; writes to unmapped addresses are ignored.

| Offset | Register Name | Access | Reset Value | Bit Descriptions |
| --- | --- | --- | --- | --- |
| `0x00` | `CTRL_REG` | `RW` | `32'h0000_000F` | `[3:0]` **`port_en`**: Output port enable mask (`1` = enabled, `0` = disabled).|
|`[4]` **`arb_mode`**: `0` = Fixed Priority (`In 0 > In 1 > In 2 > In 3`), `1` = Round-Robin.|
|`[5]` **`qos_en`**: `0` = Ignore `PRIO` field, `1` = Strict `PRIO` arbitration first, tie-break with `arb_mode`.|
|`[31:6]` Reserved (`0`). |
| `0x04` | `KEY_REG` | `RW` | `32'h0000_0000` | `[7:0]` **`xor_key`**: 8-bit cipher key used when `OPCODE == 2'b10`.|
|`[31:8]` Reserved (`0`). |
| `0x08` | `IRQ_EN_REG` | `RW` | `32'h0000_0000` | `[0]` **`fcs_err_en`**: Enable interrupt on FCS mismatch.|
|`[1]` **`len_err_en`**: Enable interrupt on Length error.|
|`[2]` **`drop_err_en`**: Enable interrupt on Dropped packet (disabled port or FIFO full). |
| `0x0C` | `IRQ_STAT_REG` | `W1C` | `32'h0000_0000` | `[0]` **`fcs_err`**: Set to `1` when a bad FCS packet is detected.|
|`[1]` **`len_err`**: Set to `1` when a length error is detected.|
|`[2]` **`drop_err`**: Set to `1` when a valid packet is dropped.|
|*Note: `irq = |
| `0x10` | `PKT_CNT_01` | `RO` | `32'h0000_0000` | `[31:16]` Output Port 1 transmitted packet count (saturating).|
|`[15:0]` Output Port 0 transmitted packet count (saturating). |
| `0x14` | `PKT_CNT_23` | `RO` | `32'h0000_0000` | `[31:16]` Output Port 3 transmitted packet count (saturating).|
|`[15:0]` Output Port 2 transmitted packet count (saturating). |
| `0x18` | `ERR_CNT_REG` | `RO` | `32'h0000_0000` | `[31:16]` **`drop_cnt`**: Total dropped packets (disabled port / buffer full).|
|`[15:0]` **`malformed_cnt`**: Total malformed packets (bad FCS / bad length). |

---

## 5. Corner-Case & Error Rules (Mandatory Behavior)

To ensure the RTL implementation and the UVM reference model match cycle-for-cycle in behavior:

1. **Input Buffer Capacity:** Each input port must buffer at least **32 bytes** (capable of holding one max-size 19-byte packet while receiving the start of the next). If an input buffer does not have enough free space when a new `sop` arrives, `in_ready[i]` must be driven low (`0`) to apply backpressure.
2. **Malformed Packet Drop (Phase 2):** A packet is dropped silently (never forwarded to any output) and `malformed_cnt` increments if:
* `LENGTH == 0` or `LENGTH > 16`.
* `eop` arrives earlier or later than `Byte (LENGTH + 2)`.
* Computed XOR checksum of `Bytes 0 .. (LENGTH + 1)` does not match the `FCS` byte.


3. **Disabled Destination Drop (Phase 2):**
* **Unicast / Transform (`OPCODE != 2'b01`):** If the target output port has `CTRL_REG.port_en[dest] == 0` at the time of arbitration, drop the packet, increment `drop_cnt`, and set `IRQ_STAT_REG.drop_err`.
* **Broadcast (`OPCODE == 2'b01`):** Forward only to ports where `port_en[i] == 1`. If **all** 4 ports are disabled (`port_en == 4'b0000`), drop the packet and increment `drop_cnt`.


4. **Broadcast Synchronization:** A Broadcast packet waits until **all** currently enabled output ports are idle/unlocked, locks all of them simultaneously, and transmits identically across them (stalling if any enabled output deasserts `out_ready`).

---

## 6. Project Milestones (2-Week Schedule)

| Milestone | RTL Team Deliverables | DV Team (UVM) Deliverables |
| --- | --- | --- |
| **Phase 1**|
|*(End of Week 1)* | • Top-level module matching exact Section 2 pinout.|
|• Working APB slave (`CTRL_REG`, `KEY_REG`, `PKT_CNT_*`).|
|• Store-and-forward input buffers.|
|• Fixed-priority arbitration (`arb_mode = 0`).|
|• Working `Unicast (00)` and `Broadcast (01)`.|
|• Directed SV testbench proving basic routing & backpressure. | • Complete UVM testbench topology (`env`, `apb_agent`, 4x `in_pkt_agent`, 4x `out_pkt_agent`).|
|• Constrained `packet_seq_item` (valid headers, lengths `1..16`, auto-computed `FCS`).|
|• SVA protocol assertions (`valid`/`ready` stability, `sop`/`eop` framing).|
|• Reference predictor & scoreboard passing basic Unicast/Broadcast tests. |
| **Phase 2**|
|*(End of Week 2)* | • `XOR-Cipher (10)`, `Bit-Reverse (11)` + FCS recalculation, and `Loopback`.|
|• Round-Robin arbitration (`arb_mode = 1`) and `QoS` (`qos_en = 1`).|
|• Full malformed/disabled error dropping, `ERR_CNT_REG`, and `IRQ` logic.|
|• All DV-reported bugs resolved. | • Stress sequences: 4-to-1 port collisions, back-to-back traffic, random `out_ready` stalls.|
|• Error injection sequences: bad FCS, invalid lengths (`0`, `17+`), disabled ports.|
|• Functional `covergroup` closing 100% cross-coverage (`OPCODE × PRIO × DEST_ID × Error_Type`).|
|• Bug reports logged with reproducible seeds. |