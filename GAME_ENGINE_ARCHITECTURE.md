# Google Research Football - Game Engine Architecture

This document explains where the game logic for the default gfootball engine is located and how it's structured.

## Quick Answer

The **core game engine is located in `third_party/gfootball_engine/src/`** and is written in **C++**. It's compiled into a shared library and accessed from Python through bindings.

## Architecture Overview

### Engine Type
- **Language:** C++ (compiled to native code)
- **Build System:** CMake
- **Output:** Shared library (`libgame.so` on Linux, `libgame.dylib` on macOS, `_gameplayfootball.dll` on Windows)
- **Python Bindings:** Exposed as `gfootball_engine` module with `GameEnv` class

### Build Process
```bash
# Build the engine (from repository root)
bash gfootball/build_game_engine.sh

# Or manually:
cd third_party/gfootball_engine/
mkdir -p build && cd build
cmake .. && make -j$(nproc)
```

## Game Logic Location

### Core Game Logic Components

All core game logic is in **`third_party/gfootball_engine/src/`**:

#### 1. Match & Game State (`src/onthepitch/`)
The main game simulation happens here:

| File/Directory | Purpose |
|----------------|---------|
| `match.cpp/.hpp` | **Main match controller** - orchestrates the entire game, manages game state, handles game loop |
| `team.cpp/.hpp` | **Team management** - team tactics, formations, player selection |
| `referee.cpp/.hpp` | **Rule enforcement** - offside detection, fouls, free kicks, penalties, goal validation |
| `officials.cpp/.hpp` | **Game officials** - linesmen and referee positioning |
| `ball.cpp/.hpp` | **Ball physics** - velocity, acceleration, friction, collision detection |

#### 2. Player Mechanics (`src/onthepitch/player/`)
Player behavior and physics:

| File/Directory | Purpose |
|----------------|---------|
| `player.cpp/.hpp` | **Player base class** - position, velocity, team affiliation, current action |
| `humanoid/` | **Physics-based humanoid model** - skeletal animation system, body parts, joints |
| `humanoid/humanoid.cpp` | **Humanoid physics** - skeletal animation, movement animation, collision geometry |
| `controller/` | **Player AI and control** - decision making, strategy execution |
| `playerdata.cpp/.hpp` | **Player attributes** - speed, stamina, skill ratings |

#### 3. AI & Strategy (`src/onthepitch/player/controller/`)
Player artificial intelligence:

| File/Directory | Purpose |
|----------------|---------|
| `playercontroller.cpp/.hpp` | **Main AI controller** - processes player decisions and actions |
| `strategies/` | **AI strategies** - different playing styles and tactics |
| `strategies/offtheball/` | **Off-ball movement AI** - positioning, defensive coverage, support runs |
| `elizacontroller.cpp/.hpp` | **Built-in AI** - default computer-controlled player behavior |

#### 4. AI Support Systems (`src/onthepitch/AIsupport/`)
AI decision-making infrastructure:

| File/Directory | Purpose |
|----------------|---------|
| `AIfunctions.cpp/.hpp` | **AI utility functions** - pass selection, shot decision, positioning |
| `mentalimage.cpp/.hpp` | **Game awareness** - player's understanding of game state (who has ball, teammate positions) |

#### 5. Physics & Math (`src/base/`)
Low-level physics and utilities:

| File/Directory | Purpose |
|----------------|---------|
| `geometry/` | **Collision detection** - AABB (Axis-Aligned Bounding Box), triangles, rays |
| `math/` | **Vector math** - 3D vectors, quaternions, matrices |
| `properties.cpp/.hpp` | **Configuration** - game settings and parameters |

#### 6. Graphics & Rendering (`src/systems/graphics/`)
Visual representation (OpenGL-based):

| File/Directory | Purpose |
|----------------|---------|
| `rendering/` | **OpenGL rendering** - shaders, textures, meshes |
| `graphics_scene.cpp` | **Scene management** - camera, lighting, 3D models |

#### 7. Scene Management (`src/scene/`)
3D scene structure:

| File/Directory | Purpose |
|----------------|---------|
| `scene3d/` | **3D scene graph** - nodes, hierarchical transformations |
| `objectfactory.cpp` | **Object creation** - spawning players, ball, stadium elements |

## Python Interface Layer

The C++ engine is wrapped by Python code in **`gfootball/env/`**:

| File | Purpose |
|------|---------|
| `football_env_core.py` | **Direct wrapper** around C++ `GameEnv` - sends actions, receives observations |
| `football_env.py` | **OpenAI Gym wrapper** - standardized RL interface (step, reset, render) |
| `config.py` | **Configuration** - scenario setup, player counts, rendering options |
| `observation_processor.py` | **State processing** - converts raw C++ state to RL-friendly observations |
| `wrappers.py` | **Environment wrappers** - reward shaping, observation stacking, etc. |

### How Python Calls C++ Engine

```python
# In football_env_core.py
import gfootball_engine as libgame  # Import compiled C++ library

# Create C++ GameEnv object
self._env = libgame.GameEnv()

# Step the simulation
observations = self._env.step(actions)

# Reset the game
self._env.reset(config)
```

## Game Loop Flow

1. **Python (`football_env.py`)** → Receives agent action
2. **Python (`football_env_core.py`)** → Converts action to C++ format
3. **C++ (`GameEnv`)** → Receives action via Python bindings
4. **C++ (`Match`)** → Updates game state (100 ms timestep)
5. **C++ (`Player`)** → Executes player actions
6. **C++ (`Ball`)** → Updates ball physics
7. **C++ (`Referee`)** → Checks rules (goals, fouls, offside)
8. **C++ (`GameEnv`)** → Returns observations to Python
9. **Python (`observation_processor.py`)** → Converts to RL observations
10. **Python (`football_env.py`)** → Returns (observation, reward, done, info)

## Key Game Engine Files

Here are the most important files for understanding game logic:

1. **`third_party/gfootball_engine/src/onthepitch/match.cpp`**
   - Main game loop and state management
   - Coordinates all game systems

2. **`third_party/gfootball_engine/src/onthepitch/player/player.cpp`**
   - Player behavior and state
   - Action execution

3. **`third_party/gfootball_engine/src/onthepitch/ball.cpp`**
   - Ball physics simulation
   - Collision with players and field

4. **`third_party/gfootball_engine/src/onthepitch/referee.cpp`**
   - Game rules enforcement
   - Goal detection, offside, fouls

5. **`third_party/gfootball_engine/src/onthepitch/player/controller/playercontroller.cpp`**
   - AI decision making
   - Strategy selection

6. **`third_party/gfootball_engine/src/onthepitch/AIsupport/mentalimage.cpp`**
   - Player game awareness
   - Tactical understanding

## Modifying Game Logic

If you want to modify game behavior:

### For Game Rules:
- Edit `third_party/gfootball_engine/src/onthepitch/referee.cpp`
- Examples: change offside rules, modify foul detection

### For Player Physics:
- Edit `third_party/gfootball_engine/src/onthepitch/player/player.cpp`
- Edit `third_party/gfootball_engine/src/onthepitch/player/humanoid/humanoid.cpp`
- Examples: change player speed, modify stamina system

### For Ball Physics:
- Edit `third_party/gfootball_engine/src/onthepitch/ball.cpp`
- Examples: adjust ball friction, modify bounce behavior

### For AI Behavior:
- Edit `third_party/gfootball_engine/src/onthepitch/player/controller/playercontroller.cpp`
- Edit files in `third_party/gfootball_engine/src/onthepitch/player/controller/strategies/`
- Examples: change AI difficulty, modify tactical decisions

### After Modifying C++ Code:
```bash
# Rebuild the engine (from repository root)
bash gfootball/build_game_engine.sh

# Or with pip (rebuilds and reinstalls)
python3 -m pip install -e .
```

## Original Engine Credit

The C++ engine is based on **Gameplay Football** by Bastiaan Konings Schuiling:
- Original open-source football game engine
- Modified by Google Research for RL applications
- Copyright notices in `third_party/gfootball_engine/src/` files

## Additional Resources

- **Compiling the Engine:** `gfootball/doc/compile_engine.md`
- **Environment API:** `gfootball/doc/api.md`
- **Observations & Actions:** `gfootball/doc/observation.md`
- **Scenarios:** `gfootball/doc/scenarios.md`

## Summary

**The default gfootball game logic is located in:**
```
third_party/gfootball_engine/src/onthepitch/
```

This C++ code handles:
- ✅ Match simulation and game state
- ✅ Player physics and humanoid animations
- ✅ Ball physics and collision detection
- ✅ AI decision making and strategies
- ✅ Rule enforcement (referee system)
- ✅ Team tactics and formations

The engine is compiled to a native library and exposed to Python via bindings in `gfootball/env/`.
