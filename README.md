# digital_oscilloscope_basys3

An FPGA-based **digital oscilloscope** implemented in **VHDL** for the **Digilent Basys 3 (Artix-7, xc7a35tcpg236-1)**.  
It samples an analog signal using the on-chip **Xilinx XADC**, stores in the block RAMs and renders a live waveform + UI on a **640×480 VGA** display.

---

## Features

- **XADC sampling** (on channel VAUX6)
- **On-chip sample buffer** stored in FPGA RAM (circular buffer, **1024 samples** by default)
- **Trigger** on threshold crossing, with a timeout-based “force trigger” to avoid stalling
- **Adjustable controls** via Basys 3 buttons:
  - Trigger level
  - Vertical position (offset)
  - Volts/div (display scaling)
  - Time/div (sampling period / decimation factor)
  - Run/Stop capture
- **VGA oscilloscope view**
  - Grid + axes
  - Waveform polyline rendering (column-wise cached Y positions for smooth drawing)
  - Trigger level marker
  - Text overlay: VOLT/DIV, TIME/DIV, TRIG, STATUS, RUN/STOP
- **7-segment display** shows the *currently active* control value (with unit indicators in the logic)

---

## Project Architecture (high level)

### Data path
1. **`xadc_module.vhd`**  
   Reads conversion results from XADC via DRP. The read address is set to the aux-channel result register (default VAUX6).  
   Output: `adc_data[11:0]` + `adc_valid` strobe.

2. **Sampling + trigger + buffer (in `oscilloscope_top.vhd`)**
   - A sample strobe is generated from `TIME_SCALE(time_per_div)` (table-driven decimation in **clock cycles**).
   - On `adc_valid`, samples are written into a **circular RAM buffer** (`block_ram.vhd`).
   - Trigger logic detects crossings around `trigger_level`. When triggered, it captures and advances pointers for display.

3. **Display pipeline**
   - **`vga_controller.vhd`** generates VGA timing and pixel coordinates.
   - At the start of each frame/line, waveform Y values are computed from RAM reads and cached into a `waveform_buffer(0..639)`.
   - Pixels are colored based on grid/axes/waveform/trigger/text priority.

### UI / controls
- **`oscilloscope_features.vhd`** debounces buttons and implements a simple control state machine.
  - `BTN_CENTER` cycles active control: Trigger → V-Pos → Volts/Div → Time/Div → …  
  - `BTN_UP/DOWN` adjust the active parameter
  - Provides `run_mode`, `trigger_level`, `vertical_pos`, `volts_per_div`, `time_per_div`, and a `status` code.

- **`display_decoder.vhd`** selects what numerical value to show on the **7-segment** display depending on active control.
- **`seven_segment_driver.vhd`** multiplexes the Basys 3 4-digit seven segment.
- **`simple_text_display.vhd`** draws the on-screen UI text using a character ROM.

---

## Repository Layout

- `src/` — VHDL sources (top-level + modules)
- `constraints/` — XDC constraints (Basys 3 pins, clock)
- `ip/` — Vivado IP configuration files (`*.xci`) **only**
- `scripts/` — Vivado Tcl scripts (recreate project / build)
- `vivado_reference/` — optional reference `.xpr` (not required to build)

---

## Build Instructions (Vivado)

**Vivado version:** The original project was created with **Vivado v2024.2**.

### Option A — Recreate the project from Tcl (recommended)
From the repo root:

```tcl
# In Vivado Tcl console or batch mode
source scripts/create_project.tcl
```

If your script expects an output folder (depends on how it was generated), use:

```bash
vivado -mode batch -source scripts/create_project.tcl -tclargs ./_vivado
```

Then open the generated `.xpr`, run **Synthesis → Implementation → Generate Bitstream**.

### Option B — Open the reference .xpr
Open `vivado_reference/final_project_oscilloscope.xpr` and build.
(Still recommended to keep build artifacts out of the repo.)

---

## Hardware Notes (Basys 3 + XADC)

- The XADC aux inputs are **analog** and must be within the allowed input range.  
  If you are measuring higher voltages, use an external **divider / conditioning circuit**.
- Default aux channel used in this repo is **VAUX6**.  
  To change the channel, update the DRP address in `xadc_module.vhd` (the register address corresponds to the desired channel’s conversion result).

---

## Configuration Quick Tips

- **Sample rate / Time-Div:** controlled by the `TIME_SCALE` table in `oscilloscope_top.vhd` and the 100 MHz board clock.  
  The sampling strobe roughly follows: `Tsample ≈ (TIME_SCALE[time_per_div] + 1) / 100e6`.
- **Volts-Div scaling:** controlled by `VOLTS_SCALE` table. This is a **display scaling factor**, not a calibrated voltage readout.
- **Buffer depth:** `RAM_ADDR_WIDTH=10` → 2^10 = **1024 samples**.

---

## License

MIT (see `LICENSE`).

---

## Screenshots / Demo

> Add a screenshot or a short GIF here once you capture it from a monitor or simulator.
