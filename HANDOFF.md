# HANDOFF.md

**State as of 2026-10-18 (bootstrap session).** Repo exists with governing docs only. No code, no CAD, no hardware ordered.

## Just completed

- `git init`, `.gitignore` (ignores `.env`, Python junk, all CAD binaries, `data/`, `logs/`).
- `CLAUDE.md` (imports `../CLAUDE-CAD.md`), `PLAN.md` (Phases 0 to 10 plus stretch), this file.
- Architecture fixed by owner this session: single articulated 5 or 6 DOF arm, wrist rotating 90 degrees between two soft pneumatic grippers, overhead camera vision on ordinary pieces and a standard board, screen plus rotary knob interface, Raspberry Pi runs everything.

## In progress

Nothing. Stopped cleanly at the docs commit.

## Next steps, in order

1. Phase 0: close the open decisions in `CLAUDE.md` (motor type and soft finger material are the big ones), write `docs/bom.md`, order parts, draft `config.toml` including the measured piece table.
2. Phase 1 can start on the laptop immediately, in parallel with parts lead time.
3. The shared CAD calibration comb print (the "Calibrate clearances, holes, walls, and minimum features" item under Open tasks in `../CLAUDE-CAD.md`) blocks Phase 3. Do it early.

## Open decisions and blockers

See the Open decisions table in `CLAUDE.md`. None of them block Phase 1.

Risk the owner has accepted: soft pneumatic grippers on ordinary pieces is the hardest gripping option. The plan bench-tests one gripper on a stand (Phase 3) before the arm is designed around it, and names a rigid parallel-jaw fallback. Vision is gated behind a full typed-move game (Phase 7).
