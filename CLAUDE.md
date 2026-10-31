# Aegis firmware project (ArduPlane fork)

This repo is an ArduPilot checkout used to understand and extend the firmware of the
**FrSky AEGIS Lite** fixed-wing flight stabilization controller. The shipped Aegis
firmware is a custom ArduPlane build by FrSky; this fork is where we rebuild and
modify it. Owner: wade3337-ctrl (personal GitHub). Work happens on branch `aegis-dev`.

## What the target hardware is (established by decoding the shipped .apj)

- Vehicle: **ArduPlane** on ChibiOS. Quadplane/VTOL compiled out (`HAL_QUADPLANE_ENABLED=0`).
- MCU: **STM32H730** (Cortex-M7). ArduPilot `board_id 7200` (custom, not upstream). USBID 0x1209/0x0002.
- Runs memory-mapped from **external QSPI flash at 0x90000000**; the `.apj` carries the
  firmware in `extf_image`, and `image` (internal flash) is empty. Internal `flash_total` 131072.
- Shipped build banner "Aegis V1.0-dev (68c2e4e0)"; feature strings place it at ~ArduPlane 4.6.
  Compiled with ARM GCC 10.2.1 (10-2020-q4-major), the same toolchain installed here.
- FrSky additions seen in the binary (not in upstream): `RTH point set/cleared`, `GPS_GAIN`,
  `RUDD_DT_GAIN`, `UP_EY_WIZARD`. "RTH" is FrSky's name for ArduPilot RTL.
  Working theory: the RTH point is a rally point (RTL's target comes from
  `calc_best_rally_or_home_location`); the gain knob rides on AP_Tuning or a copy of it.
- Sensors in the shipped firmware: Piksi/Swift RTK GPS (SBP), PX4Flow, LidarLite, baro,
  compass, airspeed, MSP-telemetry ESC, FPORT2 RC.
- Closest upstream board template: **SPRacingH7RF** (the one STM32H730 + OctoSPI
  external-flash board). An Aegis hwdef should be derived from it.

## Base version and branch

- Based on tag **Plane-4.6.3** (commit 3fc7011). Branch `aegis-dev` tracks `fork/aegis-dev`
  at https://github.com/wade3337-ctrl/ardupilot. `origin` is upstream ArduPilot/ardupilot.
- The clone is shallow (depth 1). Deepen with `git fetch --unshallow origin` if history is needed.

## Build environment (WSL Ubuntu, no sudo needed)

The build runs inside WSL (Ubuntu). Native Windows builds are not supported by ArduPilot.

- Repo path in WSL: `~/ardupilot` (`/home/wadej/ardupilot`).
  From Windows the same folder is `\\wsl.localhost\Ubuntu\home\wadej\ardupilot`.
- Toolchain: `~/opt/gcc-arm-none-eabi-10-2020-q4-major/bin` (not on PATH by default).
- Python deps: venv at `~/ardupilot-venv`.
- Build (baseline board; takes ~30 s):

```bash
export PATH=$HOME/opt/gcc-arm-none-eabi-10-2020-q4-major/bin:$PATH
source ~/ardupilot-venv/bin/activate
cd ~/ardupilot
./waf configure --board SPRacingH7RF
./waf plane
```

- Output: `build/SPRacingH7RF/bin/arduplane.apj` (plus `.bin`/`.hex`). This .apj has the same
  layout as the shipped Aegis firmware (empty internal image, populated `extf_image`).
- From a Windows shell, run WSL commands as `wsl -e bash -lc "<command>"`.
- The user's WSL sudo password is not known to them: **do not use sudo**. Everything needed is
  installed user-locally.

## Git conventions

- Commit on `aegis-dev`; push to `fork`. Never push to `origin` (upstream).
- WSL git has no user.name/email configured; pass them inline on commits
  (`git -c user.name="..." -c user.email="wade3337@gmail.com" commit ...`) or set them first.
- From Windows git over the UNC path, `waf` may show as modified; that is a file-mode
  artifact. Check `git status` from inside WSL before committing.
- Parameters: never reuse an `AP_GROUPINFO` index; keep the `@Param` doc block above each
  parameter (it generates docs and GCS display).

## Where things are in the code (ArduPlane/)

- Scheduler / main loop: `ArduPlane.cpp` (`scheduler_tasks` table). Every loop:
  ahrs_update → update_control_mode → stabilize → set_servos.
- Mode framework: `mode.h` (base class, `Mode::Number` enum, RTL = 11), `mode.cpp`
  (`Mode::run` = roll/pitch/yaw rate controllers), registry switch in `control_modes.cpp`.
- Mode change transaction: `Plane::set_mode` in `system.cpp`.
- RC → mode: `control_modes.cpp` (`mode_switch_changed`, FLTMODE1..6);
  dedicated switch via `RC_Channel.cpp` (`AUX_FUNC::RTL`).
- RTL: `mode_rtl.cpp`; destination from `do_RTL` in `commands_logic.cpp`
  (rally point or home); altitude from `get_RTL_altitude_cm` in `altitude.cpp`.
- Navigation → attitude → servos: `navigation.cpp`, `Attitude.cpp`
  (`calc_nav_roll`, `stabilize`, `calc_throttle`), `servos.cpp`.
- Parameters: `Parameters.cpp` / `Parameters.h` (old `g` GSCALAR table and newer
  `ParametersG2` group). Tuning knob: `tuning.cpp` (TUNE_PARAM / TUNE_CHAN / TUNE_RANGE).
- Simplest complete mode to copy as a template: `mode_fbwa.cpp`.
- Board definitions: `libraries/AP_HAL_ChibiOS/hwdef/<board>/hwdef.dat`.

## Goals and planned next steps

1. Understand the firmware (done: walkthrough of loop, modes, RTL, params, tuning).
2. Prove the edit→build loop with a small, safe change (e.g. a new parameter + GCS message).
3. Create an Aegis board definition (`board_id 7200`) modeled on SPRacingH7RF. Needs real
   pin-level details for the Aegis board; treat as hardware-risky, verify carefully.
4. Reconstruct FrSky's custom features (RTH point, gain knob) on top of upstream.
5. Longer term: request FrSky's GPLv3 corresponding source for the exact Aegis tree.

## Related material (on the Windows PC, outside this repo)

- The shipped "Aegis test build 26.10.22" zip was unpacked and analyzed; the firmware was
  decoded to a raw 938 KB ARM binary (base 0x90000000) for reference / Ghidra if needed.
- The ETHOS radio Lua tool for Aegis (separate zip) is plain, commented Lua source.
