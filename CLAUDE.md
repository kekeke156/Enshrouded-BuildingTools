# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A mod for Enshrouded (Windows game) shipped as a single proxy `dinput8.dll` with **no C runtime**. It hooks the game's building code (found by byte signatures) to add Area mode, Replace, connected delete, free placement, object lines, an in-game menu, and a Vulkan-drawn 3D preview. It is cross-compiled from Linux with clang/lld for `x86_64-w64-mingw32`. The game itself cannot be run here; verification is the build plus unit tests.

## Commands

```sh
./build.sh             # build/dinput8.dll
./build.sh test        # build + unit tests (host compiler; HOSTCXX overrides)
./build.sh package     # build + tests + dist/EnshroudedBuildingTools-<version>.zip (what CI runs)
ENSHROUDED_EXE=/path/to/enshrouded.exe ./build.sh test   # also run game-binary signature checks
```

There is no per-test filter: `tests/test_core.cpp` is one binary (`build/test_core`). Planner and translation-file tests always run; signature checks (each `SIG_*` in `src/main.cpp` matches exactly once at the expected RVA, hooks steal position-independent bytes) run only when the game exe is given. The exe must never be committed.

Translation helper (`src/strings.h` is the source of English texts; tests check every `lang/*.txt` against it):

```sh
python3 tools/lang.py en       # regenerate lang/en.txt from src/strings.h
python3 tools/lang.py status   # missing/extra keys per language
python3 tools/lang.py sync     # reorder all lang files like en.txt, add missing keys as "# key = English"
```

Shaders: `src/shaders/*` are compiled to the checked-in `src/*_vert.h` / `src/*_frag*.h` headers with `glslangValidator` (fragment shaders twice, second time with `-DDEPTH` and a `Depth` name) — see `docs/BUILDING.md`. Regenerate after any shader edit.

## Architecture

- **Single translation unit.** `src/main.cpp` is the whole mod and `#include`s `strings.h` (X-macro `BT_STRINGS`), `vkdraw.h` (Vulkan overlay) and `objects.h` (object-in-hand placement) mid-file. Code must stay freestanding: `-ffreestanding -fno-exceptions -fno-rtti`, no CRT/STL; `memcpy`/`memset` are defined in `main.cpp`, `___chkstk_ms` in `chkstk.S`, own entry point `DllMainCRTStartup`.
- **`src/core.h`** holds platform-free logic (signature scanner, area planner `plan_fill`/`plan_area`, roof/round/diagonal fills, fixed-point rounding). It is shared with the tests — put testable logic here, not in `main.cpp`.
- **`src/cave.S`** is replacement code patched into the game's voxel write loop (Replace).
- **Hooks**: inline 5–10 byte jumps to relays + trampolines in one 64 KB block near the game image; chains onto other mods' entry jumps. Functions other mods also hook by their first bytes (Creative Mode's `PayCost`, the play-state UI hooked by GreeneyeDS mods) are reached by retargeting their `call` sites instead. Compatibility with other `dinput8.dll` mods (XHL Native Core chain loader, Creative Mode, XHL Planting Assistant, GreeneyeDS mods) is a recurring concern.
- **Building pipeline**: the game's ECS runs place/remove in the in-process server world and client prediction worlds; dry run (`PlaceShape`/`RemoveShape`) → payment (`PayCost`) → commit. The mod acts only in the **server world**; clients just supply the cursor. Per-player settings travel to dedicated servers packed into existing input fields (e.g. round/diagonal fills encoded as roof fill values) — keep these encodings consistent between `main.cpp`, `core.h` and the `cells.vert` shader, which repeats the roof rule from push-constant bits.
- **Undo budget**: undo steps hold ≤32 records and the pool is tiny; the mod records undo only while it clearly fits and trims the cost list so undo refunds exactly what it takes back.

Read `docs/HOW-IT-WORKS.md` for the design and `docs/reverse-engineering-notes.md` for every game address (RVAs). `tools/re/` has the Python (capstone) scripts for analysing the exe; `sigs.py` generates signatures.

## Conventions

- **Versioning per commit**: each development commit bumps `MOD_VERSION` in `src/main.cpp` to the next `x.y.z-devN` and puts it in the commit subject, e.g. `Leave plants to XHL Planting Assistant when it plants (0.31.2-dev4)`. The release commit drops the suffix (`Release x.y.z: ...`). Merging a new `MOD_VERSION` to `main` auto-publishes a GitHub release and Nexus upload.
- **CHANGELOG.md**: each `## x.y.z` section is used verbatim as GitHub/Nexus release notes — keep it short and written for players. Update `package/BuildingTools-README.txt`, `README.md` and docs when features or keys change.
- **New player-visible text**: add it to `src/strings.h`, then run `tools/lang.py en` and `tools/lang.py sync`. English and Polish are reviewed; other languages are drafts.
- **New options**: the ini is written with comments by `main.cpp` (config section); older ini files keep their values and gain new options.
- **Game update**: run the binary checks, re-find moved functions using the notes, regenerate signatures with `tools/re/sigs.py`, update `src/main.cpp` and the expected RVAs in `tests/test_core.cpp`.
- Prose style in docs, comments and changelog is plain, short sentences in simple English.
