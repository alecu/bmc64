# Pimmodore support for BMC64

This fork of [randyrossi/bmc64](https://github.com/randyrossi/bmc64) brings
native support for the **Pimmodore** setup to current BMC64. The goal is to
upstream it, so the fork can eventually go away.

## Background

A *Pimmodore* is a Raspberry Pi running BMC64 inside a real Commodore case,
with the original keyboard driven by a [Keymmodore](https://www.keymmodore.com/)
controller (K-rnivoro). Keymmodore turns the Commodore keyboard matrix into a
USB keyboard and has several keyboard modes (C64, C128, PET business, PET
graphics, plain PC). Out of the box the user has to switch the mode by hand
whenever they switch machines in BMC64.

Sérgio Lagoela ([slagoela/bmc64](https://github.com/slagoela/bmc64), 2022–2026)
closed that gap: BMC64 drives three GPIO pins with a code for the machine being
emulated, and a [modified Keymmodore firmware](https://github.com/slagoela/Keymmodore-64/tree/Teensy++2.0)
on a Teensy++ 2.0 reads them and switches mode by itself. That fork is now
451 commits behind upstream, so this repo ports the work to current BMC64.

### Protocol (as implemented by slagoela)

Enabled with `enable_pimmodore=true` (or `=1`) in `cmdline.txt`. Not available
when DPI video is on, because DPI uses the GPIO header.

| Pi GPIO | Signal       | Teensy++ 2.0 |
|---------|--------------|--------------|
| GPIO 9  | PIMMODORE_P0 | B4           |
| GPIO 10 | PIMMODORE_P1 | B5           |
| GPIO 11 | PIMMODORE_P2 | B6           |

`P2 P1 P0` is a 3-bit mode number:

| Value | Mode         | Used for                                      |
|-------|--------------|-----------------------------------------------|
| 0     | PC           | fallback                                      |
| 1     | C64          | C64, C64SC, VIC-20, Plus/4, Plus4Emu, C64DTV  |
| 2     | C128         | C128                                          |
| 3     | PET business | PET models with a business keyboard           |
| 4     | PET graphics | PET models with a graphics keyboard           |

The pins are set at boot, whenever the GPIO configuration is reset, and when
the PET model is changed from the menu. GPIO 9 and 10 are "no fixed purpose"
pins in every built-in GPIO configuration. GPIO 11 is new. When the option is
on, all three pins show as *Reserved for Pimmodore* in the custom GPIO menu.

## Plan

### 1. Fork and set up the repository ✅

- [x] Fork `randyrossi/bmc64` to `alecu/bmc64`.
- [x] Branch layout:
  - `master`: an exact mirror of `randyrossi/master`. Never commit here, so
    PR branches always start from upstream.
  - `pimmodore`: the integration branch (default branch of the fork). It holds
    this file, the fork-only settings below, and the merged `pr/*` branches.
  - `pr/*`: one branch per upstream PR, each based on `master`.
- [ ] Keep up to date by merging `master` into `pimmodore` and rebasing the
  `pr/*` branches.

### 2. Port slagoela's changes ✅

Done on two branches. Upstream moved `kernel.cpp`, `viceapp.*` and
`viceoptions.*` under `src/`, and now has an emulator-independent
`emux_machine_class`, which made the port simpler than the original.

| Change in slagoela/bmc64 | What happened |
|---|---|
| `enable_pimmodore` option, GPIO 11 as `NO_FIXED_PURPOSE_5` (`NUM_GPIO_PINS` 23 → 24), mode output, skipping the pins in custom GPIO reads and resets, menu items | Ported in `pr/pimmodore-gpio`. Old settings files stay valid (`custom_gpio=` is saved as `index,value` pairs and bounds-checked). The saved bindings for the three pins are kept instead of being cleared. |
| `SetupPimmodoreGPIO()` writing the pins on GPIO reset and PET model change | Redone: the kernel checks `emu_get_pimmodore_mode()` once per frame and writes the pins only when the mode changes. Covers boot, PET model changes and keyboard changes without hooks in the PET menu. Plus4Emu is handled through `emux_machine_class`, so the original's `#if RASPI_PLUS4EMU` branch (with its missing `;` after `return`) is gone. |
| PET business vs. graphics: `petkeyboard_get()` and the `machine_get_keyboard_type()` fallback | Dropped. `KeyboardType` is always registered upstream and `petmodel_set()` updates it, so the fallback never runs. The mode reads it via a new `Setting_KeyboardType`. |
| `SetupUserport()` only reapplies pin modes when the DDR changes | Ported in `pr/userport-ddr-cache`, with a fix: the original cache was never reset, so switching GPIO config away from the user port and back left output pins as inputs. |
| README section with wiring | General part in `docs/USER_GUIDE.md` ("Keymmodore Keyboard Mode Outputs"). Teensy wiring stays in this file. |
| `enable_pimmodore=1` in `sdcard/cmdline.txt` | Fork only, on `pimmodore`. Machine switching keeps options it doesn't know, so it survives. |
| `gpu_mem=64` → `128` "for PET graphics" | Not ported. Upstream now raises `gpu_mem` per profile in `machines.txt` where needed. Revisit only if the hardware test shows a PET display problem. |
| Prebuilt `kernel*.zip` in the repo | Dropped; releases come from CI. |
| Build fixes for GCC/WSL/autotools | Dropped; upstream builds with its own GCC 15.2.1. |

### 3. Build and release in GitHub Actions

- [x] `.github/workflows/pimmodore-build.yml` (fork only): builds pi3 with all
  six machines on every push to `pimmodore`, fails unless all six kernels
  exist (upstream's `make_all.sh` exits 0 even when the build fails), and
  uploads them as an artifact.
- [x] Fork-only release settings on `pimmodore`:
  - `VERSION_STRING` is `5.2.6-pimmodore1` (`third_party/common/menu.c`).
    Bump the number for each test release.
  - `updater = off` in `updater.cfg`. Otherwise the Web UI would offer
    upstream releases, and installing one removes Pimmodore support.
    Setting it up for the fork's own releases is described in
    `docs/BUILDING.md` ("For forks") if it's ever needed.
- [ ] Release: push tag `v5.2.6-pimmodore1`. Upstream's `c-cpp.yml` checks the
  tag against `VERSION_STRING`, builds pi3, pi2 and pi0, and creates a
  **draft** release with `bmc64-v5.2.6-pimmodore1.files.zip`. Publish it as a
  pre-release so it can be shared.
- [ ] Hardware test (Mauricio, Pimmodore):
  - [ ] Unzip the release to a FAT32 SD card; `cmdline.txt` already has
    `enable_pimmodore=true`.
  - [ ] Boot each machine (Switch Machine menu) and check the Keymmodore mode:
    C64 → 1, VIC20 → 1, Plus/4 and Plus4Emu → 1, C128 → 2.
  - [ ] PET: switch the model in the menu between a business model (e.g.
    8032) and a graphics model (e.g. 2001, 4032). The mode should follow
    (3 and 4) without a reboot.
  - [ ] Typing works in each mode, including C128 extra keys and PET layouts.
  - [ ] Custom GPIO menu shows GPIO 09, 10 and 11 as "Used by Pimmodore".
  - [ ] Joysticks or other GPIO devices in use still work.
  - [ ] With `enable_pimmodore` removed from `cmdline.txt`, GPIO 9/10/11 are
    inputs again (Keymmodore no longer switches) and the custom GPIO menu can
    bind them.

### 4. Upstream PRs

Each based on `master`. Upstream's `docs/SCOPE.md` matters here: third-party
hardware support is "best effort and community driven", and contributions
that add support are welcome. The project doesn't support community hardware
itself, so the PR should be small, opt-in and self-contained.

1. **`pr/userport-ddr-cache`: Only reconfigure user port pins when the DDR
   changes.** A standalone bug fix: Circle's `SetMode()` drives output pins
   LOW, so today every user port output glitches low once per frame.
2. **`pr/pimmodore-gpio`: Optional Keymmodore keyboard mode outputs.** The
   feature. Touches the same lines in `kernel.cpp`/`kernel.h` as PR 1 (member
   initialisers), so whichever lands second needs a trivial rebase.
3. *(After the hardware test)* **Community Projects entries** for Keymmodore
   and Pimmodore in `docs/COMMUNITY_PROJECTS.md`, with their authors' consent
   and wording.
4. *(Maybe)* **CI build on push/PR** for upstream, based on
   `pimmodore-build.yml`, and making `make_all.sh` exit non-zero on failure.

**Why BMC64 should support this natively** (for the PR 2 description):
- Real Commodore cases with the original keyboard are among BMC64's main use
  cases. It already supports the Keyrah, TheC64 Maxi case, and a real keyboard
  over GPIO. Keymmodore connects a real keyboard over USB and handles C128 and
  PET layouts.
- Without this, the user has to change the keyboard mode by hand every time
  they switch machines or PET models, which breaks the "power on and use it
  like the real thing" experience BMC64 aims for.
- It's opt-in, uses three pins that no built-in configuration uses, is
  disabled together with DPI, and changes nothing when off.
- The interface is just three digital output lines, so any other keyboard
  controller can use it too.
- Maintained in a fork it falls behind (the original fork ended up 451
  commits behind); in upstream it stays current.

Before opening PR 2, ask the maintainers (GitHub issue or Discord) whether
they prefer the `pimmodore` name or a generic one (e.g. `keyboard_mode_gpio`).

## Open questions

- Is the Plus/4 really meant to use the C64 keyboard mode on a Keymmodore?
- Does the Keymmodore firmware need the Teensy++ 2.0 build from
  slagoela/Keymmodore-64, or does current Keymmodore firmware read these pins?
