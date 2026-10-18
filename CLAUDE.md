# chess-robot
@../CLAUDE-CAD.md

A robotic chess opponent that plays on a standard light-and-dark chessboard with ordinary pieces. An overhead camera reads the human's move, Stockfish picks a reply, and a single 3D-printed articulated arm executes it. The wrist carries two soft pneumatic grippers and rotates 90 degrees between them, so a capture is one trip: gripper A lifts the captured piece, the wrist rotates, gripper B places the mover, and A drops its piece in the bin. A built-in screen and rotary knob handle game setup. A Raspberry Pi runs everything.

This is also the first real project for the shared AI + CAD workflow (`../CLAUDE-CAD.md`, `../cad-standards.toml`, `../docs/`). Every printed part is modelled parametrically in Fusion 360 via the MCP tools. Lessons about the CAD workflow itself go in the shared files, not here.

## Repo structure (planned)

```
chess-robot/
├── CLAUDE.md             # this file
├── PLAN.md               # phased build plan with checkboxes
├── HANDOFF.md            # current state, read first every session
├── config.toml           # ALL physical and tuning constants: board, arm links, servo limits, air pressure, camera, engine
├── .env.example          # template; real .env is gitignored (nothing secret today, kept for discipline)
├── requirements.txt
├── designs/              # one folder per printed or cast part
│   └── <part>/
│       ├── parameters.json   # exported Fusion user-parameter table; the diffable design intent
│       ├── exports/          # STL / STEP, gitignored
│       └── snapshots/        # screenshots after each CAD session, committed (visual changelog)
├── scripts/
│   ├── inspect/          # readOnly Fusion MCP scripts, safe any time
│   └── modify/           # Fusion MCP scripts that change geometry
├── robot/                # Python package, runs on the Pi; pure modules also run on the laptop
│   ├── config.py         # loads config.toml, validates, crashes on missing or malformed keys
│   ├── board_geometry.py # pure: square name -> board-frame xyz, piece heights and grip heights, bin slots
│   ├── kinematics.py     # pure: FK and IK for the arm, joint limit checks, both gripper tool offsets
│   ├── motion_planner.py # pure: chess move -> ordered waypoint list incl. wrist rotation and which gripper acts
│   ├── chess_engine.py   # python-chess board state + Stockfish wrapper
│   ├── board_vision.py   # pure: camera frame -> 64 occupancy states -> inferred move
│   ├── servo_bus.py      # HARDWARE: the only module that talks to the joint motors
│   ├── pneumatics.py     # HARDWARE: the only module that drives pump and valves
│   ├── camera.py         # HARDWARE: the only module that captures frames
│   ├── interface.py      # HARDWARE: screen and rotary knob
│   ├── game_loop.py      # orchestrates one game: sense, think, move, repeat
│   └── cli.py            # entry points: jog, home, grip, calibrate, play
├── tests/                # pytest; pure modules fully tested without hardware
└── docs/
    ├── bom.md            # every purchased part with price, link, and what it is for
    ├── wiring.md         # power, servo bus, valves, camera, screen, knob, e-stop
    ├── pneumatics.md     # pump, reservoir, regulator, valves, tubing, pressure limits
    └── calibration.md    # how to re-zero the arm, tune grip pressure, re-register the camera
```

## Tech stack

| Layer | Tool | Notes |
|---|---|---|
| Mechanism | Single articulated arm (5 or 6 DOF), 3D printed in PLA on the Prusa CORE One | Own parametric design in Fusion 360. May borrow geometry ideas from open designs but every part is modelled here. |
| Wrist | Rotates 90 degrees between two end effectors | Lets a capture happen in one trip. Planner treats each gripper as its own tool offset. |
| Grippers | Two soft pneumatic grippers, compressed-air actuated fingers | Must handle ordinary pieces of varied shape. Finger material and air source are open decisions. |
| Pneumatics | Small 12 V pump, reservoir, regulator, solenoid valves, tubing | Pressure limit lives in `config.toml` and is enforced by a mechanical regulator, not just software. |
| Motors | OPEN DECISION, see below | Serial bus servos with position feedback are the recommended default. |
| Controller | Raspberry Pi (5 recommended) | Runs engine, vision, planning, motor and valve control, interface. Code developed on the laptop, pulled onto the Pi via git. |
| Chess | `python-chess` + Stockfish | Stockfish binary on the Pi, skill level set from the knob and stored in `config.toml` defaults. |
| Vision | OpenCV + numpy, Pi Camera Module or USB webcam mounted overhead | Occupancy diff between frames, not piece recognition. The engine already knows what piece is on every square. |
| Interface | Small screen (SPI or HDMI) + rotary encoder with push | Game setup: colour, difficulty, new game, confirm move. Not a touchscreen, not a web UI. |
| CAD | Fusion 360 via MCP, rules in `../CLAUDE-CAD.md` | Physical constants from `../cad-standards.toml`. |
| Language | Python 3.11+ | Per `../CLAUDE.md` conventions: plain synchronous code, type hints, `logging`, no async. |

## Non-negotiables

1. **Safety first, every powered test.** A physical e-stop cuts the servo rail and the pump. It is wired and tested before the first powered motion. Software joint limits and a speed cap live in `config.toml` and `servo_bus.py` refuses commands outside them. A mechanical regulator caps air pressure below the gripper's rated limit regardless of software. No motion before a successful home. No hand in the workspace while the arm or pump is powered.
2. **Pure core.** `board_geometry`, `kinematics`, `motion_planner`, and `board_vision` take data in and return data out. No serial, no GPIO, no camera, no filesystem, no sleep. Only `servo_bus.py`, `pneumatics.py`, `camera.py`, and `interface.py` touch hardware. This is what makes the robot testable on a Windows laptop with nothing attached.
3. **Config over code.** Board square size, piece heights, arm link lengths, both gripper tool offsets, servo IDs, joint limits, grip pressures per piece type, valve timings, camera intrinsics, engine strength: all in `config.toml`. No magic numbers in Python.
4. **CAD and config must agree.** Link lengths and tool offsets used by `kinematics.py` come from `config.toml`. The same values appear in `designs/<part>/parameters.json`. A test asserts they match. Reprinting a link changes both in one commit.
5. **Bench-prove the gripper before the arm.** A single soft gripper on a stand must pick and release every piece type reliably (Phase 3) before arm CAD is finalised. If it cannot, the fallback is a rigid parallel-jaw gripper and that decision is made explicitly, not drifted into.
6. **Typed moves before vision.** The arm plays a full game with the human typing moves (Phase 7) before any vision code drives the game loop. Vision runs in shadow mode first and is trusted only once it agrees with typed input over a logged sample.
7. **Lean scope.** In scope: what the spec above lists. Out of scope without a check-in: app, web UI, voice, online play, chess clock, multiple boards, piece recognition by vision, promotion with spare pieces. Anything not in PLAN.md needs a check-in.
8. **Secrets via env only.** `.env` gitignored, `.env.example` committed. Nothing secret today; the rule stands anyway.
9. **CAD rules inherit.** Everything in `../CLAUDE-CAD.md` applies: centimetre-native API trap, `createByString('25 mm')`, named components, fully constrained sketches, screenshot after every modification, provisional values declared as provisional.

## Open decisions (confirm with owner before the affected phase)

| Decision | Default if forced | Affects |
|---|---|---|
| Motor type: serial bus servos (Feetech STS3215 class, position feedback, daisy-chain) vs steppers vs hobby PWM servos | Serial bus servos | BOM, `servo_bus.py`, joint housings, Phase 0 |
| Soft finger material: printed TPU bellows vs cast silicone (Ecoflex class) in a printed mould | Printed TPU first, silicone if TPU is too stiff | Phase 3 bench test, gripper CAD |
| Air source: 12 V diaphragm pump + small reservoir vs CO2 cartridge | Pump + reservoir | BOM, `docs/pneumatics.md`, noise |
| Valve topology: one 3-way valve per gripper vs shared supply with two 2-way valves | One 3-way valve per gripper | `pneumatics.py`, wiring |
| Arm reach: board size drives link lengths. Standard 2.25 in squares = 457 mm board plus a capture bin beside it | Compute from `config.toml` board dims in Phase 2, then fix | Kinematics, CAD, torque budget |
| Camera: Pi Camera Module 3 vs USB webcam | Pi Camera Module 3 | Overhead mount CAD, `camera.py` |
| Screen: small SPI display vs HDMI panel | Small SPI display, 2 to 3 in | `interface.py`, enclosure CAD |
| Pi model and power: separate supplies for logic, servo rail, pump | Pi 5, dedicated servo supply, pump on its own switched rail | BOM, wiring |

## Code conventions

Everything in `../CLAUDE.md` Coding style applies. Project specifics:

- Coordinates: board frame has origin at the a1 corner, x along files a to h, y along ranks 1 to 8, z up, all mm. Arm base pose relative to the board frame is a `config.toml` value. Every function that takes a position names its frame in the type hint or docstring.
- Angles in degrees at module boundaries, radians only inside math. Joint arrays are ordered base to wrist. Grippers are `A` and `B`; the wrist angle that puts each one pointing down is in config.
- `motion_planner` outputs a list of named waypoints (`approach`, `grip`, `lift`, `rotate_wrist`, `travel`, `lower`, `release`, `retreat`) each tagged with the active gripper. `servo_bus` and `pneumatics` execute them. Captures, castling, and en passant are planner-level sequences of ordinary steps, not special hardware paths.
- Grip is pressure plus dwell, both per piece type in config. `pneumatics.py` confirms pressure reached before the planner proceeds. A grip that does not reach pressure raises.
- Logging: INFO to stdout, DEBUG to `logs/` (gitignored). Every executed waypoint logs joint angles and valve state. Every vision decision logs the occupancy diff that produced it.
- Tests: `pytest`, no hardware. IK tests assert every square and bin slot is reachable by both grippers with a configurable margin. Planner tests assert sequences for quiet moves, captures (one trip), castling, en passant.
- Hardware modules fail loud: a servo that does not acknowledge, a pressure that does not build, a frame that does not arrive, a home that does not settle all raise and stop the game. Never retry motion silently.

## Reading order for a fresh session

1. `HANDOFF.md` (state, next step).
2. This file.
3. `PLAN.md` for the current phase only.
4. `config.toml` if touching any number.
5. `../cad-standards.toml` and `../docs/` only when doing CAD work (the import above already loads the CAD module).

Do not read `designs/*/exports/` or `logs/`. Do not open `.f3d` files; they are not in the repo.
