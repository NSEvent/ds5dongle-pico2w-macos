# DS5Dongle — Pico 2 W build with a working macOS speaker

A ready-to-flash **Raspberry Pi Pico 2 W (RP2350)** firmware build of
[awalol/DS5Dongle](https://github.com/awalol/DS5Dongle), plus the one-time config
that makes the **DualSense built-in speaker work as a macOS audio output** through
the dongle.

> **This is not a code fork.** The firmware source is upstream `master`,
> unmodified. The "fix" is upstream's existing **`lock_volume`** config field
> (which isn't in the stable `v0.7.2-hotfix` release) combined with a build of
> `master`. All firmware credit goes to
> [awalol/DS5Dongle](https://github.com/awalol/DS5Dongle) (MIT). This repo only
> provides a prebuilt Pico 2 W `.uf2` and the macOS setup steps so you don't have
> to build it yourself.

## The problem it solves

DS5Dongle turns a DualSense into a wired-USB controller. On the stable
`v0.7.2-hotfix` release the **controller speaker is silent on macOS**. The dongle
sets the controller's hardware speaker level only from the *host's* USB-Audio
volume command — but macOS does software volume and **never sends that command**,
so the speaker stays at level 0. Writing `speaker_volume` with the config tool
doesn't stick either: the controller reports its own volume (`0`) back and
overwrites the config on every state update. The symptom is **a split-second of
sound when the controller connects, then silence**.

(The microphone works fine — input doesn't go through this path.)

## The fix

Upstream `master` added a **`lock_volume`** config field that tells the firmware
to *ignore* the controller's volume-report overwrite. Build `master`, flash it,
then set `lock_volume=1` plus a non-zero `speaker_volume`, and the level holds —
audible speaker output.

## Flash the prebuilt firmware

1. Download **`ds5-bridge-pico2w.uf2`** from the [latest release](../../releases/latest).
2. Put the Pico 2 W into BOOTSEL — triple-click the BOOTSEL button on a running
   dongle, or hold BOOTSEL while plugging in USB. It mounts as the `RP2350` volume.
3. Drag the `.uf2` onto `RP2350`; it reboots into the new firmware.
4. Reconnect the DualSense: hold **PS + Create** until the light bar flashes, then
   **short-press BOOTSEL** to scan.
5. Lock and raise the volume (use the `config_tool.py` in this repo — the one
   bundled with the stable release predates `lock_volume`):
   ```sh
   pip install hidapi
   python config_tool.py set lock_volume=1 speaker_volume=100 headset_volume=100
   ```
   The config tool only talks to the device while a DualSense is connected through
   the Pico.
6. Select **DualSense Wireless Controller** as your macOS output and raise the
   system volume. The speaker plays.

To roll back, just re-flash the stable `ds5-bridge-<version>.uf2` from
[upstream releases](https://github.com/awalol/DS5Dongle/releases) (you lose the
speaker fix).

## Build it yourself

This binary was built from awalol/DS5Dongle `master` @
[`fa83c77`](https://github.com/awalol/DS5Dongle/commit/fa83c77fbd30440542719824780cc5d20edd5608),
`wake` variant, on macOS (Apple Silicon) with the Arm GNU Toolchain 14.2.rel1:

```sh
git clone --recurse-submodules https://github.com/awalol/DS5Dongle.git
cd DS5Dongle
export PATH="$HOME/toolchains/arm-gnu-toolchain-14.2.rel1-darwin-arm64-arm-none-eabi/bin:$PATH"
tools/build-macos.sh        # clones Pico SDK 2.2.0, pins TinyUSB 0.20.0, builds
# -> build/wake/ds5-bridge.uf2
```

**macOS toolchain note:** Homebrew's `arm-none-eabi-gcc` *formula* is built
without newlib headers (the build rejects it), and the `gcc-arm-embedded` *cask*
needs `sudo`. The simplest no-sudo path is the official Arm GNU Toolchain tarball
from [developer.arm.com](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads)
(`darwin-arm64-arm-none-eabi`), extracted to a user directory and put on `PATH`.

## Related

[**ControllerKeys**](https://github.com/NSEvent/xbox-controller-mapper) — a macOS
controller-to-keyboard/mouse/macro remapper. It pairs naturally with this build:
run a DualSense through the dongle as wired USB and ControllerKeys maps its
buttons, touchpad, and gyro like any wired DualSense.

## Credit & license

Firmware and `config_tool.py` © awalol, **MIT-licensed** — see [`LICENSE`](LICENSE)
and the upstream project: <https://github.com/awalol/DS5Dongle>. This repository
redistributes an unmodified `master` build under the same MIT license; it adds
only this README and a prebuilt binary. If the speaker behavior changes upstream
(e.g. a release ships `lock_volume`), prefer the official build.
