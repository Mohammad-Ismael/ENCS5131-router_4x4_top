# Hardware Specification of 4x4 Smart Packet Router (`router_4x4_top`)

## 1. Overview & Architecture

### 1.1 Introduction

The `router_4x4_top` is a **4-port packet switch** commonly found in Network-on-Chip (NoC) interconnects and network interface controllers. Its job is to accept variable-length byte packets arriving simultaneously on four independent input channels (`in_0` through `in_3`), inspect each packet's header to determine its destination and required processing, resolve traffic conflicts when multiple inputs target the same output, and transmit valid packets out of the four output channels (`out_0` through `out_3`).

The hardware is divided into two distinct functional parts:

   1. **The Data part (Streaming Interfaces):** Four input ports and four output ports operating on an 8-bit (`1 byte/cycle`) `valid`/`ready` handshake protocol. All ports operate synchronously on the same system clock (`clk`).
   2. **The Control & Telemetry part (APB3 Slave Interface):** A memory-mapped register interface (`psel`, `penable`, `pwrite`, `paddr`, `pwdata`, `prdata`) that allows a host CPU (or UVM testbench) to enable/disable ports, select arbitration algorithms, load encryption keys, read traffic/error statistics, and service interrupts (`irq`).

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

### 3.1 Visual Packet Frame Layout

Every packet transmitted into or out of `router_4x4_top` is a contiguous sequence of **4 to 19 bytes** (`1` Header byte + `1` Length byte + `1..16` Payload bytes + `1` Checksum byte).

```text
  Beat 0 (sop = 1)                                               Beat N+2 (eop = 1)
 +----------------+----------------+------------+-----+------------+----------------+
 |     Byte 0     |     Byte 1     |   Byte 2   | ... |  Byte N+1  |    Byte N+2    |
 |     HEADER     |     LENGTH     | PAYLOAD[0] |     |PAYLOAD[N-1]|      FCS       |
 +----------------+----------------+------------+-----+------------+----------------+
  /              \
 /                \
+----+----+----+----+----+----+----+----+
|  7 |  6 |  5 |  4 |  3 |  2 |  1 |  0 |
+----+----+----+----+----+----+----+----+
| DEST_ID | OPCODE  |  PRIO   |  CTRL   |
+---------+---------+---------+---------+

```

---

### 3.2 Detailed Field Specifications

#### A. Byte 0: `HEADER` (`[7:0]`)

The `HEADER` byte carries all routing, priority, and processing instructions for the packet:

| Bit Slice | Field Name | Width | Legal Values | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[7:6]` | **`DEST_ID`** | 2 | `0` to `3` | **Destination Output Port ID.** Selects target output port `out_0` (`2'b00`), `out_1` (`2'b01`), `out_2` (`2'b10`), or `out_3` (`2'b11`). Ignored when `OPCODE == 2'b01` (Broadcast) or `CTRL[0] == 1'b1` (Loopback). |
| `[5:4]` | **`OPCODE`** | 2 | `0` to `3` | **Packet Operation Code.** Defines how the packet is routed and whether payload bytes are transformed (`00` = Unicast, `01` = Broadcast, `10` = XOR-Cipher, `11` = Bit-Reverse). |
| `[3:2]` | **`PRIO`** | 2 | `0` to `3` | **Quality-of-Service (QoS) Priority.** `2'b00` is lowest priority; `2'b11` is highest priority. Used by the arbiter only when `CTRL_REG.qos_en == 1'b1`. |
| `[1:0]` | **`CTRL`** | 2 | `0` to `3` | **Control Flags:**<br>

<br>• **`CTRL[0]` (Loopback Enable):** When `1'b1`, overrides `DEST_ID` and routes the packet back to the output port matching its arrival input port (`in_i` $\rightarrow$ `out_i`).<br>

<br>• **`CTRL[1]` (Reserved):** Unused by hardware; must not trigger an error if non-zero and must pass through unmodified. |

> **Header Preservation Rule:** Regardless of routing mode (`DEST_ID`, `Loopback`, or `Broadcast`), **`Byte 0 (HEADER)` and `Byte 1 (LENGTH)` are never modified** as they pass through the router. Downstream receivers rely on seeing the original `HEADER` bits.

---

#### B. Byte 1: `LENGTH` (`[7:0]`)

* Specifies the exact number of **Payload bytes (`N`)** that follow Byte 1.
* **Valid Range:** `8'd1` (`8'h01`) through `8'd16` (`8'h10`).
* **Total Packet Length:** Because every packet also includes 3 overhead bytes (`HEADER`, `LENGTH`, and `FCS`), the total number of clock beats in a valid packet is always **`N + 3`** (minimum 4 beats, maximum 19 beats).
* **Error Condition:** Any packet arriving with `LENGTH == 8'd0` or `LENGTH > 8'd16`, or whose `eop` signal arrives earlier or later than beat `N + 2`, is flagged as a **Length Error** (`len_err`) and dropped.

---

#### C. Bytes `2` to `N+1`: `PAYLOAD` (`N × 8` bits)

* Carries `N` raw data bytes (`PAYLOAD[0]` through `PAYLOAD[N-1]`).
* Depending on `OPCODE`, these bytes either pass through untouched (`OPCODE = 00, 01`) or undergo combinational transformation (`OPCODE = 10, 11`) as they exit the crossbar.

---

#### D. Byte `N+2`: `FCS` — Frame Check Sequence (`[7:0]`)

* An 8-bit longitudinal parity checksum used to detect data corruption.
* **Formula:** Bitwise XOR ($\oplus$) of `HEADER`, `LENGTH`, and every `PAYLOAD` byte:

$$\text{FCS} = \text{HEADER} \oplus \text{LENGTH} \oplus \text{PAYLOAD}[0] \oplus \text{PAYLOAD}[1] \oplus \dots \oplus \text{PAYLOAD}[N-1]$$


* **Ingress Check:** Every incoming packet's `Byte N+2` must equal the XOR of `Bytes 0 .. N+1`. If it does not match, the packet is flagged with an **FCS Error** (`fcs_err`) and dropped.
* **Egress Regeneration:** When `OPCODE` modifies the payload (`2'b10` or `2'b11`), the router **recomputes `FCS**` over the outgoing `HEADER`, `LENGTH`, and *transformed* `PAYLOAD` bytes so the outgoing packet remains valid for downstream receivers.

---

### 3.3 Supported Packet Operations (`OPCODE`) & Precedence Rules

| `OPCODE` | Operation Name | Phase | Routing Target | Payload Transformation | Output `FCS` Behavior |
| --- | --- | --- | --- | --- | --- |
| `2'b00` | **Unicast** | Phase 1 | `CTRL[0] ? src_port : DEST_ID` | **None:** `out_byte = in_byte` | Forwarded unchanged |
| `2'b01` | **Broadcast** | Phase 1 | **All** ports where `port_en[j] == 1` | **None:** `out_byte = in_byte` | Forwarded unchanged |
| `2'b10` | **XOR-Cipher** | Phase 2 | `CTRL[0] ? src_port : DEST_ID` | **XOR with Key:** `out_byte = in_byte ^ KEY_REG[7:0]` | **Recomputed** over new payload |
| `2'b11` | **Bit-Reverse** | Phase 2 | `CTRL[0] ? src_port : DEST_ID` | **Bit Mirror:** `out_byte = {in_byte[0], in_byte[1], in_byte[2], in_byte[3], in_byte[4], in_byte[5], in_byte[6], in_byte[7]}` | **Recomputed** over new payload |

#### Important Operational Clarifications (Preventing RTL/DV Mismatches)

1. **Broadcast vs. Loopback Precedence:** `OPCODE == 2'b01` (**Broadcast**) takes precedence over `CTRL[0]` (**Loopback**). If a packet has `OPCODE == 2'b01` and `CTRL[0] == 1'b1`, it is broadcast to all enabled ports (not looped back to a single port).
2. **Loopback + Payload Transformation:** For `OPCODE == 2'b10` (XOR-Cipher) and `OPCODE == 2'b11` (Bit-Reverse), setting `CTRL[0] == 1'b1` routes the transformed packet back to its source port (`out_i`) with the payload transformation and new `FCS` applied.
3. **`KEY_REG` Stability:** When using `OPCODE == 2'b10`, `KEY_REG[7:0]` is sampled at the start of packet egress (`out_sop`) and held constant for the duration of that packet's transmission. (Testbenches should avoid modifying `KEY_REG` while traffic is actively draining.)

---

### 3.4 Step-by-Step Worked Examples

#### Example A: Phase 1 Unicast Packet (3 Payload Bytes, 6 Beats Total)

Suppose **Input Port 1** receives a Unicast packet destined for **Output Port 2** with Priority 3 and Payload `{8'h10, 8'h24, 8'h55}`:

1. **Construct `HEADER`:**
* `DEST_ID` = `2'b10` (Port 2), `OPCODE` = `2'b00` (Unicast), `PRIO` = `2'b11` (3), `CTRL` = `2'b00` (Normal).
* `HEADER` = `8'b10_00_11_00` = **`8'h8C`**.


2. **Construct `LENGTH`:** `3` payload bytes $\rightarrow$ **`8'h03`**.
3. **Compute `FCS`:**
* `0x8C ^ 0x03` = `0x8F`
* `0x8F ^ 0x10` = `0x9F`
* `0x9F ^ 0x24` = `0xBB`
* `0xBB ^ 0x55` = **`8'hEE`**


4. **Wire Trace (Ingress on `in_1` $\rightarrow$ Egress on `out_2`):**
* Beat 0 (`sop=1, eop=0`): `0x8C`
* Beat 1 (`sop=0, eop=0`): `0x03`
* Beat 2 (`sop=0, eop=0`): `0x10`
* Beat 3 (`sop=0, eop=0`): `0x24`
* Beat 4 (`sop=0, eop=0`): `0x55`
* Beat 5 (`sop=0, eop=1`): `0xEE`



---

#### Example B: Phase 2 XOR-Cipher + Loopback Packet (1 Payload Byte, 4 Beats Total)

Suppose `KEY_REG[7:0]` is programmed to **`8'hAA`**, and **Input Port 3** receives a 1-byte XOR-Cipher packet with **Loopback enabled** (`CTRL[0] = 1`):

1. **Construct `HEADER`:**
* `DEST_ID` = `2'b01` (ignored due to Loopback), `OPCODE` = `2'b10` (XOR-Cipher), `PRIO` = `2'b01` (1), `CTRL` = `2'b01` (Loopback).
* `HEADER` = `8'b01_10_01_01` = **`8'h65`**.


2. **Ingress Bytes on `in_3`:**
* Beat 0 (`sop=1`): `0x65` (`HEADER`)
* Beat 1: `0x01` (`LENGTH = 1`)
* Beat 2: `0x3C` (`PAYLOAD[0]`)
* Beat 3 (`eop=1`): `0x58` (`FCS = 0x65 ^ 0x01 ^ 0x3C = 0x58`)


3. **Internal Processing & Egress Bytes on `out_3`:**
* Because `CTRL[0] == 1` and the packet arrived on `in_3`, the router targets **`out_3`**.
* `PAYLOAD[0]` is XORed with `KEY_REG`: `0x3C ^ 0xAA` = **`0x96`**.
* `FCS` is recomputed over the outgoing bytes: `0x65 ^ 0x01 ^ 0x96` = **`0xF2`**.
* **Transmitted on `out_3`:** `0x65` (`sop=1`), `0x01`, `0x96`, `0xF2` (`eop=1`).


## 4. APB Register Map (`0x00` – `0x18`)

### 4.1 Register Map Overview

All control, status, and telemetry registers reside in a 28-byte address space (`0x00` through `0x18`) accessed via the 32-bit APB3 slave interface.

| Offset | Register Name | Access | Reset Value | Summary Description |
| --- | --- | --- | --- | --- |
| `0x00` | **`CTRL_REG`** | `RW` | `32'h0000_000F` | Output port enable mask, arbitration mode, and QoS enable. |
| `0x04` | **`KEY_REG`** | `RW` | `32'h0000_0000` | 8-bit encryption key for `XOR-Cipher` (`OPCODE = 2'b10`) packets. |
| `0x08` | **`IRQ_EN_REG`** | `RW` | `32'h0000_0000` | Interrupt mask bits for FCS, Length, and Drop error events. |
| `0x0C` | **`IRQ_STAT_REG`** | `W1C` | `32'h0000_0000` | Sticky error status flags (Write-1-to-Clear). |
| `0x10` | **`PKT_CNT_01`** | `RO` | `32'h0000_0000` | Transmitted packet counters for Output Port 1 (`[31:16]`) and Port 0 (`[15:0]`). |
| `0x14` | **`PKT_CNT_23`** | `RO` | `32'h0000_0000` | Transmitted packet counters for Output Port 3 (`[31:16]`) and Port 2 (`[15:0]`). |
| `0x18` | **`ERR_CNT_REG`** | `RO` | `32'h0000_0000` | Dropped packet counter (`[31:16]`) and malformed packet counter (`[15:0]`). |

---

### 4.2 Bus Rules & Access Policy Definitions

1. **32-Bit Word Transfers Only:** Because APB3 does not include byte-strobe (`pstrb`) signals, every write updates the full 32-bit register at `paddr`.
2. **Unmapped & Unaligned Addresses:**
* Reading any address not listed in Section 4.1 (including unaligned addresses such as `0x01`, `0x02`, or addresses `> 0x18`) must return `prdata = 32'h0000_0000` without error.
* Writing to any unmapped address is silently ignored.


3. **Standard Field Access Policies (UVM RAL Compatible):**
* **`RW` (Read / Write):** Software can read and write the field normally.
* **`RO` (Read-Only):** Hardware updates the field; software writes have no effect.
* **`W1C` (Write-1-to-Clear):** Hardware sets the bit to `1'b1` when an event occurs. Software clears the bit by writing `1'b1` to that specific bit position. Writing `1'b0` leaves the bit unchanged (allowing software to clear one flag without accidentally clearing others).
* **`RSVD` (Reserved):** Unused bits. Must always read back as `0` (**Read-As-Zero**) and ignore writes (**Writes-Ignored**).



---

### 4.3 Detailed Register Definitions

#### 1. `CTRL_REG` — Router Control Register (`Offset: 0x00`, `Reset: 32'h0000_000F`)

Controls output port gating and crossbar arbitration behavior. Note that **`port_en` resets to `4'b1111` (`0xF`)** so all four output ports are immediately active out of reset without requiring APB configuration first.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:6]` | `RSVD` | `RO` | `26'h0` | Reserved. Always reads `0`. |
| `[5]` | **`qos_en`** | `RW` | `1'b0` | **Quality-of-Service Priority Enable:**|
|• `1'b0`: Ignore the packet `HEADER.PRIO` field; arbitrate strictly using `arb_mode`.|
|• `1'b1`: Compare packet `HEADER.PRIO` first (`3` highest, `0` lowest), and use `arb_mode` only to break ties between packets of equal priority. |
| `[4]` | **`arb_mode`** | `RW` | `1'b0` | **Arbitration Algorithm Select:**|
|• `1'b0`: **Fixed Priority** (`Input 0` > `Input 1` > `Input 2` > `Input 3`).|
|• `1'b1`: **Round-Robin** (fair rotating priority per output port). |
| `[3:0]` | **`port_en`** | `RW` | `4'hF` | **Output Port Enable Mask (1 bit per output port):**|
|• `port_en[j] == 1'b1`: Output Port `j` is enabled and accepts packets.|
|• `port_en[j] == 1'b0`: Output Port `j` is disabled. Unicast/transform packets targeting Port `j` are dropped; Broadcast packets skip Port `j`. |

---

#### 2. `KEY_REG` — Payload XOR-Cipher Key Register (`Offset: 0x04`, `Reset: 32'h0000_0000`)

Stores the 8-bit cipher mask applied to payload bytes when `HEADER.OPCODE == 2'b10`.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:8]` | `RSVD` | `RO` | `24'h0` | Reserved. Always reads `0`. |
| `[7:0]` | **`xor_key`** | `RW` | `8'h00` | **XOR Encryption Key:** Each payload byte of a packet with `OPCODE == 2'b10` is bitwise XORed with `xor_key[7:0]` during egress. |

---

#### 3. `IRQ_EN_REG` — Interrupt Enable Register (`Offset: 0x08`, `Reset: 32'h0000_0000`)

Masks or unmasks individual error flags in `IRQ_STAT_REG` from asserting the top-level `irq` output pin.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:3]` | `RSVD` | `RO` | `29'h0` | Reserved. Always reads `0`. |
| `[2]` | **`drop_err_en`** | `RW` | `1'b0` | `1'b1` = Allow `IRQ_STAT_REG.drop_err` to assert `irq`; `1'b0` = Masked. |
| `[1]` | **`len_err_en`** | `RW` | `1'b0` | `1'b1` = Allow `IRQ_STAT_REG.len_err` to assert `irq`; `1'b0` = Masked. |
| `[0]` | **`fcs_err_en`** | `RW` | `1'b0` | `1'b1` = Allow `IRQ_STAT_REG.fcs_err` to assert `irq`; `1'b0` = Masked. |

---

#### 4. `IRQ_STAT_REG` — Interrupt Status Register (`Offset: 0x0C`, `Reset: 32'h0000_0000`)

Latches sticky error flags whenever a packet error or drop event occurs. Even if `IRQ_EN_REG` is `0`, hardware **must still set** these status bits so software can poll them.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:3]` | `RSVD` | `RO` | `29'h0` | Reserved. Always reads `0`. |
| `[2]` | **`drop_err`** | `W1C` | `1'b0` | Set to `1'b1` by hardware whenever a valid packet is dropped because its target output port(s) are disabled (`port_en == 0`). Write `1` to clear. |
| `[1]` | **`len_err`** | `W1C` | `1'b0` | Set to `1'b1` by hardware whenever a packet has `LENGTH == 0`, `LENGTH > 16`, or `eop` arrives on the wrong byte count. Write `1` to clear. |
| `[0]` | **`fcs_err`** | `W1C` | `1'b0` | Set to `1'b1` by hardware whenever a packet with a valid length fails the `FCS` XOR checksum verification. Write `1` to clear. |

**Top-Level Interrupt Output (`irq`) Equation:**

```systemverilog
assign irq = (irq_stat_reg[0] & irq_en_reg[0]) |
             (irq_stat_reg[1] & irq_en_reg[1]) |
             (irq_stat_reg[2] & irq_en_reg[2]);

```

---

#### 5. `PKT_CNT_01` — Output Ports 0 & 1 Packet Counter (`Offset: 0x10`, `Reset: 32'h0000_0000`)

Tracks the total number of complete packets successfully transmitted out of `out_0` and `out_1`.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:16]` | **`port1_tx_cnt`** | `RO` | `16'h0000` | Increments by `1` on every `out_valid[1] && out_ready[1] && out_eop[1]` handshake. Saturates at `16'hFFFF`. |
| `[15:0]` | **`port0_tx_cnt`** | `RO` | `16'h0000` | Increments by `1` on every `out_valid[0] && out_ready[0] && out_eop[0]` handshake. Saturates at `16'hFFFF`. |

---

#### 6. `PKT_CNT_23` — Output Ports 2 & 3 Packet Counter (`Offset: 0x14`, `Reset: 32'h0000_0000`)

Tracks the total number of complete packets successfully transmitted out of `out_2` and `out_3`.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:16]` | **`port3_tx_cnt`** | `RO` | `16'h0000` | Increments by `1` on every `out_valid[3] && out_ready[3] && out_eop[3]` handshake. Saturates at `16'hFFFF`. |
| `[15:0]` | **`port2_tx_cnt`** | `RO` | `16'h0000` | Increments by `1` on every `out_valid[2] && out_ready[2] && out_eop[2]` handshake. Saturates at `16'hFFFF`. |

---

#### 7. `ERR_CNT_REG` — Dropped & Malformed Packet Counter (`Offset: 0x18`, `Reset: 32'h0000_0000`)

Provides separate 16-bit telemetry counters for packets dropped due to routing/port configuration versus packets rejected due to frame corruption.

| Bits | Field Name | Access | Reset | Detailed Functional Description |
| --- | --- | --- | --- | --- |
| `[31:16]` | **`drop_cnt`** | `RO` | `16'h0000` | Total valid packets dropped due to disabled target port(s) (`port_en == 0`). Increments by `1` per dropped packet. Saturates at `16'hFFFF`. |
| `[15:0]` | **`malformed_cnt`** | `RO` | `16'h0000` | Total incoming packets rejected due to a Length error (`len_err`) or FCS checksum error (`fcs_err`). Increments by `1` per rejected packet. Saturates at `16'hFFFF`. |

---

### 4.4 Critical Register Edge Cases (Preventing RTL / UVM Mismatches)

1. **Simultaneous Hardware Set vs. Software `W1C` Clear:**
If a hardware error event (`hw_err_pulse`) occurs on the **exact same clock cycle** that software performs an APB write of `1'b1` to clear that same bit in `IRQ_STAT_REG`, **the hardware set must take priority** so the new error event is never lost:
```systemverilog
// Recommended SystemVerilog pattern for W1C bits:
logic [2:0] w1c_clear;
assign w1c_clear = (apb_write && (paddr == 8'h0C)) ? pwdata[2:0] : 3'b000;

always_ff @(posedge clk) begin
    if (!rst_n)
        irq_stat_reg <= 3'b000;
    else
        irq_stat_reg <= (irq_stat_reg & ~w1c_clear) | hw_err_pulse[2:0];
end

```


2. **Simultaneous Multi-Port Errors on the Same Cycle:**
If two input ports (e.g., `in_0` and `in_2`) simultaneously detect a malformed packet on the exact same clock edge, `malformed_cnt` must increment by **`2`** (i.e., count the total number of rejected packets across all 4 input ports, not just `1` per clock cycle). The same applies to `drop_cnt` if multiple input ports drop packets on the same cycle.
3. **Single-Packet Error Precedence (`len_err` vs. `fcs_err`):**
Length validation is performed **before** FCS validation. If an incoming packet has an invalid length (`LENGTH == 0`, `LENGTH > 16`, or mismatched `eop` position), hardware asserts `len_err` **only** (do not also assert `fcs_err` for that same packet) and increments `malformed_cnt` by `1`. `fcs_err` is only evaluated when the packet frame length is valid.
4. **Saturating Counters (No Wrap-Around):**
All six 16-bit counters (`port0..3_tx_cnt`, `drop_cnt`, `malformed_cnt`) must **saturate at `16'hFFFF**` (`65,535`) rather than wrapping around to `16'h0000`. They can only be cleared by asserting `rst_n == 1'b0`.

## 5. Corner-Case, Arbitration & Error Rules (Mandatory Behavior)

To ensure the RTL implementation and the UVM golden reference model behave identically down to the clock cycle, both teams must adhere strictly to the buffer management, arbitration, error-handling, and broadcast rules defined below.

---

### 5.1 Input Buffering, Packet Framing & Backpressure Rules

#### A. 32-Byte Storage & Metadata Architecture

Each of the four input ports (`in_0` through `in_3`) must implement a **32-byte Store-and-Forward buffer**.

* **Why 32 bytes?** A maximum-length packet is **19 bytes** (`1 Header + 1 Length + 16 Payload + 1 FCS`). Providing 32 bytes of storage guarantees that an input port can hold one complete maximum-size packet waiting for arbitration while simultaneously absorbing up to 13 bytes of the next incoming packet without stalling.
* **Beginner Implementation Tip (Data FIFO + Packet Descriptor Queue):** Because multiple minimum-length packets (4 bytes each) can fit inside a 32-byte buffer at the same time (up to $32 / 4 = 8$ packets), beginners should pair their **32-byte data FIFO** with a small **8-entry Packet Descriptor FIFO**. Each descriptor entry stores the validated packet's `HEADER`, `LENGTH`, and status once `eop` passes verification.

#### B. Backpressure Assertion (`in_ready[i]`)

* **Normal Operation:** Input Port `i` drives `in_ready[i] = 1'b1` whenever its byte buffer has at least **1 free byte** available (`free_bytes > 0`) and its descriptor queue is not full.
* **Buffer Full:** When `free_bytes == 0` (all 32 byte slots are occupied by unconsumed/in-progress bytes), Input Port `i` must immediately drive `in_ready[i] = 1'b0`.
* **Reset:** While `rst_n == 1'b0`, `in_ready[3:0]` must be driven to `4'b0000`.

#### C. Packet Rollback on Malformed Ingress

Because `router_4x4_top` is Store-and-Forward, bytes of an incoming packet are written into the input buffer *before* the `FCS` byte arrives at the end.

* **Write-Pointer Rollback Rule:** Maintain a `commit_wr_ptr` (pointing to the end of the last *verified valid* packet) and a speculative `temp_wr_ptr` (advancing as bytes of the current packet arrive).
* If the current packet passes Length and FCS checks on `eop`, update `commit_wr_ptr <= temp_wr_ptr` and push an entry into the Packet Descriptor FIFO.
* If the current packet fails Length or FCS checks, **revert `temp_wr_ptr <= commit_wr_ptr**`. This cleanly erases the corrupted packet in a single cycle without disturbing earlier valid packets already queued ahead of it.



---

### 5.2 Arbitration Rules: Fixed Priority, Round-Robin & QoS

Each Output Port `j` (`j` $\in \{0, 1, 2, 3\}$) has its own independent **4-to-1 Arbiter** that selects among input ports requesting Output `j`. Arbitration occurs **only when Output Port `j` is `IDLE` (not currently locked mid-packet)**.

#### Mode 1: Fixed Priority (`CTRL_REG.qos_en == 0`, `CTRL_REG.arb_mode == 0`)

* Input ports have a static, unchanging priority order:

$$\text{Input } 0 \text{ (Highest)} > \text{Input } 1 > \text{Input } 2 > \text{Input } 3 \text{ (Lowest)}$$


* Among all inputs with a valid head-of-line packet targeting Output `j`, the lowest-numbered input port always wins.

#### Mode 2: Round-Robin (`CTRL_REG.qos_en == 0`, `CTRL_REG.arb_mode == 1`)

* Each Output Port `j` maintains a 2-bit rotating priority pointer **`rr_ptr[j]`**, which resets to **`2'd0`** when `rst_n == 1'b0`.
* When Output Port `j` is `IDLE` and one or more inputs request it, the arbiter scans inputs in circular order starting at `rr_ptr[j]`:

$$\text{Order: } \quad \text{rr\textunderscore ptr}[j] \;\rightarrow\; (\text{rr\textunderscore ptr}[j] + 1) \bmod 4 \;\rightarrow\; (\text{rr\textunderscore ptr}[j] + 2) \bmod 4 \;\rightarrow\; (\text{rr\textunderscore ptr}[j] + 3) \bmod 4$$


* The first requesting input encountered in this circular scan wins the grant.
* **Pointer Update Rule:** When Input Port `w` wins arbitration on Output Port `j`, **`rr_ptr[j]` updates to `(w + 1) % 4**` at the start of packet transmission (`out_sop[j]`). `rr_ptr[j]` **must not** change on idle cycles or during subsequent beats of a locked packet.

#### Mode 3: Quality-of-Service Priority (`CTRL_REG.qos_en == 1`)

When `qos_en == 1'b1`, packet priority (`HEADER.PRIO`, where `3` is highest and `0` is lowest) overrides port index:

1. **Step 1 (Priority Filtering):** Inspect the `PRIO` field of all input ports currently requesting Output Port `j`, and determine `max_prio` (the highest `PRIO` among active requesters).
2. **Step 2 (Tie-Breaking):** Consider *only* the requesting inputs whose packet has `PRIO == max_prio`:
* If `CTRL_REG.arb_mode == 1'b0`, break ties using **Fixed Priority** (`In 0 > In 1 > In 2 > In 3`).
* If `CTRL_REG.arb_mode == 1'b1`, break ties using **Round-Robin** (`rr_ptr[j]` circular scan), and update `rr_ptr[j] <= (winner + 1) % 4`.



#### Worked Arbitration Example (Round-Robin + QoS)

Suppose Output Port 2 is `IDLE`, `rr_ptr[2] == 2'd2`, and three inputs have packets ready for Output Port 2:

* **Input 0:** `PRIO = 2'd1`
* **Input 1:** `PRIO = 2'd3`
* **Input 3:** `PRIO = 2'd3`

1. **If `qos_en == 0, arb_mode == 1` (Pure Round-Robin):**
Scan order from `rr_ptr[2] = 2` is: `In 2` (no req) $\rightarrow$ **`In 3` (req!)**. **Input 3 wins**, and `rr_ptr[2]` updates to `(3 + 1) % 4 = 0`.
2. **If `qos_en == 1, arb_mode == 1` (QoS + Round-Robin):**
Highest active priority is `max_prio = 3` (held by **Input 1** and **Input 3**; Input 0 is eliminated). Scanning circularly from `rr_ptr[2] = 2` among `{In 1, In 3}` hits **`In 3`** before `In 1`. **Input 3 wins**, and `rr_ptr[2]` updates to `0`.

---

### 5.3 Malformed Packet Detection & Flushing Rules

An incoming packet on Input Port `i` is classified as **malformed** if it violates any of the three frame integrity rules below. Malformed packets are **never forwarded** to any output port.

| Error Condition | Detection Criteria | `IRQ_STAT_REG` Bit Set | Counter Incremented |
| --- | --- | --- | --- |
| **1. Zero or Oversized `LENGTH**` | Byte 1 (`LENGTH`) is `8'd0` **or** `LENGTH > 8'd16`. | `len_err` (`[1]`) | `malformed_cnt` (`+1`) |
| **2. Early or Late `eop` Framing** | `in_eop[i]` arrives before Beat `LENGTH + 2` (premature end), **or** `in_eop[i]` is still `0` on Beat `LENGTH + 2` (packet too long). | `len_err` (`[1]`) | `malformed_cnt` (`+1`) |
| **3. Checksum (`FCS`) Mismatch** | Length is valid (`1..16`) and `in_eop[i]` arrives on Beat `LENGTH + 2`, **but** `in_data[i]` on that beat $\neq$ `HEADER ^ LENGTH ^ PAYLOAD[0..N-1]`. | `fcs_err` (`[0]`) | `malformed_cnt` (`+1`) |

#### Important Framing Edge Cases

1. **Strict Precedence (`len_err` before `fcs_err`):** If a packet has an invalid length (Condition 1 or 2), hardware must trigger **`len_err` only**—do not also evaluate or assert `fcs_err` for that packet, and increment `malformed_cnt` by **`1`** (not 2).
2. **Oversized Packet Drain Behavior:** If a sender keeps driving bytes past Beat 18 (`max packet = 19 bytes`, Beats `0..18`) without asserting `in_eop[i]`, Input Port `i` enters a `DROP_DRAIN` state: it continues asserting `in_ready[i] = 1'b1` to drain and discard incoming bytes until `in_valid[i] && in_ready[i] && in_eop[i]` is observed, at which point `len_err` and `malformed_cnt` pulse once and normal `IDLE` reception resumes on the next beat.
3. **Unexpected `in_sop` Mid-Packet:** Well-behaved senders must never assert `in_sop[i]` while a packet is already in progress. (DV SVA assertions will flag this as a protocol violation.)

---

### 5.4 Output Port Enable (`port_en`) & Packet Drop Rules

Once a packet passes ingress validation and reaches the head of Input Port `i`'s descriptor queue, its destination port(s) are checked against `CTRL_REG.port_en[3:0]`:

1. **Unicast / Transform Packets (`OPCODE == 2'b00`, `2'b10`, or `2'b11`):**
* Determine the single target output port: `target = CTRL[0] ? i : DEST_ID`.
* If `CTRL_REG.port_en[target] == 1'b0`, Input Port `i` **immediately drops the packet** from its buffer without requesting arbitration, increments `ERR_CNT_REG.drop_cnt` by `1`, and sets `IRQ_STAT_REG.drop_err = 1'b1`.


2. **Broadcast Packets (`OPCODE == 2'b01`):**
* If **at least one** output port is enabled (`CTRL_REG.port_en != 4'b0000`), the broadcast packet is **not dropped**; it is forwarded simultaneously to the subset of ports where `CTRL_REG.port_en[j] == 1'b1`.
* If **all four** output ports are disabled (`CTRL_REG.port_en == 4'b0000`), the broadcast packet is **immediately dropped**, `ERR_CNT_REG.drop_cnt` increments by `1`, and `IRQ_STAT_REG.drop_err` is set to `1'b1`.


3. **Mid-Packet Disable Protection:**
* If software disables `CTRL_REG.port_en[j]` via APB *while* Output Port `j` is already locked and mid-way through transmitting a packet, Output Port `j` **completes the current packet normally** through `out_eop[j]` before honoring the disabled state for subsequent packets.



---

### 5.5 Multi-Port Broadcast Synchronization & Flow Control

When an input port has a valid **Broadcast packet** (`OPCODE == 2'b01`) and `port_en != 4'b0000`, it must transmit the packet in **exact cycle-by-cycle lockstep** across all currently enabled output ports:

1. **Atomic All-or-Nothing Grant (Deadlock Prevention):**
* Let `bcast_mask = CTRL_REG.port_en[3:0]` be the set of enabled output ports.
* Input Port `i` can begin transmitting its broadcast packet **if and only if every port `j` in `bcast_mask` is `IDLE` (unlocked) on the exact same clock cycle** and Input `i` wins arbitration on those ports.
* *Crucial Rule:* Do **not** lock one output port while waiting for another enabled output port to finish its current packet. Locking ports partially creates circular deadlock if two input ports attempt broadcasts at the same time.


2. **Multi-Broadcast Tie-Breaking:**
* If two or more input ports simultaneously request a Broadcast on the same cycle, resolve the conflict using the arbitration settings (`qos_en` and `arb_mode`) of the **lowest-numbered enabled output port** (e.g., Output 0 if `port_en[0] == 1`), and update that port's `rr_ptr`.


3. **Lockstep Byte Transfer & Stalling:**
* Once granted, all ports in `bcast_mask` lock onto Input Port `i` simultaneously and drive identical `out_valid`, `out_sop`, `out_eop`, and `out_data` signals.
* Because Input Port `i` reads from a single FIFO, **all participating output ports must advance together**. A beat advances (and the input FIFO pops one byte) **if and only if every port `j` in `bcast_mask` has `out_ready[j] == 1'b1**`:
```systemverilog
// Byte advances only when all targeted broadcast ports are ready:
assign bcast_beat_advance = &((out_ready[3:0] | ~bcast_mask[3:0]));

```

* If even one enabled output port deasserts `out_ready[j] = 0`, all participating broadcast output ports hold their `out_valid`, `out_sop`, `out_eop`, and `out_data` steady until every targeted receiver is ready.


4. **Broadcast Packet Accounting:**
* On the final `out_eop` beat of a broadcast to $K$ enabled ports, each of the $K$ participating output ports increments its respective 16-bit counter (`port0..3_tx_cnt`) by `1`.

---

### 5.6 Quick-Reference Behavior Summary

| Scenario | Forwarded to Output? | `PKT_CNT_*` | `ERR_CNT_REG` | `IRQ_STAT_REG` |
| --- | --- | --- | --- | --- |
| Valid Unicast / Transform (`port_en[dest] == 1`) | Yes (`1` port) | `+1` on target port | No change | No change |
| Valid Broadcast (`K` ports have `port_en == 1`, $K \ge 1$) | Yes (`K` ports) | `+1` on each of `K` ports | No change | No change |
| Valid Unicast / Transform (`port_en[dest] == 0`) | **Dropped** | No change | `drop_cnt + 1` | `drop_err = 1` |
| Valid Broadcast (`port_en == 4'b0000`) | **Dropped** | No change | `drop_cnt + 1` | `drop_err = 1` |
| Invalid `LENGTH` (`0`, `>16`, or wrong `eop` beat) | **Dropped** | No change | `malformed_cnt + 1` | `len_err = 1` |
| Valid `LENGTH`, but bad `FCS` checksum byte | **Dropped** | No change | `malformed_cnt + 1` | `fcs_err = 1` |

---

## 6. Project Milestones & Execution Plan (2-Week Schedule)

### 6.1 Parallel Execution & Team Coordination Model

Because the project spans **two 1-week phases**, the **RTL Team (50%)** and **DV Team (50%)** must work **in parallel from Day 1** rather than sequentially. Both teams build against this frozen specification:

1. **Day 1 — Dummy RTL Wrapper:** The DV team compiles their UVM environment immediately against an empty `router_4x4_top` stub (matching Section 2's exact port list) so testbench development never waits on RTL readiness.
2. **End of Week 1 — Phase 1 Integration Gate:** Both teams merge their Phase 1 code. The DV team runs their UVM sanity suite on the live Phase 1 RTL to verify basic packet flow before Phase 2 begins.
3. **Week 2 — Continuous Bug-Fix & Closure Loop:** As the RTL team adds Phase 2 features, the DV team runs constrained-random and error-injection regressions, logging reproducible bug reports (with failing seeds) for the RTL team to fix before final sign-off.

---

### 6.2 Phase 1: Core Switching Engine & UVM Infrastructure (Week 1)

#### A. RTL Team Deliverables (Phase 1)

By the end of Week 1, the RTL team must deliver a synthesizable, lint-clean baseline router supporting **Unicast (`00`)** and **Broadcast (`01`)** switching under **Fixed Priority** arbitration:

* **Top-Level & Module Hierarchy:** Clean `router_4x4_top` matching Section 2 pin-for-pin, partitioned into logical sub-modules (e.g., `apb_slave`, `input_port`, `arbiter`, `crossbar`, `output_port`).
* **APB3 Register File (Baseline):**
* Full read/write support for `CTRL_REG` (`0x00`) and `KEY_REG` (`0x04`).
* Working 16-bit saturating packet counters `PKT_CNT_01` (`0x10`) and `PKT_CNT_23` (`0x14`).
* Unmapped addresses return `32'h0000_0000`; `pready` tied to `1'b1`.
* *(Phase 2 registers `IRQ_EN_REG`, `IRQ_STAT_REG`, `ERR_CNT_REG`, and `irq` may read `0` during Phase 1.)*


* **Store-and-Forward Input Ports (`in_0..3`):**
* 32-byte storage buffer + packet descriptor queue per input port.
* Proper `in_ready` backpressure assertion when buffer space runs low.
* Basic length counter so the input port knows when `eop` completes a valid packet.


* **Crossbar & Fixed-Priority Arbitration:**
* Non-preemptive packet locking (`out_sop` through `out_eop`).
* Fixed-priority arbitration (`Input 0 > Input 1 > Input 2 > Input 3`) when `arb_mode == 0`.
* Working `OPCODE == 2'b00` (**Unicast**) and `OPCODE == 2'b01` (**Broadcast** with lockstep multi-port synchronization and `out_ready` stall propagation).


* **Directed SystemVerilog Testbench:**
* A self-checking non-UVM directed testbench written by the RTL team demonstrating: (1) APB register write/readback, (2) single-packet Unicast routing across all 4 ports, (3) 2-to-1 port contention resolved by Fixed Priority, (4) Broadcast to all 4 ports, and (5) `out_ready` backpressure stalling without data loss.

#### B. DV Team (UVM) Deliverables (Phase 1)

By the end of Week 1, the DV team must build the complete UVM testbench architecture and verify it against the Phase 1 RTL:

* **Transaction Items (`uvm_sequence_item`):**
* `apb_seq_item`: Fields for `addr`, `data`, `write` (read/write direction), with constraints for valid register offsets.
* `packet_seq_item`: Fields for `dest_id`, `opcode`, `prio`, `ctrl`, `length`, `payload[]`, and `fcs`, plus helper methods `pack_bytes()`, `unpack_bytes()`, and `calc_fcs()`. Include default validity constraints (`length inside {[1:16]}`, `fcs == calc_fcs()`).

* **UVM Agents & Virtual Sequencer:**
* **1× APB Agent (Active):** `apb_driver`, `apb_monitor`, `apb_sequencer` implementing exact 2-cycle Setup/Access APB3 timing.
* **4× Input Packet Agents (Active):** One agent per input port (`in_0..3`) driving byte streams (`in_valid`, `in_data`, `in_sop`, `in_eop`) while respecting `in_ready` stalls.
* **4× Output Packet Agents (Passive + Ready Control):** Monitors capturing outgoing byte streams on `out_0..3` and reconstructing `packet_seq_item` objects, paired with configurable `out_ready` backpressure control (always-ready vs. random stall).
* **Virtual Sequencer (`router_virtual_sequencer`):** Coordinates stimulus across the APB sequencer and all four Input Packet sequencers simultaneously.

* **SystemVerilog Assertions (SVA Protocol Checker):**
* Bound to or embedded in the streaming interfaces to verify:
1. Signal stability (`valid`, `data`, `sop`, `eop` remain constant while `valid && !ready`).
2. Framing rules (`sop` and `eop` never asserted simultaneously; no `sop` mid-packet).
3. No unknown (`X`/`Z`) values on `out_valid`, `out_sop`, `out_eop`, `out_data`, `in_ready`, `prdata`, or `irq` after reset.

* **Reference Model (Predictor) & Scoreboard:**
* Predicts expected output packets per port (including Broadcast replication) and compares predicted vs. actual packets received from the Output Monitors.

* **Phase 1 Sanity Sequence Library:**
* `apb_reg_sanity_seq`, `unicast_sanity_seq`, `broadcast_sanity_seq`, and `backpressure_sanity_seq` passing with **0 UVM errors** on Phase 1 RTL.

---

### 6.3 Phase 2: Full Feature Set, Stress Testing & Coverage Closure (Week 2)

#### A. RTL Team Deliverables (Phase 2)

By the end of Week 2, the RTL team must implement all remaining specification features and resolve all bugs filed by the DV team:

* **Payload Transformations & Egress FCS Regeneration:**
* `OPCODE == 2'b10` (**XOR-Cipher** using `KEY_REG[7:0]`).
* `OPCODE == 2'b11` (**Bit-Reverse** of every payload byte).
* Dynamic egress `FCS` recomputation for transformed packets so outgoing checksums remain valid.
* `CTRL[0] == 1` (**Loopback** override routing `in_i` $\rightarrow$ `out_i`, obeying Broadcast precedence).


* **Advanced Arbitration Modes:**
* **Round-Robin** rotating pointer (`rr_ptr[0..3]`) when `CTRL_REG.arb_mode == 1`.
* **QoS Priority** filtering (`HEADER.PRIO`) when `CTRL_REG.qos_en == 1`, using `arb_mode` for tie-breaking.


* **Error Handling, Rollback & Interrupts:**
* Ingress Length validation (`LENGTH == 0`, `LENGTH > 16`, premature/late `eop`, and oversized packet draining) with pointer rollback.
* Ingress `FCS` checksum verification with strict precedence (`len_err` takes priority over `fcs_err`).
* Disabled destination port dropping (`CTRL_REG.port_en`) for both Unicast/Transform and all-disabled Broadcasts.
* Saturating `ERR_CNT_REG` (`drop_cnt` and `malformed_cnt`, handling simultaneous multi-port errors on the same cycle).
* `IRQ_EN_REG`, `IRQ_STAT_REG` (`W1C` with hardware-set priority), and top-level `irq` output.


* **Bug Resolution:** Fix and regression-test every defect logged by the DV team.

#### B. DV Team (UVM) Deliverables (Phase 2)

By the end of Week 2, the DV team must stress-test every corner case in Section 5 and achieve functional coverage closure:

* **Upgraded Reference Model & Register Checking:**
* Extend the predictor to model `XOR-Cipher`, `Bit-Reverse` + new `FCS`, `Loopback`, `port_en` gating, and expected `PKT_CNT_*` / `ERR_CNT_REG` values.
* Add register verification checks confirming `W1C` clearing, counter saturation, and `irq` pin assertion/deassertion.


* **Constrained-Random Stress & Error-Injection Sequences:**
* `hotspot_4to1_seq`: All 4 inputs continuously hammering a single output port under Fixed Priority, Round-Robin, and QoS modes.
* `back_to_back_burst_seq`: Zero-idle-cycle packet streams mixing minimum (4-byte) and maximum (19-byte) packets with randomized `out_ready` stalls.
* `error_injection_seq`: Deliberately injects `LENGTH = 0`, `LENGTH = 17..255`, truncated packets (early `eop`), oversized packets (late `eop`), corrupted `FCS` bytes, and packets targeting disabled ports (`port_en[j] = 0`).
* `irq_service_seq`: Triggers errors with interrupts enabled, verifies `irq == 1`, clears flags via `W1C` writes to `IRQ_STAT_REG`, and verifies `irq == 0`.


* **Functional Coverage (`covergroup` Closure):**
* Achieve **100% functional coverage** across the following mandatory coverpoints and crosses:
* `cp_opcode`: All 4 opcodes (`00`, `01`, `10`, `11`).
* `cp_dest_id`: All 4 destination ports (`0`, `1`, `2`, `3`).
* `cp_prio`: All 4 priority levels (`0`, `1`, `2`, `3`).
* `cp_loopback`: `CTRL[0]` (`0`, `1`).
* `cp_length`: Boundary bins (`1`, `[2:15]`, `16`).
* `cp_arb_cfg`: Cross of `CTRL_REG.arb_mode` $\times$ `CTRL_REG.qos_en`.
* `cp_errors`: `fcs_err`, `len_err_zero`, `len_err_oversize`, `len_err_early_eop`, `drop_unicast_disabled`, `drop_bcast_all_disabled`.
* `cross_ops_dest`: `cp_opcode` $\times$ `cp_dest_id`.


* **Reproducible Bug Reporting:** Every RTL failure must be logged using the standardized format below so the RTL team can reproduce it immediately:
> **[BUG-ID]** Short Title | **Test & Seed:** `<test_name> +ntb_random_seed=<seed>` | **Failure Time:** `<time_ns>` | **Port/Reg:** `<e.g., out_2>` | **Expected vs. Actual:** `<exact byte/register mismatch>`

---

### 6.4 Recommended Day-by-Day Pacing Guide

| Day | RTL Team Focus | DV Team (UVM) Focus |
| --- | --- | --- |
| **Day 1** | Study spec; partition modules; code `apb_slave` & register file. | Compile dummy wrapper; build `apb_seq_item`, `packet_seq_item`, & interfaces. |
| **Day 2** | Code `input_port` (32B buffer, write/read pointers, length counter). | Build `apb_agent` and `in_pkt_agent` (driver + monitor). |
| **Day 3** | Code `arbiter` (Fixed Priority) and `crossbar` muxing. | Build `out_pkt_agent`, `virtual_sequencer`, and SVA protocol assertions. |
| **Day 4** | Integrate `router_4x4_top` (Unicast + Broadcast); start directed TB. | Build Phase 1 Reference Model and UVM Scoreboard. |
| **Day 5** | Debug directed TB (backpressure, multi-port collisions, broadcast). | Run Phase 1 sanity sequences against dummy/early RTL; debug UVM env. |
| **Day 6–7** | **Phase 1 Integration & Freeze:** Fix sanity bugs; pass Week 1 gate. | **Phase 1 Integration & Freeze:** Run UVM sanity suite on Phase 1 RTL. |
| **Day 8** | Add `XOR-Cipher`, `Bit-Reverse`, egress `FCS` gen, and `Loopback`. | Upgrade predictor for transformations/loopback; add `covergroup`s. |
| **Day 9** | Add `Round-Robin` pointer (`rr_ptr`) and `QoS` priority logic. | Write `hotspot_4to1_seq` and `back_to_back_burst_seq`. |
| **Day 10** | Add `len_err`/`fcs_err` checks, pointer rollback, `drop_cnt`, and `irq`. | Write `error_injection_seq` and `irq_service_seq`; update predictor. |
| **Day 11–12** | Run full directed regression; fix bugs reported by DV team. | Run constrained-random regressions on Phase 2 RTL; file bug reports. |
| **Day 13** | Final bug fixes, code cleanup, and synthesis/lint check. | Write targeted sequences to close any remaining functional coverage holes. |
| **Day 14** | **Final Project Sign-Off & Demo** (clean regression + waveform walkthrough). | **Final Project Sign-Off & Demo** (0 UVM errors + 100% functional coverage). |

---
