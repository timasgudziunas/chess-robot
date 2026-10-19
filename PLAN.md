# PLAN.md: chess-robot build plan

Phases run top to bottom. Each phase ends in a working, tested state and gets its own commit (`Phase N complete: <detail>`). Tick checkboxes as tasks finish. Hardware phases are done only when the physical test described has actually happened, not when the code is written.

Open decisions in `CLAUDE.md` must be closed before the phase that lists them.

## Phase 0: Decisions and BOM

Lock the hardware choices and order parts so lead time overlaps with software work.

- [ ] Confirm motor type (serial bus servos recommended). Pick exact model and count (5 or 6 joints + wrist rotation).
- [ ] Confirm board and piece set. Measure square size, every piece's height and grip diameter, and record them in `config.toml`.
- [ ] Rough torque budget: longest extension holding the heaviest piece plus the two-gripper wrist and tubing. Confirm chosen servos clear it with 2x margin.
- [ ] Pneumatics: pump, reservoir, regulator, valves, tubing, fittings. Confirm the regulator's max output is below the finger rating.
- [ ] Confirm Pi model, camera, screen, rotary encoder, power supplies, e-stop switch, servo bus adapter.
- [ ] Write `docs/bom.md` with every part, price, link, purpose. Order parts.
- [ ] Draft `config.toml`: board dims, piece table, servo IDs, provisional link lengths and tool offsets, pressure limits.

**Done when:** `docs/bom.md` is committed, parts are ordered, and `config.toml` loads with every key present (placeholders marked `provisional`).

## Phase 1: Software skeleton, no hardware

Play chess against Stockfish in the terminal using the project's real config and engine wrapper.

- [ ] `requirements.txt`, `robot/config.py` (loads toml, validates types and required keys, raises on anything missing).
- [ ] `robot/chess_engine.py`: python-chess board, Stockfish at configured skill level, legal move validation, game-over detection.
- [ ] `robot/board_geometry.py`: square name to board-frame xyz, piece height and grip height lookup, capture bin slot allocation.
- [ ] `robot/cli.py play --no-arm`: terminal game, human types moves in SAN or UCI, engine replies.
- [ ] Tests for config loading, geometry, and engine wrapper.

**Done when:** a full game runs to checkmate or resignation in the terminal on the laptop, `pytest` green.

## Phase 2: Kinematics and motion planner

Prove the arm geometry reaches everything with both grippers before printing anything.

- [ ] `robot/kinematics.py`: FK and IK for the chosen DOF count, joint limits from config, two tool offsets (gripper A and B), reachability check with margin.
- [ ] `robot/motion_planner.py`: move to waypoint list. Quiet moves, one-trip captures using both grippers, castling, en passant.
- [ ] Simple matplotlib 3D plot of the arm at a pose and over a waypoint sequence, for eyeballing.
- [ ] Tests: every square and every bin slot reachable by both grippers from the configured base pose with margin. Planner tests for each move type.
- [ ] Fix link lengths, tool offsets, and base position in `config.toml` from the reachability results. Close the arm reach open decision.

**Done when:** `pytest` proves full reachability for both grippers, planner tests pass, link lengths are committed.

## Phase 3: Soft gripper and pneumatics bench prototype

The gripper is the riskiest part of the whole build. Prove it on a stand before designing the arm around it.

- [ ] Run the shared calibration comb print (the "Calibrate clearances, holes, walls, and minimum features" item under Open tasks in `../CLAUDE-CAD.md`) if not yet done. Clearance and hole values in `../cad-standards.toml` must be measured before any fitted part is printed.
- [ ] `designs/soft-gripper/`: finger geometry and manifold, parametric in Fusion. Print in TPU (or print a mould and cast silicone if TPU is too stiff).
- [ ] Bench rig: one gripper on a fixed stand over the board, pump, regulator, one valve, manual switch.
- [ ] `docs/pneumatics.md`: schematic, pressure limits, part list.
- [ ] `robot/pneumatics.py` first version: pump on/off, valve open/close, pressure read if a sensor is fitted. `robot/cli.py grip` on the Pi or laptop GPIO board.
- [ ] Test: pick and release each piece type 10 times at a tuned pressure. Record pressure and dwell per piece type into `config.toml`.
- [ ] Decide: soft gripper passes, or fall back to a rigid parallel-jaw gripper. Record the decision and data in `HANDOFF.md`.

**Done when:** every piece type lifts and releases 10 of 10 on the bench, or the fallback decision is made and documented.

## Phase 4: CAD and print the arm

Model every part parametrically in Fusion 360 through the MCP workflow, print, assemble mechanically.

- [ ] Parts: base, shoulder, upper arm, forearm, wrist with 90 degree rotation and two gripper mounts, servo housings, tubing and cable routing, capture bin. One `designs/<part>/` each.
- [ ] Link lengths and tool offsets in Fusion user parameters match `config.toml`. Export `parameters.json` for each part. Add the config-vs-parameters test from CLAUDE.md rule 4.
- [ ] Screenshot every part into `snapshots/` at the end of each CAD session.
- [ ] Print, test-fit servos and grippers, iterate. Assemble unpowered, route air lines so they survive full wrist rotation.

**Done when:** all joints move freely by hand through their configured ranges, servos and both grippers are mounted, `parameters.json` for every part is committed and matches config.

## Phase 5: Electronics and Pi bring-up

Power the arm safely and move every joint and valve from the command line.

- [ ] `docs/wiring.md`: servo rail, pump rail, e-stop cutting both, bus adapter, valves, camera, screen, encoder.
- [ ] Wire it. E-stop tested before servos or pump are connected.
- [ ] `robot/servo_bus.py`: set servo ID, read position, move to angle with speed cap, refuse out-of-limit commands.
- [ ] `robot/cli.py jog`, `home`, `grip A|B`. Home pose defined in config.
- [ ] Pi setup: OS, Python env, Stockfish, repo cloned, `pytest` green on the Pi.
- [ ] Record real joint zero offsets into `config.toml`. Write `docs/calibration.md`.

**Done when:** from the Pi, `home` works, `jog` puts each gripper over all four corner squares and the bin, and `grip` actuates each gripper, verified by eye.

## Phase 6: Pick and place

The arm moves real pieces reliably.

- [ ] `robot/cli.py move e2e4` executes a planner sequence on hardware.
- [ ] Reliability run: 20 consecutive quiet moves across the board including far corners, alternating grippers. Log every failure with cause.
- [ ] One-trip capture sequence: A lifts the captured piece, wrist rotates, B places the mover, A drops into the bin.
- [ ] Castling and en passant sequences on hardware.

**Done when:** 20 of 20 varied moves succeed, plus one capture, one castle, one en passant, each on video or witnessed.

## Phase 7: Full game, typed human moves

End-to-end game with the human typing their moves. This is the first "it plays chess" milestone and the gate before vision.

- [ ] `robot/game_loop.py`: human types move, validated, engine replies, arm executes, repeat to game over.
- [ ] Game-over handling: arm returns home, result printed.
- [ ] Error path: any hardware fault stops the game, vents the grippers, logs state.

**Done when:** one complete game is played against the robot with no manual intervention.

## Phase 8: Vision in shadow mode

Camera reads the board but does not yet drive the game.

- [ ] Overhead camera mount designed in Fusion (`designs/camera-mount/`), printed, installed.
- [ ] `robot/camera.py`: capture frame. `robot/board_vision.py`: find board corners, warp to square grid, per-square occupancy, diff two states to an inferred move given the known legal move list.
- [ ] Calibration routine: register board corners, save to config. Document in `docs/calibration.md`.
- [ ] Shadow mode: during typed games, vision logs its inferred move beside the typed move.

**Done when:** vision agrees with typed input on 50 consecutive human moves under normal room lighting.

## Phase 9: Autonomous play

Vision replaces typing.

- [ ] Human confirms their move by pressing the knob (or Enter until Phase 10); vision infers the move; an ambiguous diff asks for a typed fallback.
- [ ] Full game with no keyboard input except confirm.

**Done when:** a complete game is played with no typed moves.

## Phase 10: Screen and knob interface

Game setup without a keyboard or SSH session.

- [ ] `designs/control-panel/`: enclosure for screen and encoder, printed.
- [ ] `robot/interface.py`: menu for colour, difficulty, new game; shows engine's move and game state; knob press confirms the human's move.
- [ ] Boot to the interface on power-up (systemd service).
- [ ] Refresh `HANDOFF.md` and the `CLAUDE.md` tree to match reality. Delete anything in the plan that was never built.

**Done when:** power on, pick settings with the knob, play a complete game, power off, with no keyboard or SSH.

## Stretch (needs check-in before starting)

- Promotion with spare pieces staged beside the board.
- Pressure sensor feedback to detect a missed grip before lifting.
- Adjustable engine strength mid-game.
