# Smart Parking System — FPGA Verilog (Quartus)

A hardware description of a fully digital smart parking management system implemented in Verilog, designed for synthesis and simulation in **Intel Quartus Prime**. The system tracks up to 3 cars, records entry/exit timestamps, calculates parking cost, and displays results on seven-segment displays — all driven by an FSM-based controller.

A demo video is included (`demo.mp4`).

---

## Features

- Supports up to **3 simultaneous parking slots** (2-bit car IDs: 0, 1, 2)
- Tracks **entry and exit timestamps** per car using a free-running hardware timer
- Calculates **parking cost** based on time spent; cost is capped at **99 units**
- Displays **car count**, **cost tens digit**, and **cost units digit** on seven-segment displays
- Indicates **full** and **empty** parking status via dedicated output signals
- **Edge detection** on entry/exit buttons to prevent repeated triggering
- **FSM controller** with a reset-delay lock to ensure clean initialisation
- Configurable clock divider with a **simulation mode** for fast testbench runs

---

## Module Overview

| Module | Description |
|--------|-------------|
| `parking_project` | Top-level module — wires all submodules together |
| `fsm_controller` | 4-state FSM: `IDLE → ENTER → EXIT → DISPLAY` |
| `timestamp_buffer` | Stores entry timestamps and tracks per-car presence |
| `cost_calculator` | Computes cost combinatorially; caps result at 99 |
| `cost_buffer` | Holds calculated cost per car for display |
| `counter` | Up/down counter (0–3); generates `full` and `empty` flags |
| `clock_divider` | Divides system clock to 1 Hz ticks; simulation mode included |
| `edge_detector` | Rising-edge detector for button debouncing |
| `bcd_converter` | Converts binary cost to BCD tens and units digits |
| `seven_seg` | Drives a common-anode 7-segment display for digits 0–9 |

---

## FSM States

```
         entry_btn pressed           exit_btn pressed
   ┌──────────────────────┐   ┌──────────────────────┐
   │                      ▼   │                      ▼
IDLE ──────────────────► ENTER      IDLE ──────────────────► EXIT
  ▲                       │           ▲                       │
  └───────────────────────┘           │                       ▼
        (back to IDLE)                └──────────── DISPLAY ──┘
                                         (show_cost, write_cost)
```

| State | Actions |
|-------|---------|
| `IDLE` | Wait for `entry_btn` (no car present) or `exit_btn` (car present) |
| `ENTER` | Write current timestamp; increment car counter |
| `EXIT` | Read stored entry timestamp; decrement car counter |
| `DISPLAY` | Calculate and latch cost; drive seven-segment displays |

A 2-cycle reset delay (`reset_delay`) keeps the FSM locked in `IDLE` immediately after reset.

---

## I/O Description (Top-Level)

| Port | Direction | Width | Description |
|------|-----------|-------|-------------|
| `clk` | Input | 1 | System clock (50 MHz typical for DE-series boards) |
| `rst` | Input | 1 | Asynchronous active-high reset |
| `entry_btn` | Input | 1 | Car entry button |
| `exit_btn` | Input | 1 | Car exit button |
| `car_id` | Input | 2 | Selects which parking slot (0–2) |
| `ccount_display` | Output | 7 | Seven-segment: current car count |
| `cost_display_units` | Output | 7 | Seven-segment: cost units digit |
| `cost_display_tens` | Output | 7 | Seven-segment: cost tens digit |
| `full` | Output | 1 | High when all 3 slots are occupied |
| `empty` | Output | 1 | High when no slots are occupied |

---

## Cost Calculation

```
cost = ((exit_time - entry_time) >> 2) × cost_per_unit
```

- Time is measured in hardware ticks (1 Hz after clock division).
- The right-shift by 2 divides elapsed ticks by 4, giving a coarser time unit.
- `cost_per_unit` is hardcoded to `1` in the top-level instantiation.
- Cost is **capped at 99** to fit within two BCD digits on the seven-segment display.
- If `entry_time == 0` or `exit_time ≤ entry_time`, cost is 0.

---

## Clock Divider

The `clock_divider` module has two modes selected by the `SIMULATION` parameter:

| Mode | `SIMULATION` | Divide-by | Tick rate (50 MHz input) |
|------|-------------|-----------|--------------------------|
| Hardware | `0` | `25,000,000` | ~2 Hz |
| Simulation | `1` | `1,000,000` | ~50 Hz |

The top-level instantiates with `SIMULATION(0)` for real hardware. Set `SIMULATION(1)` in a testbench for fast simulation.

---

## Setup (Quartus Prime)

### 1. Create a new project

- Open Quartus Prime → **New Project Wizard**
- Set top-level entity to `parking_project`
- Select your target FPGA (e.g., Cyclone IV / DE2-115 or Cyclone V / DE0-CV)

### 2. Add source files

- Add `Quartaz_code.txt` (rename to `parking_project.v` or split into individual `.v` files — one per module)

### 3. Assign pins

Map the top-level ports to your board's physical pins via **Assignments → Pin Planner**:

| Signal | Typical DE2-115 pin |
|--------|-------------------|
| `clk` | PIN_Y2 (50 MHz oscillator) |
| `rst` | KEY[0] |
| `entry_btn` | KEY[1] |
| `exit_btn` | KEY[2] |
| `car_id[1:0]` | SW[1], SW[0] |
| `ccount_display[6:0]` | HEX0 |
| `cost_display_units[6:0]` | HEX1 |
| `cost_display_tens[6:0]` | HEX2 |
| `full` | LEDR[0] |
| `empty` | LEDR[1] |

### 4. Compile and program

```
Processing → Start Compilation
Tools → Programmer → Start
```

---

## Simulation

To simulate in ModelSim (Quartus-integrated):

1. Set `SIMULATION = 1` in `clock_divider` instantiation to speed up tick generation.
2. Create a testbench that drives `clk`, `rst`, `entry_btn`, `exit_btn`, and `car_id`.
3. Assert `rst` for at least 3 clock cycles, then release before driving button inputs.
4. Use **Tools → Run Simulation Tool → RTL Simulation** in Quartus.

---

## File Reference

| File | Contents |
|------|----------|
| `Quartaz_code.txt` | All Verilog modules in a single file |
| `demo.mp4` | Hardware demonstration video |
| `.gitattributes` | Git LFS tracking for binary files (`.bin`, `.zip`) |
