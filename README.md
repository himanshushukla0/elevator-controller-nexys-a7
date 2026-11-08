# 🛗 8-Floor Smart Elevator Controller

An FPGA implementation of an 8-floor elevator controller designed in Verilog HDL for the **Digilent Nexys A7-100T** (Xilinx Artix-7 `XC7A100T-1CSG324C`).

The controller implements the **LOOK / SCAN scheduling algorithm** (elevator algorithm), a 6-state finite state machine (FSM), synchronous single-clock-enable domain crossing, input conditioning (synchronization and 10 ms debouncing), safety interlocks, dynamic 8-digit 7-segment display multiplexing, and PWM-dimmed RGB cabin status indicators.

---

## 📌 Features

- **8-Floor Servicing (Floors 0 to 7)**: Floor calls latched via onboard slide switches (`SW[7:0]`).
- **LOOK / SCAN Scheduling**: Prioritizes calls along the direction of travel, servicing intermediate floors sequentially before reversing direction.
- **Robust 6-State FSM**: 
  - `STATE_IDLE`
  - `STATE_MOVE_UP`
  - `STATE_MOVE_DOWN`
  - `STATE_DOOR_OPEN`
  - `STATE_DOOR_CLOSING`
  - `STATE_EMERGENCY`
- **Safety Interlocks & Overrides**:
  - **Emergency Stop (`BTNC`)**: Immediate freeze from any state; toggle-latched to resume on second press.
  - **Door Hold Open (`BTNU`)**: Continuous reload of the door timer while held.
  - **Force Close (`BTND`)**: Overrides door open countdown to close doors immediately.
  - **Obstruction Sensor (`BTNL`)**: Automatically reopens closing doors.
  - **Overweight Sensor (`BTNR`)**: Blocks elevator departure and freezes doors in open state.
  - **Fire Service Recall (`SW[11]`)**: Overrides all calls and recalls cabin immediately to Ground Floor (Floor 0).
- **Comprehensive Visual Feedback**:
  - **8-Digit 7-Segment Display**:
    - `AN0`: Current Floor (`0` - `7`)
    - `AN1`: Direction (`U` for UP, `d` for DOWN, blank when IDLE)
    - `AN2`: Next Target Stop (`0` - `7`)
    - `AN3`: Separator (Blank)
    - `AN7:AN4`: Real-time status text (`OPEn`, `CLOS`, `FULL`, `StOP`)
  - **LEDs (`LED[15:0]`)**:
    - `LED[7:0]`: Real-time pending floor call indicator
    - `LED[12:8]`: 5-level door countdown timer bar graph
    - `LED[13]`: Door Open indicator
    - `LED[14]`: Moving DOWN (6 Hz blinking indicator)
    - `LED[15]`: Moving UP (6 Hz blinking indicator)
  - **RGB Status Indicators (`LD16`, `LD17`)**:
    - Green: Idle
    - Blue: Moving (UP / DOWN)
    - Yellow: Door Open / Closing
    - Red: Emergency Stop
    - *Hardware-friendly 1/16 PWM dimming (6.25% duty cycle) to prevent eye fatigue.*
- **Reliable Hardware Design**:
  - Single 100 MHz clock domain with synchronous 1-cycle clock enables.
  - Jitter-free restart on movement/door entry.
  - Asynchronous assert, synchronous release 2-FF reset synchronizer on `CPU_RESETN`.
  - Zero compile warnings under `` `default_nettype none ``.

---

## 🏗️ Architecture & Module Hierarchy

```
elevator_top.v
├── clk_en_gen        Synchronous 100 MHz prescaler (en_fsm, en_blink, en_ssd, en_1ms, en_1s)
├── input_cond        2-FF synchronizers, 10 ms debouncers, 1-cycle edge-detect pulses
├── request_latch     Pending request register with Fire Recall & clear-on-arrival
├── elevator_fsm      6-State LOOK/SCAN motion engine & safety controller
├── ssd_mux           8-Digit 760 Hz multiplexed seven-segment display driver
└── rgb_status        PWM-dimmed dual RGB LED cabin status controller
```

---

## 🎮 Nexys A7 Hardware Mapping

### Switches (`SW[15:0]`)
| Switch | Pin | Function |
| :--- | :--- | :--- |
| `SW[7:0]` | `R13, U18, T18, R17, R15, M13, L16, J15` | Floor Request Calls (Floors 7 down to 0) |
| `SW[8]` | `T8` | Speed Select (`0`: Realistic 2.0s/floor, `1`: Fast Simulation 0.5s/floor) |
| `SW[9]` | `U8` | Maintenance Mode (Ignores passenger calls) |
| `SW[10]` | `R16` | Auto-Park Mode (Returns to Floor 0 after 10s idle) |
| `SW[11]` | `T13` | Fire Service Recall (Forces immediate return to Floor 0) |

### Pushbuttons & Reset
| Button | Pin | Function |
| :--- | :--- | :--- |
| `CPU_RESETN` | `C12` | Active-Low Master Reset (Red CPU Reset Button) |
| `BTNC` | `N17` | Emergency Stop (Toggle Latch) |
| `BTNU` | `M18` | Door Hold Open |
| `BTND` | `P18` | Door Force Close |
| `BTNL` | `P17` | Door Obstruction Sensor |
| `BTNR` | `M17` | Overweight Sensor |

### LEDs & Displays
| Peripheral | Pins | Description |
| :--- | :--- | :--- |
| `LED[7:0]` | `U16..H17` | Pending Floor Requests |
| `LED[12:8]` | `V15..V16` | Door Timer Bar Graph (1 to 5) |
| `LED[13]` | `V14` | Door Open (Solid ON) |
| `LED[14]` | `V12` | Moving DOWN (Blinking at 6 Hz) |
| `LED[15]` | `V11` | Moving UP (Blinking at 6 Hz) |
| `RGB16` | `N15, M16, R12` | LD16 Cabin Status (Red, Green, Blue) |
| `RGB17` | `N16, R11, G14` | LD17 Cabin Status (Red, Green, Blue) |
| `AN[7:0]` | `U13..J17` | 8-Digit Common Anodes (Active-Low) |
| `SEG[6:0]` | `L18..T10` | 7-Segment Cathodes `{CG..CA}` (Active-Low) |
| `DP` | `H15` | Decimal Point (Tied High / OFF) |

---

## ⚡ Getting Started (Vivado)

### 1. Clone the Repository
```bash
git clone https://github.com/<your-username>/elevator-controller-nexys-a7.git
cd elevator-controller-nexys-a7
```

### 2. Create Vivado Project
1. Open **Xilinx Vivado (2020.2 or newer)**.
2. Create a new RTL project targeted for **Nexys A7-100T** (Part: `xc7a100tcsg324-1`).
3. Add design source:
   - `elevator_top.v`
4. Add simulation testbench:
   - `tb_elevator.v`
5. Add constraints file:
   - `nexys_a7_elevator.xdc`

### 3. Run Behavioral Simulation
1. Set `tb_elevator` as the top module in Simulation Sources.
2. Click **Run Simulation > Run Behavioral Simulation**.
3. Observe state transitions, single-cycle edge pulses, and LOOK/SCAN scheduling behavior.

### 4. Synthesize, Implement & Program
1. Click **Generate Bitstream**.
2. Connect your **Nexys A7-100T** via micro-USB.
3. Open **Hardware Manager > Auto Connect**.
4. Program device with the generated `.bit` file.

---

## 🧪 Simulation Verification

The testbench (`tb_elevator.v`) validates the following critical behaviors:
1. **LOOK/SCAN Turnaround**: Calling higher floors sequentially, followed by lower floors, verifying no out-of-order reversals.
2. **Emergency Stop Latching**: Verifies the cabin immediately halts, holds state, and resumes upon a second toggle.
3. **Obstruction Handling**: Injects obstruction pulses while doors close, verifying instantaneous abort and reopening.
4. **Fire Service Recall**: Enforces priority interrupt to recall cabin to Floor 0.

---

## 📄 License
This project is released under the [MIT License](LICENSE).

