# ⚡ MICRO4 — 4-Bit CPU on DE10-Lite FPGA

```
  ███╗   ███╗██╗ ██████╗██████╗  ██████╗ ██╗  ██╗
  ████╗ ████║██║██╔════╝██╔══██╗██╔═══██╗██║  ██║
  ██╔████╔██║██║██║     ██████╔╝██║   ██║███████║
  ██║╚██╔╝██║██║██║     ██╔══██╗██║   ██║╚════██║
  ██║ ╚═╝ ██║██║╚██████╗██║  ██║╚██████╔╝     ██║
  ╚═╝     ╚═╝╚═╝ ╚═════╝╚═╝  ╚═╝ ╚═════╝      ╚═╝
         A handcrafted 4-bit CPU in Verilog
```

> **Press a button. Count to 10. Watch the 7-segment display tick up.  
> That's a CPU you built from scratch — running on real silicon.**

---

## 📋 Table of Contents

- [What is this?](#-what-is-this)
- [Hardware Required](#-hardware-required)
- [Features](#-features)
- [Architecture](#-architecture)
- [Instruction Set](#-instruction-set)
- [The Program](#-the-program)
- [How to Run](#-how-to-run-on-fpga)
- [File Structure](#-file-structure)
- [Pin Assignments](#-pin-assignments)

---

## 🧠 What is this?

**MICRO4** is a fully functional 4-bit microprocessor written in Verilog and synthesized onto an **Intel DE10-Lite FPGA**. It is not a soft-core clone — it's a hand-designed CPU with its own instruction set, registers, ALU, and program ROM.

The demo program counts **0 → 10** on two 7-segment displays, one button press at a time, then automatically resets to 0. Simple idea. Real CPU doing real work.

---

## 🛠 Hardware Required

| Item | Details |
|------|---------|
| FPGA Board | Terasic DE10-Lite |
| FPGA Chip | Intel MAX 10 — `10M50DAF484C7G` |
| Software | Quartus Prime Lite (free) |
| Cable | USB Type-B (USB-Blaster port) |

---

## ✨ Features

- 🔲 **4-bit data path** — registers, ALU, buses
- 📝 **16 opcodes** — LOAD, MOV, ADD, SUB, AND, OR, XOR, NOT, SHL, SHR, JMP, JZ, JNZ, IN, OUT, LD
- 🏎 **8-bit Program Counter** — 256 instruction address space
- 🗃 **4 general-purpose registers** — R0, R1, R2, R3
- 🚩 **Flags** — Zero (Z) and Carry (C)
- 💾 **256 × 8-bit instruction ROM**
- 🔘 **Hardware debounce** — 20 ms settling for clean button reads
- 🖥 **Two 7-segment displays** — BCD output showing tens and units
- 💡 **4 LEDs** — mirror raw counter value in binary

---

## 🏗 Architecture

```
                    ┌─────────────────────────────────────┐
                    │           micro4_top                │
                    │                                     │
   50 MHz CLK ─────▶│  ┌──────────────────────────────┐  │
                    │  │         cpu4 core            │  │
   KEY0 (reset) ───▶│  │                              │  │
                    │  │  ┌──────┐   ┌─────────────┐  │  │
   KEY1 (count) ───▶│  │  │  PC  │──▶│  ROM 256×8  │  │  │
    (debounced)     │  │  │ 8-bit│   └──────┬──────┘  │  │
                    │  │  └──────┘          │instr     │  │
                    │  │                    ▼          │  │
                    │  │  ┌────────────────────────┐   │  │
                    │  │  │      Decoder           │   │  │
                    │  │  └────────┬───────────────┘   │  │
                    │  │          │                    │  │
                    │  │  ┌───────▼──────┐  ┌──────┐  │  │
                    │  │  │  Registers   │  │ ALU  │  │  │
                    │  │  │  R0 R1 R2 R3 │◀▶│      │  │  │
                    │  │  └──────────────┘  └──────┘  │  │
                    │  │       Flags: Z  C             │  │
                    │  └──────────────────────────────┘  │
                    │                                     │
                    │  R0 ──▶ BCD Split ──▶ HEX1 │ HEX0  │
                    │  R0 ──▶ LEDR[3:0]           │       │
                    └─────────────────────────────────────┘
```

---

## 📟 Instruction Set

All instructions are **8-bit** words: `[7:4]` = opcode, `[3:2]` = Rd, `[1:0]` = Rs

| Opcode | Mnemonic | Operation | Flags |
|--------|----------|-----------|-------|
| `0000` | `NOP` | No operation | — |
| `0001` | `LOAD Rd, imm4` | Rd ← immediate nibble | Z |
| `0010` | `MOV Rd, Rs` | Rd ← Rs | Z |
| `0011` | `ADD Rd, Rs` | Rd ← Rd + Rs | Z, C |
| `0100` | `SUB Rd, Rs` | Rd ← Rd − Rs | Z, C |
| `0101` | `AND Rd, Rs` | Rd ← Rd & Rs | Z |
| `0110` | `OR Rd, Rs` | Rd ← Rd \| Rs | Z |
| `0111` | `XOR Rd, Rs` | Rd ← Rd ^ Rs | Z |
| `1000` | `NOT Rd` | Rd ← ~Rd | Z |
| `1001` | `SHL Rd` | Rd ← Rd << 1 | Z, C |
| `1010` | `SHR Rd` | Rd ← Rd >> 1 | Z, C |
| `1011` | `JNZ addr` | PC ← addr if Z = 0 | — |
| `1100` | `LD Rd, [Rs]` | Rd ← RAM[Rs] | Z |
| `1101` | `JMP addr` | PC ← addr (unconditional) | — |
| `1110` | `JZ addr` | PC ← addr if Z = 1 | — |
| `1111 xx 00` | `OUT Rs` | LEDs ← Rs | — |
| `1111 xx 11` | `IN Rd` | Rd ← KEY1 state | Z |

> **Jump address encoding:** `target = {instr[5:0], 2'b00}` — targets must be multiples of 4.

---

## 📜 The Program

This is the assembly running inside the ROM, controlling the counter:

```asm
; ── Registers ─────────────────────────────────────────────
; R0 = counter (0–10)
; R1 = 1       (increment constant)
; R2 = 10      (limit constant)
; R3 = scratch (for comparisons and key reads)

; ── Initialise ────────────────────────────────────────────
INIT:
    LOAD  R0, 0       ; counter = 0
    LOAD  R1, 1       ; step    = 1
    LOAD  R2, 10      ; limit   = 10
    OUT   R0          ; show 0 on LEDs / display

; ── Wait for KEY1 to be pressed ───────────────────────────
WAIT_PRESS:
    IN    R3          ; R3 = KEY1 state (1 = pressed)
    JZ    WAIT_PRESS  ; if not pressed → keep polling

; ── Wait for KEY1 to be released (prevents auto-repeat) ───
WAIT_RELEASE:
    IN    R3
    JNZ   WAIT_RELEASE ; if still held → keep waiting

; ── One confirmed press: increment ────────────────────────
    ADD   R0, R1      ; R0 = R0 + 1
    OUT   R0          ; update LEDs

; ── Did we hit 10? ────────────────────────────────────────
    MOV   R3, R0      ; R3 = R0
    SUB   R3, R2      ; R3 = R0 - 10  → sets Z if R0 == 10
    JZ    INIT        ; if Z=1 → wrap back to 0
    JMP   WAIT_PRESS  ; else → wait for next press
```

---

## 🚀 How to Run on FPGA

### 1 — Install Quartus Prime Lite
Download free from: https://www.intel.com/content/www/us/en/products/details/fpga/development-tools/quartus-prime/resource.html

### 2 — Create the project
```
File → New Project Wizard
  Name:    micro4_top
  Device:  10M50DAF484C7G  (MAX 10 family)
```

### 3 — Add files & pin assignments
```
Add:    micro4_top.v
Import: micro4_top.qsf   (Assignments → Import Assignments)
```

### 4 — Compile
```
Processing → Start Compilation   (Ctrl + L)
```
✅ Look for **"Full Compilation was successful"**

### 5 — Flash to board
```
Tools → Programmer
  → Hardware Setup → select USB-Blaster
  → Add File: output_files/micro4_top.sof
  → Start
```

### 6 — Use it

| Input | Result |
|-------|--------|
| Power on / KEY0 | Display resets to `00` |
| Press KEY1 | Display increments: `01`, `02` … `10` |
| Display reaches `10` | Automatically resets to `00` |
| Press KEY0 anytime | Instant reset to `00` |

> 💡 **Want it to survive power-off?** Convert the `.sof` to a `.pof` using  
> `File → Convert Programming Files`, then re-flash with the `.pof`.

---

## 📁 File Structure

```
micro4_top/
│
├── micro4_top.v          ← All Verilog source (single file)
│   ├── micro4_top        top-level module + BCD split
│   ├── debounce          20 ms button debounce
│   ├── cpu4              CPU core (PC, registers, ALU, FSM)
│   ├── rom256x8          256×8 instruction ROM + program
│   └── seg7              7-segment decoder (active-low)
│
├── micro4_top.qsf        ← Quartus project + pin assignments
│
└── README.md             ← You are here
```

---

## 📌 Pin Assignments

| Signal | Pin | Function |
|--------|-----|---------|
| `MAX10_CLK1_50` | P11 | 50 MHz system clock |
| `KEY0` | B8 | Hard reset (active-low) |
| `KEY1` | A7 | Manual increment (active-low) |
| `LEDR[0]` | A8 | LED bit 0 |
| `LEDR[1]` | A9 | LED bit 1 |
| `LEDR[2]` | A10 | LED bit 2 |
| `LEDR[3]` | B10 | LED bit 3 |
| `HEX0[6:0]` | C14–C17 | Units digit (7-seg) |
| `HEX1[6:0]` | C18–B17 | Tens digit (7-seg) |

---

## 🧩 Extending the Project

Ideas to try next:

- **Count down** — change `ADD` to `SUB` and detect zero instead of 10
- **Adjustable limit** — read the limit from `SW[3:0]` switches via `IN`
- **Two-digit arbitrary counter** — extend ROM with a second BCD digit in R1
- **Stopwatch** — replace KEY1 polling with a timer interrupt using the 50 MHz clock
- **Custom opcodes** — add `MUL` (shift-and-add) to the ALU

---

---

**Built with Verilog · Runs on real silicon · No libraries, no shortcuts**

*Every flip-flop placed. Every gate synthesized. Every clock edge counts.*
