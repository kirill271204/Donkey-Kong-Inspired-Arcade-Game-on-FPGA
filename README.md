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
| `dk_top` | Top-level: instantiates all modules, routes signals, manages game state (lives,\\n score, win/lose) |
| `clk_div` | Divides 100 MHz → 25 MHz pixel clock |
| `vga_sync (×2)` | Generates hpos, vpos, hsync, vsync, and frame_tick |
| `scene_render` | Draws sloped girder platforms and ladders using bitwise arithmetic |
| `info_bar` | Renders title text, score, and heart-shaped life icons in the top 100 px |
| `gorilla` | Animated Donkey Kong sprite with a 4-state FSM (NORMAL → REACH → HOLD → THROW) |
| `player` | Mario sprite with a 4-state FSM (WALK, CLIMB, FALL, DYING), gravity accumulator for jumps |
| `barrels` | Manages a pool of 24 barrel_unit instances; spawns one barrel every 150 frames |
| `barrel_unit` | Independent rolling barrel with bounding-box collision detection |
| `dust` | 5-phase particle animation triggered on player or barrel landing |
| `blood` | 6-phase splatter animation on player–barrel collision |
| `trophy` | Static win-objective sprite; triggers player_wins on bounding-box overlap |
| `game_over_screen` | Centred "GAME OVER" overlay with dark background box |
| `win_screen` | Centred "YOU WIN" overlay with dark background box |

### Player Mechanics
The player FSM governs all movement:
- WALK: Left/right buttons move ±2 px per frame; vertical position snaps to the current floor.
- JUMP: Centre button sets a negative vertical velocity; the FSM transitions to FALL.
- FALL: A gravity accumulator (vy <= vy + 1) updates py each frame, producing a parabolic arc. Landing is detected when py re-enters a floor band.
- CLIMB: When overlapping a ladder and pressing up/down, py changes by ±2 px per frame until the next floor is reached.
- DYING: Triggered on barrel collision; decrements lives.

### Barrel System
- 24 independent barrel_unit instances managed by a parent barrels module.
- A spawn counter fires every 150 frames, activating the lowest-indexed idle unit.
- Each barrel rolls along sloped platforms independently.
- Collision is checked per-unit via bounding-box overlap with the player.
- Barrel speed is globally adjustable via SW[1:0] slide switches (difficulty control).

### Rendering Approach
All graphics are generated combinationally each pixel clock and no frame buffer is used:
- Platforms are computed arithmetically: slopes via hpos >> 5, zigzag fill via lower bits of hpos.
- Ladders use fixed coordinate ranges with rungs at vpos[2:0] == 0.
- Sprites use 16×16 bitmap ROMs displayed at 2× scale (32×32 on screen), with separate colour masks for body parts.
- Text uses an 8×8 font ROM displayed at 2× scale, indexed by a case-selected character ID.
- Compositing is a priority multiplexer (blood > win screen > game over > player > trophy > barrels > gorilla > dust > info bar > scene).

## Hardware
- Board: Digilent Nexys 4 DDR (Artix-7 XC7A100T)
- Display: 640×480 @ 60 Hz VGA
- Inputs: 5 push buttons (movement + jump), 2 slide switches (difficulty)
- Outputs: VGA RGB, 7-segment display (score + elapsed time), LEDs (player state)

## Testing
Three testbenches were developed in Vivado's behavioural simulator:
- VGA Sync: Verifies hpos counts 0–799, vpos counts 0–524, and sync pulses fire at the correct intervals per the VESA 640×480 @ 60 Hz standard.
- Player FSM: Confirms walk movement (±2 px/frame), jump parabola via gravity accumulator, and correct state transitions (WALK → FALL → landing → WALK).
- Barrel Unit: Validates synchronous spawn activation, rolling movement, and bounding-box collision assertion/deassertion.

To accelerate simulation, vpos and hpos were held at their frame_tick values (481, 0), allowing FSM updates every clock cycle instead of once per full VGA frame.
