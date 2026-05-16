# UART (Universal Asynchronous Receiver-Transmitter) — SystemVerilog Implementation

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Module List](#module-list)
4. [Register Map](#register-map)
5. [Module: `uart_tx_top` — Transmitter](#module-uart_tx_top--transmitter)
6. [Module: `uart_rx_top` — Receiver](#module-uart_rx_top--receiver)
7. [Module: `regs_uart` — Register File](#module-regs_uart--register-file)
8. [Register Definitions](#register-definitions)
   - [FCR — FIFO Control Register](#fcr--fifo-control-register)
   - [LCR — Line Control Register](#lcr--line-control-register)
   - [LSR — Line Status Register](#lsr--line-status-register)
   - [DIV — Baud Rate Divisor](#div--baud-rate-divisor)
   - [THR / RHR — Transmit / Receive Holding Registers](#thr--rhr--transmit--receive-holding-registers)
   - [SCR — Scratch Pad Register](#scr--scratch-pad-register)
9. [Baud Rate Clock Generation](#baud-rate-clock-generation)
10. [Signal Glossary](#signal-glossary)
11. [State Machines](#state-machines)
    - [TX State Machine](#tx-state-machine)
    - [RX State Machine](#rx-state-machine)
12. [Parity Logic](#parity-logic)
13. [Known Issues / Notes](#known-issues--notes)

---

## Overview

This repository contains a **16550-compatible UART** implementation written in **SystemVerilog**. It supports:

- Configurable word lengths: 5, 6, 7, or 8 data bits
- Optional parity (odd, even, mark, or space)
- 1 or 2 stop bits
- FIFO-based TX and RX paths
- Break signal generation and detection
- Framing error, parity error, and overrun error detection
- Baud rate generation via a programmable divisor latch

The design is split into three major modules: a **transmitter (`uart_tx_top`)**, a **receiver (`uart_rx_top`)**, and a **register file (`regs_uart`)** that exposes the configuration and status registers to a host bus interface.

---

## Architecture Diagram
<img width="2816" height="1536" alt="uart" src="https://github.com/user-attachments/assets/a2acab71-5dc9-4ed1-8f1c-5a90ab01e43a" />


---

## Module List

| Module         | Purpose                                               |
|----------------|-------------------------------------------------------|
| `uart_tx_top`  | Serial transmitter — shift register, parity, stop bits|
| `uart_rx_top`  | Serial receiver — start detection, sampling, framing  |
| `regs_uart`    | APB-style register file — LCR, LSR, FCR, DIV, etc.   |

Struct types used across modules:

| Type     | Fields                              |
|----------|-------------------------------------|
| `fcr_t`  | FIFO Control Register fields        |
| `lcr_t`  | Line Control Register fields        |
| `lsr_t`  | Line Status Register fields         |
| `csr_t`  | Top-level container for all CSRs    |
| `div_t`  | 16-bit Divisor Latch (MSB + LSB)    |

---

## Register Map

| Addr | DLAB | Access | Register                         |
|------|------|--------|----------------------------------|
| 0x0  | 0    | W      | THR — Transmit Holding Register  |
| 0x0  | 0    | R      | RHR — Receive Holding Register   |
| 0x0  | 1    | R/W    | DLL — Divisor Latch LSB          |
| 0x1  | 1    | R/W    | DLM — Divisor Latch MSB          |
| 0x1  | 0    | R/W    | IER — Interrupt Enable Register  |
| 0x2  | —    | R      | IIR — Interrupt Identification   |
| 0x2  | —    | W      | FCR — FIFO Control Register      |
| 0x3  | —    | R/W    | LCR — Line Control Register      |
| 0x4  | —    | R/W    | MCR — Modem Control Register     |
| 0x5  | —    | R      | LSR — Line Status Register       |
| 0x6  | —    | R      | MSR — Modem Status Register      |
| 0x7  | —    | R/W    | SCR — Scratch Pad Register       |

> The **DLAB** bit (bit 7 of LCR) selects whether address 0x0/0x1 accesses the data FIFO or the baud rate divisor registers.

---

## Module: `uart_tx_top` — Transmitter

### Port List

| Port           | Dir    | Width | Description                                     |
|----------------|--------|-------|-------------------------------------------------|
| `clk`          | Input  | 1     | System clock                                    |
| `rst`          | Input  | 1     | Asynchronous reset (active high)                |
| `baud_pulse`   | Input  | 1     | One-cycle pulse at 16× baud rate                |
| `pen`          | Input  | 1     | Parity enable (from LCR)                        |
| `thre`         | Input  | 1     | TX Holding Register Empty flag (from LSR)       |
| `stb`          | Input  | 1     | Stop bit select: 0=1 stop bit, 1=2 stop bits    |
| `sticky_parity`| Input  | 1     | Sticky parity enable (from LCR)                 |
| `eps`          | Input  | 1     | Even parity select (from LCR)                   |
| `set_break`    | Input  | 1     | Forces TX line low (break condition)            |
| `din[7:0]`     | Input  | 8     | Parallel data from TX FIFO                      |
| `wls[1:0]`     | Input  | 2     | Word length select: 00=5b, 01=6b, 10=7b, 11=8b  |
| `pop`          | Output | 1     | Read strobe to TX FIFO                          |
| `sreg_empty`   | Output | 1     | Shift register empty flag                       |
| `tx`           | Output | 1     | Serial transmit line                            |

### Internal Registers

| Signal       | Width | Description                                       |
|--------------|-------|---------------------------------------------------|
| `shft_reg`   | 8     | Parallel-to-serial shift register                 |
| `tx_data`    | 1     | Current bit being driven to `tx`                  |
| `d_parity`   | 1     | Raw computed data parity (XOR of data bits)        |
| `bitcnt`     | 3     | Counts remaining data bits to send                |
| `count`      | 5     | 16× oversampling down-counter (0–15)              |
| `parity_out` | 1     | Final parity bit after applying parity mode       |

### Operation Summary

The TX engine operates on `baud_pulse` ticks. Each bit period is **16 baud pulses** wide (count 15 → 0). The state machine loads data from the TX FIFO, serialises it LSB-first, optionally appends a parity bit, and then drives a stop condition (line HIGH) before returning to idle.

---

## Module: `uart_rx_top` — Receiver

### Port List

| Port           | Dir    | Width | Description                                     |
|----------------|--------|-------|-------------------------------------------------|
| `clk`          | Input  | 1     | System clock                                    |
| `rst`          | Input  | 1     | Asynchronous reset (active high)                |
| `baud_pulse`   | Input  | 1     | One-cycle pulse at 16× baud rate                |
| `rx`           | Input  | 1     | Serial receive line                             |
| `sticky_parity`| Input  | 1     | Sticky parity mode enable                       |
| `eps`          | Input  | 1     | Even parity select                              |
| `pen`          | Input  | 1     | Parity enable                                   |
| `wls[1:0]`     | Input  | 2     | Word length select                              |
| `push`         | Output | 1     | Write strobe to RX FIFO                         |
| `pe`           | Output | 1     | Parity error flag                               |
| `fe`           | Output | 1     | Framing error flag                              |
| `bi`           | Output | 1     | Break interrupt flag                            |
| `dout[7:0]`    | Output | 8     | Received parallel data byte                     |

### Internal Registers

| Signal    | Width | Description                                          |
|-----------|-------|------------------------------------------------------|
| `rx_reg`  | 1     | One-cycle delayed copy of `rx` for edge detection    |
| `bitcnt`  | 3     | Counts remaining data bits to receive                |
| `count`   | 4     | Oversampling down-counter                            |
| `pe_reg`  | 1     | Intermediate parity error latch                      |

### Falling Edge Detection

```systemverilog
assign fall_edge = rx_reg;   // NOTE: See Known Issues section
```

The receiver monitors for a falling edge on the `rx` line to detect the beginning of a start bit. `rx_reg` is the registered (delayed by one clock) version of `rx`. The intent is `fall_edge = rx_reg & ~rx`, but the current implementation only uses `rx_reg` — see [Known Issues](#known-issues--notes).

### Sampling Strategy

The RX state machine samples each incoming bit at **count == 7** (the midpoint of the 16-clock bit window), which maximises timing margin against jitter on the `rx` line.

---

## Module: `regs_uart` — Register File

### Port List

| Port                   | Dir    | Width | Description                                   |
|------------------------|--------|-------|-----------------------------------------------|
| `clk`                  | Input  | 1     | System clock                                  |
| `rst`                  | Input  | 1     | Asynchronous reset                            |
| `wr_i`                 | Input  | 1     | Write enable from host bus                    |
| `rd_i`                 | Input  | 1     | Read enable from host bus                     |
| `rx_fifo_empty_i`      | Input  | 1     | RX FIFO empty flag                            |
| `rx_oe / rx_pe / rx_fe / rx_bi` | Input | 1 each | Error flags from RX engine              |
| `addr_i[2:0]`          | Input  | 3     | Register select address                       |
| `din_i[7:0]`           | Input  | 8     | Write data from host                          |
| `tx_push_o`            | Output | 1     | Push strobe to TX FIFO                        |
| `rx_pop_o`             | Output | 1     | Pop strobe from RX FIFO                       |
| `baud_out`             | Output | 1     | Baud pulse to TX and RX engines               |
| `tx_rst / rx_rst`      | Output | 1     | FIFO reset signals (from FCR)                 |
| `rx_fifo_threshold[3:0]`| Output| 4     | RX FIFO trigger level (from FCR)              |
| `dout_o[7:0]`          | Output | 8     | Read data to host bus                         |
| `csr_o`                | Output | csr_t | Full CSR struct exposed to TX/RX engines      |
| `rx_fifo_in[7:0]`      | Input  | 8     | Data from RX FIFO for host reads              |

### THR Access Logic

```systemverilog
// Write to TX FIFO when: write cycle, address 0, DLAB = 0
assign tx_fifo_wr = wr_i & (addr_i == 3'b000) & (csr.lcr.dlab == 1'b0);
assign tx_push_o  = tx_fifo_wr;

// Read from RX FIFO when: read cycle, address 0, DLAB = 0
assign rx_fifo_rd = rd_i & (addr_i == 3'b000) & (csr.lcr.dlab == 1'b0);
assign rx_pop_o   = rx_fifo_rd;
```

---

## Register Definitions

### FCR — FIFO Control Register

**Address:** 0x2 (Write only)

| Bits  | Field         | Description                                              |
|-------|---------------|----------------------------------------------------------|
| [7:6] | `rx_trigger`  | RX FIFO interrupt trigger level: 00=1B, 01=4B, 10=8B, 11=14B |
| [5:4] | `reserved`    | Reserved, write 0                                        |
| [3]   | `dma_mode`    | DMA mode select                                          |
| [2]   | `tx_rst`      | Transmit FIFO reset (self-clearing)                      |
| [1]   | `rx_rst`      | Receive FIFO reset (self-clearing)                       |
| [0]   | `ena`         | FIFO enable (must be set for 16550 FIFO operation)       |

### LCR — Line Control Register

**Address:** 0x3 (Read/Write)

| Bits | Field          | Description                                                 |
|------|----------------|-------------------------------------------------------------|
| [7]  | `dlab`         | Divisor Latch Access Bit — 1 = access DLL/DLM, 0 = data    |
| [6]  | `set_break`    | Force TX line low (break signal)                            |
| [5]  | `stick_parity` | Sticky parity: forces parity bit to fixed value             |
| [4]  | `eps`          | Even Parity Select: 0 = odd, 1 = even                       |
| [3]  | `pen`          | Parity Enable: 1 = parity bit transmitted and checked        |
| [2]  | `stb`          | Stop bits: 0 = 1 stop bit, 1 = 2 stop bits (1.5 for 5-bit) |
| [1:0]| `wls`          | Word Length: 00=5b, 01=6b, 10=7b, 11=8b                    |

**Word Length (`wls`) Encoding:**

| wls  | Data Bits | `bitcnt` initial value |
|------|-----------|------------------------|
| 2'b00 | 5        | 3'b100 (4 remaining after first bit) |
| 2'b01 | 6        | 3'b101 |
| 2'b10 | 7        | 3'b110 |
| 2'b11 | 8        | 3'b111 |

The TX module uses `{1'b1, wls}` as the starting `bitcnt`, meaning the first bit is sent immediately on the `start→send` transition and `bitcnt` counts down the remainder.

### LSR — Line Status Register

**Address:** 0x5 (Read only)

| Bit | Field           | Description                                               |
|-----|-----------------|-----------------------------------------------------------|
| [7] | `rx_fifo_error` | At least one error (PE/FE/BI) in RX FIFO                  |
| [6] | `temt`          | Transmitter Empty: both THR and shift register are empty  |
| [5] | `thre`          | Transmitter Holding Register Empty                        |
| [4] | `bi`            | Break Interrupt: RX line held low for > 1 full frame      |
| [3] | `fe`            | Framing Error: stop bit was 0 instead of 1                |
| [2] | `pe`            | Parity Error: received parity does not match expected     |
| [1] | `oe`            | Overrun Error: new data arrived before old was read       |
| [0] | `dr`            | Data Ready: at least one byte waiting in RX FIFO          |

### DIV — Baud Rate Divisor

**Accessed when DLAB = 1:**

| Address | Register | Description                   |
|---------|----------|-------------------------------|
| 0x0     | DLL      | Divisor Latch LSB             |
| 0x1     | DLM      | Divisor Latch MSB             |

The 16-bit divisor `{DLM, DLL}` sets the baud rate:

```
Baud Rate = f_clk / (16 × Divisor)
```

**Example:** For a 50 MHz clock and 115200 baud:
```
Divisor = 50_000_000 / (16 × 115200) ≈ 27
```

The baud clock generator produces a one-clock-wide `baud_pulse` every `Divisor` clocks, which drives both the TX and RX state machines.

### THR / RHR — Transmit / Receive Holding Registers

These are **not physical registers** in the traditional sense — they are the interfaces to the TX and RX FIFOs.

- **THR (Transmit Holding Register):** A write to address 0x0 with `DLAB=0` pushes the written byte onto the TX FIFO.
- **RHR (Receive Holding Register):** A read from address 0x0 with `DLAB=0` pops the oldest byte from the RX FIFO.

In the original 8250 UART, these were single-byte buffers. The 16550 upgrade replaced them with 16-byte FIFOs.

### SCR — Scratch Pad Register

**Address:** 0x7 (Read/Write)

An 8-bit general-purpose register with no effect on UART operation. Used by software to test bus connectivity or store temporary values.

---

## Baud Rate Clock Generation

Although the baud clock generator is not fully shown in this code, the `regs_uart` module outputs `baud_out` — a single-cycle pulse that drives both `uart_tx_top` and `uart_rx_top`.

The generator counts system clock cycles up to the 16-bit divisor value loaded into `{DLM, DLL}`. Each time the counter reaches the divisor value it emits one `baud_pulse` and resets.

Because the TX and RX state machines count **16 baud pulses per bit period**, the effective oversampling ratio is 16×, which centres the RX sample point in the middle of each incoming bit.

```
System Clock:  _____|‾|_|‾|_|‾| ... (many cycles)
baud_pulse:    ___________________|‾|___________________  (every Divisor clocks)
Bit period:    16 baud_pulses = one UART bit
RX sample:     at baud_pulse #8 of each bit window
```

---

## Signal Glossary

| Signal         | Module(s)     | Description                                              |
|----------------|---------------|----------------------------------------------------------|
| `baud_pulse`   | ALL           | 16× oversampled baud tick                                |
| `wls[1:0]`     | TX, RX, LCR   | Word length select                                       |
| `pen`          | TX, RX, LCR   | Parity enable                                            |
| `eps`          | TX, RX, LCR   | Even parity select                                       |
| `sticky_parity`| TX, RX, LCR   | Force parity bit to mark (1) or space (0)                |
| `stb`          | TX, LCR       | 2 stop bit select                                        |
| `set_break`    | TX, LCR       | Force TX low (break condition)                           |
| `thre`         | TX, LSR       | TX FIFO empty — allows TX to load next byte              |
| `pop`          | TX → FIFO     | Read next byte from TX FIFO                              |
| `push`         | RX → FIFO     | Write received byte to RX FIFO                           |
| `sreg_empty`   | TX → LSR      | TX shift register drained — feeds `temt` in LSR          |
| `fall_edge`    | RX (internal) | Start bit detection (falling edge on RX line)            |
| `pe_reg`       | RX (internal) | Computed parity error before latching to `pe` output     |
| `d_parity`     | TX (internal) | XOR of all data bits before parity mode is applied       |
| `dlab`         | LCR, regs     | Divisor Latch Access Bit                                 |

---

## State Machines

### TX State Machine

```
           ┌──────────────────────────────────┐
           │  thre=1 (FIFO empty)             │
           ▼                                  │
  ┌──────────────┐   count==0 & thre=0    ┌───┴──────────┐
  │    IDLE      ├───────────────────────►│    START     │
  │  tx=HIGH     │                        │  tx=0(start) │
  └──────────────┘                        └──────┬───────┘
           ▲                                     │ count==0
           │  (stop bits done)                   ▼
           │                             ┌───────────────┐
           │◄────────────────────────────┤     SEND      │
           │  pen=0                      │  tx=data bits │
           │                             └───────┬───────┘
           │                                     │ bitcnt==0 & pen=1
           │                             ┌───────▼───────┐
           │◄────────────────────────────┤    PARITY     │
             (stop bits done)            │  tx=parity bit│
                                         └───────────────┘
```

**Stop bit duration** is controlled by `stb` and `wls`:

| `stb` | `wls`   | Stop Bits | count target |
|-------|---------|-----------|--------------|
| 0     | any     | 1         | 15 (1 bit)   |
| 1     | 00 (5b) | 1.5       | 23           |
| 1     | other   | 2         | 31           |

### RX State Machine

```
  ┌──────────┐   fall_edge=0    ┌──────────┐
  │   IDLE   ├────────────────►│  START   │ (wait for midpoint, validate)
  └──────────┘                  └────┬─────┘
                                     │ count==0
                                     ▼
                               ┌──────────┐
                               │   READ   │ (sample bits at count==7)
                               └────┬─────┘
                          ┌─────────┴──────────┐
                       pen=0                pen=1
                          │                    │
                          ▼                    ▼
                    ┌──────────┐         ┌──────────┐
                    │   STOP   │         │  PARITY  │ (check pe at count==7)
                    └────┬─────┘         └────┬─────┘
                         │                    │
                         │◄───────────────────┘
                         │
                    push=1, check fe
                         │
                         ▼
                    back to IDLE
```

---

## Parity Logic

### TX Parity Computation

The raw parity (`d_parity`) is computed at the `start→send` transition by XOR-ing the data bits according to word length:

```systemverilog
case(wls)
  2'b00: d_parity <= ^shft_reg[4:0]; // 5-bit word
  2'b01: d_parity <= ^shft_reg[5:0]; // 6-bit word
  2'b10: d_parity <= ^shft_reg[6:0]; // 7-bit word
  2'b11: d_parity <= ^shft_reg[7:0]; // 8-bit word
endcase
```

The final parity bit is derived in the `send` state:

```systemverilog
case({sticky_parity, eps})
  2'b00: parity_out <= ~d_parity; // Odd parity
  2'b01: parity_out <=  d_parity; // Even parity
  2'b10: parity_out <= 1'b1;      // Mark parity (always 1)
  2'b11: parity_out <= 1'b0;      // Space parity (always 0)
endcase
```

### RX Parity Check

The receiver checks incoming parity in the `read` state when `bitcnt==0` (all data bits received):

```systemverilog
case({sticky_parity, eps})
  2'b00: pe_reg <= ~^{rx, dout}; // Odd: error if XOR result is even
  2'b01: pe_reg <=  ^{rx, dout}; // Even: error if XOR result is odd
  2'b10: pe_reg <= ~rx;           // Mark: error if parity bit is 0
  2'b11: pe_reg <=  rx;           // Space: error if parity bit is 1 (see note)
endcase
```

The `pe` output is latched in the `parity` state (at the midpoint sample, `count==7`) and propagated to the LSR.

---

## Known Issues / Notes

### 1. Falling Edge Detection Bug (RX)

```systemverilog
// Current (incorrect):
assign fall_edge = rx_reg;

// Correct implementation should be:
assign fall_edge = rx_reg & ~rx;  // True only on a 1→0 transition
```

As written, `fall_edge` is simply equal to the delayed `rx` signal. The `idle` state checks `!fall_edge` which therefore triggers whenever `rx_reg` is low, not only on a fresh falling edge. This could cause false start-bit detection during an ongoing low period.

### 2. RX `pe_reg` Case Overlap

In the `read` state parity case statement, two cases share the same value `2'b00`:

```systemverilog
// BUG: Two entries for 2'b00
2'b00: pe_reg <= ~^{rx,dout}; // meant as odd parity
2'b00: pe_reg <= rx;           // should be 2'b11 for space parity
```

The second entry should be `2'b11`. The space parity check will never execute, and only the first entry will be used by synthesis/simulation.

### 3. Stop Bit Timing in Parity State (TX)

After the parity bit, stop bit count targets are:

```systemverilog
count <= (stb == 1'b0) ? 5'd17 : 5'd31;
```

The value `5'd17` represents approximately 1 stop bit (differs from the `5'd15` used in the no-parity path). Verify these cycle counts match your timing requirements.

### 4. RX `count` Register Width

`count` is declared as `reg [3:0]` (4 bits, max 15) but is loaded with `5'd15` (5-bit literal). The 5-bit literal fits within 4 bits since 15 = 4'b1111, but the intent may be to allow wider counts. Compare with TX where `count` is correctly declared as `reg [4:0]`.

### 5. `bi` (Break Interrupt) Not Implemented in RX

The `bi` output port is declared and reset to 0 but never assigned in the RX state machine body. Break detection logic (RX line held low across an entire frame) needs to be added.

---

