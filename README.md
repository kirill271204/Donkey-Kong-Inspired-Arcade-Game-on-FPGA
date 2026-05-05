# Donkey-Kong-Inspired-Arcade-Game-on-FPGA
## Overview
This project provides the design of a Donkey Kong-inspired arcade game constructed entirely in Verilog HDL and deployed on the Digilent Nexys 4 DDR FPGA development board. The game outputs a 640×480 VGA signal at 60 Hz and implements animated sprites, gravity-based jumping, rolling barrel obstacles, collision detection, score tracking, and end-screen overlays, all in synchronous hardware with no software or processor involved. The developed game consists of two main characters: Mario and Donkey Kong. The objective of the game is for the player to navigate the Mario sprite from the bottom of the screen across seven sloped girder platforms, connected by eight ladders, to reach the trophy on the uppermost platform. A time-based score encourages fast completion. The obstacle to Mario’s progression is the continuous stream of barrels spawned by Donkey Kong, which roll along the floors and must be avoided or jumped over. If Mario comes into contact with a barrel, he loses one life, and if he loses all the lives, the game is over.

## How It Works
### System Architecture
The design is organised into three domains, all coordinated by a single **frame tick** signal that pulses once per vertical blanking interval:
- **Timing:** clk_div divides the 100 MHz board clock to 25 MHz. The vga_sync generates horizontal/vertical counters and sync pulses. The frame_tick signal (vpos == 481 && hpos == 0) gates all game-state updates to once per frame.
- **Game Logic:** All state machines update on frame_tick. The player FSM, barrel pool, score counter, life tracker, and win/lose conditions all advance synchronously.
- **Rendering:** Each module produces pixel colour based on (hpos, vpos) using scanline rendering (combinational pixel-membership checks and bitmap ROMs). A priority multiplexer composites all layers with no frame buffer required.

### Module Breakdown
| Module | Role |
| --- | --- |
| `dk_top` | Top-level: instantiates all<br/> modules, routes signals,<br/>manages game state (lives,\\n score, win/lose) |
| `clk_div` | ±150 V ; ±20 A |
| `vga_sync (×2)` | 56 µF (890764429002CS) |
| `scene_render` | 27 µH (AGP4233-153) |
| `info_bar` | VS-E5PH6012 |
| `gorilla` | 7.5 Ω |
| `Switching period` | 150 µs (6.667 kHz) |
| `Positive pulse width` | 10 µs |
| `Negative pulse width` | 5 µs |

