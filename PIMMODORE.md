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

### 1. Fork and set up the repository

- [ ] Fork `randyrossi/bmc64` to `alecu/bmc64`.
- [ ] Branch layout:
  - `master`: an exact mirror of `randyrossi/master`. Never commit here, so
    PR branches always start from upstream.
  - `pimmodore`: the integration branch (default branch of the fork). It holds
    this file, fork-only CI and release tweaks, and the merged feature
    branches.
  - `pr/*`: one branch per upstream PR, each based on `master`.
- [ ] Keep up to date by merging `master` into `pimmodore` and rebasing the
  `pr/*` branches.

### 2. Port slagoela's changes

What each change in the old fork becomes. Paths moved upstream: `kernel.cpp`,
`viceapp.*` and `viceoptions.*` now live under `src/`.

| Change in slagoela/bmc64 | Action |
|---|---|
| `enable_pimmodore` option (`viceoptions.*`), `circle_pimmodore_enabled()` (`kernel.*`, `circle.h`) | Port |
| GPIO 11 added as `NO_FIXED_PURPOSE_5`, `NUM_GPIO_PINS` 23 → 24 (`viceapp.*`, `circle.h`, `menu_gpio.c`) | Port. Check every array sized by `NUM_GPIO_PINS` and the bounds check where `custom_gpio=` settings are loaded (saved as `index,value` pairs, so old settings files stay valid). |
| `SetupPimmodoreGPIO()` and the mode table (`kernel.cpp`, `viceapp.h`) | Port. Fix the missing `;` after `return` in the `RASPI_PLUS4EMU` branch: as written, Plus4Emu sets P0 as an output and returns without writing the mode. |
| Skip Pimmodore pins in `ReadCustomGPIO()` / `circle_reset_gpio()` | Port |
| "Reserved for Pimmodore" items in the custom GPIO menu (`menu_gpio.c`) | Port |
| PET: tell business and graphics keyboards apart: `petkeyboard_get()`, `machine_get_keyboard_type()` falls back to the model default, reset GPIO on PET model change | Port |
| `SetupUserport()` only reapplies pin modes when the DDR changes | Port as its own change (it's independent of Pimmodore) |
| README section with wiring | Port, and move most of it into this file / docs |
| `sdcard/cmdline.txt` with `enable_pimmodore=1` by default | Fork only (`pimmodore` branch). Upstream default stays off. |
| `sdcard/config.txt` `gpu_mem=64` → `128` "for PET graphics" | Find out what actually breaks with 64 on current upstream before porting. Propose separately only if it's a real upstream bug. |
| Prebuilt `kernel*.zip` in the repo root | Drop. Releases come from CI instead. |
| Build fixes (PET compile, Plus4Emu link, `uint32_t` cookie casts, WSL `configure.proto`, `autogen.sh` in `clean_all.sh`) | Drop unless the current upstream build needs them. Upstream now uses its own GCC 15.2.1 toolchain and already builds in CI. |

Credit: keep Sérgio Lagoela as the author of the ported commits (or add
`Co-authored-by`), and let them know about the upstreaming effort.

### 3. Build and release in GitHub Actions

Upstream's `.github/workflows/c-cpp.yml` already builds pi3, pi2 and pi0 and
makes a draft release with `make_release.sh`. It only runs on a `v*` tag that
matches `VERSION_STRING` in `third_party/common/menu.c`.

- [ ] Add a fork-side workflow that runs on push and pull request, builds pi3
  (all six machines), and uploads the kernels as artifacts. This catches
  breakage early without making a release. Worth offering upstream as well.
- [ ] Releases from the fork: on `pimmodore`, set `VERSION_STRING` to e.g.
  `5.2.6 pimmodore1` and tag `v5.2.6-pimmodore1`, so the tag check in the
  existing workflow passes unchanged.
- [ ] A release is valid when:
  - [ ] the workflow finishes and `make_release.sh` finds all 18 kernels
    (6 machines × pi3/pi2/pi0);
  - [ ] the zip contains the kernels, `config.txt` and `cmdline.txt` with
    `enable_pimmodore=1`, and the usual SD card layout;
  - [ ] it boots on a real Pi 3 in a Pimmodore, and the Keymmodore follows when
    switching C64 → C128 → VIC-20 → PET business → PET graphics → Plus/4;
  - [ ] with `enable_pimmodore` off, GPIO 9/10/11 stay inputs, the custom GPIO
    menu is unchanged, and joysticks and keyboards on every built-in GPIO
    configuration still work.

### 4. Upstream PRs

Small, single-purpose PRs to `randyrossi/bmc64`, each based on `master`, in
this order:

1. **Only reconfigure userport pins when the DDR changes.**
   `SetupUserport()` currently resets the mode of all 8 pins on every call.
   Changing a pin's mode when nothing changed is wasted work and can glitch
   outputs. Independent of Pimmodore; easy to review and merge.
2. **PET: report the model's default keyboard type.**
   `machine_get_keyboard_type()` returns 0 when the `KeyboardType` resource
   isn't set, which loses the business/graphics difference. Add
   `petkeyboard_get()` and use it as the fallback. A small, correct-on-its-own
   fix that the next PR depends on.
3. **Optional machine-mode GPIO outputs for Keymmodore/Pimmodore keyboards.**
   The feature itself: the `enable_pimmodore` option, GPIO 11, mode output,
   menu reservation, PET model-change hook, README section. Off by default and
   no change at all unless enabled.
4. *(Maybe)* **CI build on push/PR**, if upstream wants it.
5. *(Maybe)* **`gpu_mem` change**, only with a reproducible PET graphics
   problem.

**Why BMC64 should support this natively** (for the PR 3 description):
- Real Commodore cases with the original keyboard are among BMC64's main use
  cases. It already supports the Keyrah, TheC64 Maxi case, and a real keyboard
  over GPIO. Keymmodore is a popular way to connect a real keyboard, and the
  one that handles C128 and PET layouts.
- Without this, the user has to change the keyboard mode by hand every time
  they switch machines, which breaks the "power on and use it like the real
  thing" experience BMC64 aims for.
- It's opt-in, uses three pins that no built-in configuration uses, is disabled
  together with DPI, and leaves default behaviour unchanged.
- The interface is just three digital output lines, so any other keyboard
  controller can use it too.
- Maintained in a fork it falls behind (the original fork is 451 commits
  behind); in upstream it keeps up for free.

Before opening PR 3, ask in a GitHub issue or discussion whether the
maintainers prefer the `pimmodore` name or a generic one (e.g.
`machine_gpio_out`), and whether they'd rather get one PR or the split above.

## Open questions

- Should GPIO 11 also be added as a normal bindable pin for custom GPIO users
  who don't use Pimmodore? (It is in slagoela's change, as a side effect.)
- Pi 4 builds: upstream has a `pi4` target. Does the GPIO code apply there too?
- Plus/4 is mapped to the C64 keyboard mode. Is that right for a Plus/4 or 16
  case with a Keymmodore?
