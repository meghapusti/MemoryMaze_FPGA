# Memory Maze: Grid Escape 🎮

> A real-time, multiplayer memory game implemented entirely in hardware on an FPGA — no CPU, no OS, just pure digital logic.

---

## Overview

**Memory Maze** is a 2-player competitive arcade game built on the **Alchitry Au FPGA** (Xilinx Artix-7) and programmed in **Lucid HDL**. Two players race on separate 4×4 **WS2812B RGB LED matrices** to reach the goal tile first, navigating from memory after the map is briefly revealed. The entire game engine — movement, collision detection, FSM control, LED driving — runs as custom hardware logic synthesized onto the FPGA fabric.

This was a group project for **50.002 Computation Structures** at SUTD.

---

## Gameplay

### Objective
Navigate your 4×4 grid to reach the **green (goal) tile** before your opponent, without stepping on any **red (restricted) tile**.

### How It Works
1. Press **Start** to load a map. The restricted and goal tiles **flash briefly for ~1 second** on both LED grids.
2. The map disappears — players must **navigate from memory**.
3. Use your 4 directional buttons to move **up, down, left, right**.
4. The first player to land on the **goal tile wins**; the other player automatically loses.
5. Press **Start** again to advance to the next map (3 maps total, cycling back to map 0).

### LED Color Encoding

| Color | Meaning |
|-------|---------|
| ⬜ White | Player position |
| 🟩 Green | Goal tile |
| 🟥 Red | Restricted (forbidden) tile |
| ⬛ Black | Empty tile |

### Win / Lose Display
- **Winner's grid** → All green
- **Loser's grid** → All red

---

## Hardware Setup

| Component | Details |
|-----------|---------|
| FPGA Board | Alchitry Au (Xilinx Artix-7, 100 MHz) |
| IO Shield | Alchitry Io (buttons, DIP switches, 7-segment display) |
| LED Displays | 2× 4×4 WS2812B addressable RGB LED matrices |
| Player 1 LED output | Pin D12 (Br connector) |
| Player 2 LED output | Pin D11 (Br connector) |
| LED power | 5V external supply |

### Button Pin Mapping

| Input | Pin | Pull |
|-------|-----|------|
| Start | C3 | pulldown |
| P1 Up | C43 | pulldown |
| P1 Down | C42 | pulldown |
| P1 Left | C8 | pulldown |
| P1 Right | C9 | pulldown |
| P2 Up | C46 | pulldown |
| P2 Down | C45 | pulldown |
| P2 Left | C5 | pulldown |
| P2 Right | C6 | pulldown |

---

## System Architecture

```
                        ┌─────────────────────────────────────┐
                        │             au_top.luc               │
                        │  (Top-level FSM + Game Coordinator)  │
                        └──────────────┬──────────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
     ┌────────▼──────┐        ┌────────▼──────┐        ┌───────▼──────┐
     │  data_ram.luc  │        │  data_ram.luc  │        │    alu.luc   │
     │   (Player 1)   │        │   (Player 2)   │        │  (Arithmetic │
     │  RAM + FSM +   │        │  RAM + FSM +   │        │   & Compare) │
     │  Win/Lose check│        │  Win/Lose check│        └──────────────┘
     └───────┬────────┘        └───────┬────────┘
             │                         │
     ┌───────▼────────┐       ┌────────▼────────┐
     │  map_rom.luc   │       │   map_rom.luc    │
     │ (3 hardcoded   │       │ (3 hardcoded     │
     │  game maps)    │       │  game maps)      │
     └────────────────┘       └─────────────────┘
             │                         │
     ┌───────▼────────┐       ┌────────▼────────┐
     │ws2812b_writer  │       │ ws2812b_writer   │
     │  (P1 LED strip │       │  (P2 LED strip   │
     │   driver)      │       │   driver)        │
     └───────┬────────┘       └────────┬─────────┘
             │                         │
     ┌───────▼────────┐       ┌────────▼─────────┐
     │  reverser.luc  │       │   reverser.luc    │
     │ (Serpentine    │       │  (Serpentine      │
     │  row fix)      │       │   row fix)        │
     └────────────────┘       └──────────────────┘
```

---

## Module Breakdown

### `au_top.luc` — Top-Level Game Controller
The main module that wires everything together. Runs a **6-state FSM**:

| State | Role |
|-------|------|
| `IDLE` | Waits for player input or start signal |
| `UPDATE_RAM_P1` | Signals P1's RAM to process movement |
| `REFRESH_P1` | Streams P1's updated grid to the LED strip |
| `UPDATE_RAM_P2` | Signals P2's RAM to process movement |
| `REFRESH_P2` | Streams P2's updated grid to the LED strip |
| `WAIT` | Reserved buffer state |

Key responsibilities:
- **Button debouncing** via `button_conditioner` and edge detection via `edge_detector`
- **Player position tracking** using D flip-flops for X/Y coordinates
- **ALU-based boundary checking** — uses ADD/SUB/CMPLT ALU operations instead of direct arithmetic to enforce grid boundaries (no stepping out of bounds)
- **Map cycling** — tracks current map with a DFF counter, wraps at map 2 → 0
- **Per-map spawn points** — each map has a designated starting position for both players
- **Win/lose display** — overrides all LED colors to solid green or red on game end

---

### `data_ram.luc` — Per-Player Game State RAM
Each player gets their own instance. Manages a 16-cell dual-port SRAM and runs an **8-state FSM**:

| State | Role |
|-------|------|
| `INIT` | Writes map data from ROM into RAM (on Start) |
| `CLEAR` | Blanks all cells to background color |
| `ERASE` | Clears old player position from RAM |
| `WRITE` | Writes new player position (white) into RAM |
| `CHECK` | Reads map ROM at new position to detect win/lose |
| `LOSE` | Holds lost state, signals `i_lost` |
| `WIN` | Holds won state, signals `i_won` |
| `IDLE` | Streams cell data to LED writer on request |

Key detail: **position-to-address mapping** is computed using the ALU — `address = y * 4 + x` is implemented as `SHL(y, 2) + x` using the shift and add ALU operations.

---

### `map_rom.luc` — Hardcoded Map Storage
Stores 3 game maps as compile-time ROM constants. Each map is a `16×2` bit array encoding one of four cell types per tile. Map is selected via a 2-bit input.

**Encoding:**
- `b11` → Start/Player (White)
- `b10` → Goal (Green)
- `b01` → Restricted (Red)
- `b00` → Empty (Black)

---

### `ws2812b_writer.luc` — WS2812B LED Protocol Driver
Bit-bangs the **WS2812B 1-wire protocol** at the hardware level. Implements the precise timing requirements:
- **Logic 1**: 800ns high, 450ns low
- **Logic 0**: 400ns high, 850ns low
- **Reset**: 50µs low

Runs a 3-state FSM (`SEND_PIXEL`, `RESET`, `CLEAR`) and streams 24-bit GRB color data for each of the 16 pixels in sequence. Exposes a `pixel_address` output so the RAM can pre-fetch the correct color one clock cycle ahead.

---

### `reverser.luc` — Serpentine Matrix Address Translator
WS2812B matrices are wired in a **serpentine (boustrophedon) pattern** — even rows go left-to-right, odd rows go right-to-left. This module transparently remaps logical pixel addresses to physical LED indices by XOR-ing the lower column bits on odd rows.

---

### `alu.luc` — Arithmetic Logic Unit
A general-purpose 16-bit ALU with 4 functional units:
- **Adder** — ADD, SUB with Z/V/N flag outputs
- **Boolean** — AND, OR, XOR, NOR (bitwise)
- **Shifter** — SHL, SHR, SRA
- **Comparator** — CMPEQ, CMPLT, CMPLE (uses Z/V/N flags)

Used throughout `au_top` and `data_ram` for all arithmetic and comparison operations instead of Lucid's native operators — demonstrating correct ALU integration from first principles.

---

## Project Structure

```
MemoryMaze_FPGA/
├── source/
│   ├── au_top.luc          # Top-level game FSM and wiring
│   ├── data_ram.luc        # Per-player RAM controller + game logic FSM
│   ├── map_rom.luc         # 3 hardcoded game maps (ROM)
│   ├── ws2812b_writer.luc  # WS2812B LED protocol driver
│   ├── reverser.luc        # Serpentine matrix address translator
│   ├── alu.luc             # 16-bit ALU (add/bool/shift/compare)
│   ├── adder.luc           # Ripple-carry adder with Z/V/N flags
│   ├── boolean.luc         # Bitwise boolean unit
│   ├── shifter.luc         # Barrel shifter
│   └── compare.luc         # Comparator (uses ALU flags)
├── constraint/
│   └── custom.acf          # Pin constraint file (FPGA I/O mapping)
├── datasheet/
│   └── WS2812B.pdf         # WS2812B LED datasheet
├── WS2812B.alp             # Alchitry Labs project file
└── README.md
```

---

## Building & Flashing

1. Open **Alchitry Labs** (v1 or v2).
2. Open `WS2812B.alp` as the project file.
3. Click **Build** — this runs Vivado synthesis and place-and-route in the background.
4. Once complete, click **Program** to flash the bitstream to the Alchitry Au board.

> Ensure the WS2812B LED matrices are powered from a **5V external supply**, not the FPGA's onboard 3.3V rail.

---

## Technical Highlights

- **Pure hardware implementation** — zero software, zero processor. All game logic is synthesized into FPGA LUTs and flip-flops.
- **Dual independent game instances** — two full `data_ram` + `ws2812b_writer` pipelines run concurrently for simultaneous per-player LED updates.
- **Hardware ALU reuse** — boundary checks, coordinate arithmetic, and address computation all route through the custom ALU, demonstrating hardware resource discipline.
- **Metastability-safe inputs** — all external button inputs pass through `button_conditioner` (synchronizer + debouncer) and `edge_detector` before use.
- **Timing-accurate LED protocol** — the WS2812B driver bit-bangs nanosecond-level pulse widths correctly against a 100 MHz clock (10 ns resolution).
- **Serpentine addressing** — physical LED wiring mismatch handled transparently in hardware via XOR-based index reversal.

---

## Demo

![Game Board](https://github.com/50002-computation-structures/1d-project-group_34/assets/117970574/4d5e61b3-d2ff-4659-baa8-39e15827bf66)

![Project Poster](https://github.com/meghapusti/MemoryMaze_FPGA/assets/134662739/3db6f942-6bbf-4390-964b-3e06c3f9b4c5)

---

## Course Context

Built for **50.002 Computation Structures** at the **Singapore University of Technology and Design (SUTD)**. The course covers digital logic design, FSMs, datapath construction, and processor architecture — this project is the culminating 1D (1-dimensional) hardware design assignment.

---

## License

This project is for educational purposes. Feel free to use it as reference for FPGA game development or WS2812B LED matrix driving.
