# Hangman FPGA SoC System

**SystemVerilog · C++ · Vivado · Vitis · MicroBlaze MCS**

A Hangman game running on an FPGA, built as a hardware/software co-design project: I designed a custom VGA **sprite peripheral in SystemVerilog** and its **bare-metal C++ driver**, integrated into a MicroBlaze MCS soft-core SoC. The game renders to a VGA monitor, takes PS/2 keyboard input, and plays audio — but the point of the project is the boundary between custom RTL and the firmware that drives it.

> **Platform:** Nexys 4 DDR (Artix-7 XC7A100T) · MicroBlaze MCS soft-core · Vivado 2019.2 · Vitis · bare-metal C++
> Built on Pong Chu's *FPro* framework (*FPGA Prototyping by SystemVerilog Examples*).

---

## What This Project Demonstrates

1. **HW/SW co-design** — I wrote both the RTL for a custom video peripheral *and* the C++ driver that controls it, so I understand the register-map contract from both sides.
2. **Memory-mapped peripheral integration** — adding a custom core into an existing pixel pipeline means conforming to its bus and streaming contracts.
3. **Resource-aware design** — a 2-bit indexed-color palette keeps the sprite RAM 6× smaller than storing full color per pixel.

---

## System Architecture

The system is a MicroBlaze MCS soft-core CPU running bare-metal C++, talking to two subsystems over a simple memory-mapped bus (the *FPro* bus). A bridge translates the MCS I/O bus into the FPro bus and uses `addr[23]` to route each access to either the **MMIO subsystem** (simple peripherals) or the **video subsystem** (the pixel pipeline).

```
┌─── MicroBlaze MCS (soft core, 100 MHz) ──────────────────┐
│   main_video_test.cpp   (bare-metal C++ game logic)      │
│        ↕  MCS I/O bus                                    │
└──────────────────────┬───────────────────────────────────┘
                       │
              chu_mcs_bridge        ← top-level wiring
   (MCS I/O bus → FPro bus; base = 0xC000_0000,
    word-aligned, addr[23] selects video / mmio subsystem)
                       │
          ┌────────────┴─────────────┐
          ▼                          ▼
   mmio_sys (MMIO subsystem)   video_sys (daisy-chain pixel pipeline)
   LED, switches, UART,         frame_buf → bar → gray →
   PS/2 keyboard, DDFS audio,     ★ hangman (slot V5) ★ → ghost →
   SSEG, SPI, I2C, timer…         osd → mouse → vga_sync → RGB[11:0] → VGA
                                (640×480, 25 MHz pixel clock, chroma-key compositing)
```

**Clocking:** an MMCM derives `clk_100M` (system/CPU) and `clk_25M` (VGA pixel clock) from the 100 MHz board clock.

**Data flow (one keypress):**
`PS/2 key → ps2_core → C++ Ps2Core::get_kb_ch() → game logic decides hit/miss → driver writes a memory-mapped register → the affected video core paints the change on the next frame.`

---

## What I Built vs. Framework

This project is built on Pong Chu's *FPro* framework — the `chu_*` library cores (frame buffer, bar, gray, ghost, osd, mouse, vga_sync, video controller) come from the book. To keep scope honest:

| File | Author | Role |
|------|--------|------|
| `chu_vga_hangman_core.sv` | **Me** | Custom video-slot peripheral (register decode + chroma-key) |
| `hangman_src.sv` | **Me** | Sprite generator (in-region test, palette lookup, animation) |
| `hangman_ram.sv` | **Me** | Sprite RAM (`$readmemb` loads the bitmap) |
| `hangman_bitmap.mem` | **Me** | 8-stage hangman bitmap data |
| `mcs_top_complete.sv` | **Me (wiring)** | Top-level: MMCM + MCS + bridge + subsystems |
| `video_sys_daisy.sv` | **Me (modified)** | Inserted the hangman core at slot V5 |
| `main_video_test.cpp` | **Me** | Bare-metal C++ game logic |
| `chu_*.sv`, `*_core.cpp/.h` | Pong Chu (*FPro*) | Framework cores and their drivers |

Writing one core *correctly* still means understanding the whole chain: every core obeys the same `si_rgb → so_rgb` streaming contract, a 1-clock delay for alignment, and a shared register-decode convention.

---

## How It Works

### The video pipeline (daisy chain)

The pixel pipeline answers one question every 25 MHz tick: *"for the pixel currently being scanned, what color?"* A `frame_counter` broadcasts the current `(x, y)` to every core. Each core is a layer — it either paints its own color for that pixel or passes the incoming color through. Layers composite bottom-to-top, like image layers, using **chroma-key** transparency.

This has to be hardware, not the CPU: VGA needs a new pixel every 40 ns with rock-steady sync, forever. A 100 MHz polling CPU gets only ~4 instructions per pixel — it physically can't guarantee continuous, zero-jitter output. So the CPU only updates *what* to draw (writing registers); dedicated hardware paints it 60 times a second.

### The hangman sprite core

The core is a movable 32×32 sprite generator. The CPU writes its position `(x0, y0)` and a control register (which of 8 hangman images, body color, auto-animate). Each tick, for the scan point `(x, y)`:

```
xr = x - x0;  yr = y - y0;              // relative coordinate
in_region = (0 ≤ xr < 32) && (0 ≤ yr < 32);
addr_r = {sid, yr[4:0], xr[4:0]};        // look up sprite RAM
plt_code = RAM[addr_r];                  // 2-bit palette index
rgb = palette(plt_code);                 // index → 12-bit color
```

### 2-bit indexed-color palette

The sprite RAM stores a **2-bit palette index** per pixel, not a full 12-bit color. A small palette table maps the index to color:

| index | meaning |
|-------|---------|
| `00` | transparent (chroma key) |
| `01` | stick (dark) |
| `10` | body (color selectable via control register) |
| `11` | white |

**Why:** 8 images × 32×32 = 8192 pixels. Storing full 12-bit color would need 8192 × 12 ≈ 98 Kbit; 2-bit indices need only 8192 × 2 = 16 Kbit — **6× smaller**. This is the classic indexed-color technique used by old game consoles, GIF, and VGA mode 13h.

### Chroma-key compositing

```systemverilog
assign chrom_rgb = (sprite_rgb != KEY_COLOR) ? sprite_rgb : si_rgb;
assign so_rgb    = bypass_reg ? si_rgb : chrom_rgb;
```

If the sprite's pixel is the key color, the background (`si_rgb`) shows through; otherwise the sprite's color wins. A `bypass` register can hide the layer entirely.

### Register map (hangman core)

`addr[13]` splits between sprite RAM and registers; `addr[1:0]` selects the register:

| address | target |
|---------|--------|
| `addr[13]=0` | sprite RAM (8 bitmaps, 2-bit/pixel) |
| `addr[1:0]=00` | bypass (1 = hide layer) |
| `addr[1:0]=01` | x0 (sprite top-left X) |
| `addr[1:0]=10` | y0 (sprite top-left Y) |
| `addr[1:0]=11` | ctrl ([2:0] image id, [3] auto-animate, [5:4] body color) |

### Bare-metal C++ driver

There's no OS. Each peripheral is a C++ class constructed with its memory-mapped base address (via `get_slot_addr()`), and all access is memory-mapped read/write through the Xilinx BSP. The class wraps the raw `*(volatile uint32_t*)(base + offset)` pokes so game code reads cleanly — `hangman.move_xy(100, 50)` instead of manual pointer arithmetic. Access is polling, no interrupts.

---

## Gameplay

- 640×480 VGA output: title, blanks for the answer, the hangman drawn stage by stage, sprites
- PS/2 keyboard: select category (0 = MicroBlaze / 1 = Verilog), guess letters
- DDFS audio: distinct tones for correct / wrong / win / lose
- 7 lives; a wrong guess advances the hangman one stage; guess the word to win, run out of lives to lose

---

## Repository Structure

```
project_Lab_final/      Vivado project — HDL sources (hangman core + FPro framework)
vitis_Lab_finalv2/      Vitis project — bare-metal C++ application
README.md
```

---

## Build & Run

1. Open `project_Lab_final` in Vivado 2019.2, generate the bitstream, and export the hardware (XSA).
2. Import the hardware into Vitis and build the application from `vitis_Lab_finalv2`.
3. Program the Nexys 4 DDR, connect a VGA monitor and PS/2 keyboard, and run.
