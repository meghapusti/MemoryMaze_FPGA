[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-24ddc0f5d75046c5622901739e7c5dd533143b0c8e959d652212380cedb1ea36.svg)](https://classroom.github.com/a/5YTzVbxp)
# 50.002 1D Project Group (Group_34)

### Group Members

Aung Khant Moe  1007196

Pusti Megha     1007128

Mali Janya      1007149

Koo Rou Zhen    1007038

Khoo Zi Qi      1006984

Glenda Koh      1007013

Justin Cho      1007496

### The Project

This project is a PvP arcade game where two players compete on a 4x4 grid to reach the goal first. Each player controls their position with four buttons: up, down, left, and right. Player positions are represented by white LEDs, restricted paths by red LEDs, and the goal by a green LED. There are three maps; pressing start transitions to the next map, briefly flashing it on both screens for one second before disappearing. Players must memorize each map quickly, navigating to the goal while avoiding restricted paths. The first to reach the goal wins; if the opponent wins, you lose automatically.

![image](https://github.com/50002-computation-structures/1d-project-group_34/assets/117970574/4d5e61b3-d2ff-4659-baa8-39e15827bf66)

#### User Manual
Here's a step-by-step instruction set on how to play "Memory Maze: Grid Escape":  

Objective: The goal of the game is to reach the finishing zone on a 4x4 grid before your opponent, without touching any of the restricted grids. 

Setup: Two players stand in front of their respective 4x4 grid screens. 

Start: Host or either player presses start to begin the game. 

Memorization: Upon pressing start, the restricted grids will flash for a brief moment, 1 second, on the screens. Players must memorise the location of these restricted grids, as they must avoid them while navigating to the finishing zone. 

Navigation: Players use 4 buttons to move left, right, up, or down on the grid. They must navigate through the grid to reach the finishing zone. 

Winning: The first player to reach the finishing zone without touching any of the restricted grids wins the game. The other player automatically loses.

## Modules Overview

### au_top.luc:
**Inputs and Outputs:**
- Inputs: Clock, reset, button inputs, etc.
- Outputs: User-controllable LEDs, LED strips, IO shield signals, etc.

**Constants:**
- Defines matrix dimensions, encoding settings, and LED colors.

**Components:**
- Utilizes flip-flops, RAM modules, ALU, FSM, etc.

**Initialization:**
- Initializes flip-flops, RAM addresses, LED colors, etc.

**Button Handling:**
- Detects button presses and conditions button inputs.

**Player Movement:**
- Handles player movement based on button inputs.

**Game Logic:**
- Updates RAM based on player movements and game events.
- Controls LED strip updates and colors based on game states.

**Game End Conditions:**
- Handles winning or losing conditions and updates LED colors accordingly.
- Connects LED outputs to LED strips and IO shield LEDs.

**Debugging and I/O:**
- Provides debugging information such as LED colors, player positions, RAM addresses, etc.

### data_ram.luc:
**Inputs and Outputs:**
- Inputs: Clock, reset, player positions, address, update signals, start signal, clear signal, map number, etc.
- Outputs: Encoded map data, ready flag, clear signal, game end flags, debug information, etc.

**Control Signals:**
- Controls various flags and signals based on game state and input conditions.
- Manages update requests, start signal, clear signal, etc.

**Initialization:**
- Initializes various internal signals and components to default values.

**RAM Management:**
- Handles read and write operations to a dual-port RAM.
- Writes player positions and map data to RAM based on game state.
- Reads map data from mapRom and provides it as encoded output.

**Game Logic:**
- Implements a finite state machine (FSM) to manage game states.
- Handles game initialization, player movement, and game end conditions.
- Checks map data at player positions for win or lose conditions.

**Debugging:**
- Provides debug information such as RAM addresses, data pointers, and game state flags.

### map_rom.luc:
**Inputs:**
- map_num[2]: A 2-bit input representing the selected map number.

**Outputs:**
- out_encoding[16][2]: A 2D array output representing the encoded values based on the selected map number. It has 16 rows and 2 columns.

**Constants:**
- Three ROMs are defined (MAP_0_ROM, MAP_1_ROM, and MAP_2_ROM), each containing encoded values. These ROMs are initialized with binary values representing the encoded outputs.

**Case Statement:**
- Used to select the appropriate ROM based on the map_num input.
- Depending on the value of map_num, the corresponding ROM (MAP_0_ROM, MAP_1_ROM, or MAP_2_ROM) is assigned to the out_encoding output.
- If the map_num input does not match any of the specified cases, MAP_0_ROM is assigned by default.

### ws2812b_writer.luc:
- Serves as a bridge between the FPGA and WS2812B LEDs.

### reverser.luc:
- Reverses every odd-numbered row, used for matrix manipulation.
