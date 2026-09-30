# Guide: how this config works and how to change it

A walkthrough of every file in this repo, what it controls, and how to make the
common edits yourself. Aimed at the splitkb.com Aurora Sofle dongle setup here.

## How a ZMK build actually works

You do not compile firmware locally. GitHub Actions does it. The flow:

1. You push a change.
2. `.github/workflows/build.yml` calls ZMK's official build workflow.
3. That workflow reads `config/west.yml` to fetch ZMK plus any extra modules,
   then reads `build.yaml` to know which board+shield combinations to build.
4. For each combination it finds the matching keymap and config in `config/`,
   compiles a `.uf2`, and uploads it as a downloadable artifact.
5. You download the artifacts and flash each `.uf2` to the right board.

Key idea: a **board** is the microcontroller (here always `nice_nano_v2`). A
**shield** is the PCB/keyboard the controller plugs into (here the Aurora Sofle
halves and dongle). One build = one board + one or more shields.

## File map

```
.github/workflows/build.yml          Triggers the ZMK build on every push
.github/workflows/keymap-drawer.yaml Renders the keymap to an SVG on keymap changes
build.yaml                           Which firmwares to build (the build matrix)
config/west.yml                      Which ZMK version + external modules to fetch
config/splitkb_aurora_sofle.keymap   Your layers and key bindings
config/splitkb_aurora_sofle.conf     Feature switches (display, RGB, BLE, studio...)
config/config_keymap-drawer.yaml     Styling for the keymap SVG (cosmetic only)
keymap-drawer/                        The generated SVG + parsed YAML (auto-updated)
boards/shields/splitkb_aurora_sofle/  The hardware definition (pins, matrix, OLED)
zephyr/module.yml                    Marks this repo as a Zephyr module (leave alone)
```

The two files you will edit most are the **keymap** and the **conf**. You rarely
touch the `boards/shields/` folder unless you change hardware (encoder, OLED,
trackpad).

## build.yaml (which firmwares get built)

Each `- board: ... shield: ...` block is one firmware. Ours:

| Block | What it produces | Flash to |
| --- | --- | --- |
| `splitkb_aurora_sofle_dongle_pro_micro dongle_display` | Central + 128x64 screen | dongle |
| `splitkb_aurora_sofle_left_peripheral` | Left half firmware | left half |
| `splitkb_aurora_sofle_right` | Right half firmware | right half |
| `settings_reset` | Wipes stored BLE bonds | either half when re-pairing |

Fields per block:
- `board`: the microcontroller. Always `nice_nano_v2` for you.
- `shield`: one or more shield names, space separated. Extra shields like
  `dongle_display` layer optional features onto the base shield.
- `cmake-args`: build-time overrides. `-DCONFIG_ZMK_KEYBOARD_NAME=\"...\"` sets
  the Bluetooth name (max 16 chars). `-DCONFIG_ZMK_STUDIO=y` turns on ZMK Studio.
- `artifact-name`: the download filename (no extension).
- `snippet`: a reusable config bundle. `studio-rpc-usb-uart` is required for
  ZMK Studio over USB.

**Why the halves are "peripheral":** in a dongle setup the dongle is the BLE
central and both halves are peripherals. That is why the left uses the
`_left_peripheral` shield instead of `_left`. The role is set in
`Kconfig.defconfig` (see below), not here.

### Common build.yaml edits
- **Rename a device as it shows over Bluetooth:** change the `\"Aurora Dongle\"`
  string. Only the central (dongle) advertises a name, so that is the only one
  that needs it.
- **Turn on USB debug logging** for a target: add `-DCONFIG_ZMK_USB_LOGGING=y` to
  its `cmake-args` (temporary; remove when done).

## config/west.yml (dependencies)

Lists what to download before building.
- `zmk` at revision `v0.3.0`: the firmware itself. This version must match the
  `@v0.3.0` pinned in `.github/workflows/build.yml`.
- `zmk-dongle-display` at `v0.3` (from englmaxi): provides the `dongle_display`
  shield and the 128x64 dongle status screen.

To add a module later (for example a Cirque trackpad driver or a widget), add a
`remote` (the GitHub org) and a `project` (the repo + revision) here, then
reference its shield/snippet in `build.yaml`. Adding a module is the moment to
double check versions line up with your ZMK version.

## config/splitkb_aurora_sofle.keymap (your layout)

This one file is shared by all three keyboard firmwares. ZMK matches it to every
`splitkb_aurora_sofle*` shield by name, so the halves and dongle all use the same
layers.

### Structure

Top of file:
- `#include` lines pull in the key names (`&kp A`), Bluetooth actions (`&bt`),
  RGB actions (`&rgb_ug`), etc. Add an include if you use a behavior it defines.
- `#define BASE 0` ... `#define L6 6`: friendly names for layer numbers so you
  can write `&mo SEL` instead of `&mo 1`. If you add a layer, add a `#define`.
- `&led_strip { chain-length = <29>; }`: how many LEDs are in the underglow
  chain. 29 = per-key LEDs only. This is a per-build hardware setting kept in the
  keymap for convenience.

Then `keymap { ... }` contains one block per layer. Each block has:
- `display-name = "BASE";`: the 3-4 char label shown on the OLED.
- `bindings = < ... >;`: exactly **60 keys**, in physical order (see grid below).
- `sensor-bindings = < ... >;`: what the encoder(s) do on this layer.

### The 60-key grid

The bindings are read left to right, top to bottom, across BOTH halves:

```
row 0:  6 left  +  6 right              = 12
row 1:  6 left  +  6 right              = 12
row 2:  6 left  +  6 right              = 12
row 3:  6 left + 2 thumb-corner + 6 right = 14
row 4:      5 left thumb + 5 right thumb  = 10
                                   total = 60
```

If a layer has anything other than 60 bindings the build fails. The `bindings`
whitespace is only for readability. To sanity-check counts:

```
awk '/bindings = </{c=0;g=1;next} g&&/>;/{print n": "c;g=0} g{for(i=1;i<=NF;i++)if($i~/^&/)c++} /_layer \{/{n=$1}' config/splitkb_aurora_sofle.keymap
```

### How the layers here are wired
- **BASE (0):** QWERTY. Left inner thumb `&mo SEL` opens the selector while held.
  Right inner thumb `&mo L2` opens L2 while held.
- **SEL (1):** number row is `&tog SEL .. &tog L6`. `&tog N` flips a layer on and
  leaves it on until pressed again. Home row selects Bluetooth profiles, and
  there are F-keys and a few RGB controls.
- **L2-L6 (2-6):** all `&trans` (transparent = "fall through to the layer
  below"), except `&to BASE` top-left to jump home. Fill these in.

### Behaviors you will use most
- `&kp KEY` tap a key. Names live in the included `keys.h` (for example `&kp A`,
  `&kp LC(C)` for Ctrl+C, `&kp LS(N1)` for `!`).
- `&mo N` momentary: layer N active only while held.
- `&tog N` toggle: press to flip layer N on/off.
- `&to N` switch to layer N and turn every other non-base layer off.
- `&trans` transparent: use the binding from a lower active layer.
- `&none` do nothing (blocks fall-through).
- `&bt BT_SEL 0..4`, `&bt BT_CLR` Bluetooth profile select / clear.
- `&mt MOD KEY` mod-tap: hold = modifier, tap = key. `&lt N KEY` layer-tap.
- `&rgb_ug RGB_TOG` / `RGB_BRI` / `RGB_EFF` underglow control.

### Recipe: add a real layer (going past L6)
1. Add `#define L7 7` near the other defines.
2. Copy an `l6_layer { ... }` block, rename it `l7_layer`, set `display-name`.
3. Give it 60 bindings.
4. Add a way in: for example change a `&trans` on SEL's number row to `&tog L7`.

### Recipe: change what the encoder does
`sensor-bindings` has two entries because the shield declares two encoder
positions (left, right). You only have the right encoder, so the second entry is
the one that fires. Example to make it scroll instead of volume on a layer:
`sensor-bindings = <&inc_dec_kp PG_UP PG_DN &inc_dec_kp PG_UP PG_DN>;`

## config/splitkb_aurora_sofle.conf (feature switches)

Kconfig flags, one per line, `CONFIG_...=y` (on) or `=n` (off). Shared by all
three firmwares. The important ones here:
- `CONFIG_ZMK_DISPLAY=y` turn the OLEDs on.
- (no `..._STATUS_SCREEN_CUSTOM`) so the halves use ZMK's built-in status screen,
  which fits the 128x32 OLEDs. The dongle's 128x64 custom screen is supplied by
  the `dongle_display` module instead.
- `CONFIG_ZMK_RGB_UNDERGLOW=y` + the `..._RGB_UNDERGLOW_*` lines: underglow on,
  plus its default effect/brightness/hue.
- `CONFIG_ZMK_SLEEP=y` deep sleep to save battery.
- `CONFIG_ZMK_STUDIO=y` allow live keymap editing via ZMK Studio.
- `CONFIG_ZMK_MOUSE=y` mouse-key emulation (also the base for a future trackpad).
- `CONFIG_EC11=y` encoder support.
- `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` stronger Bluetooth transmit power.

Lines starting with `#` are off/notes. To flip a feature, uncomment or add the
line. A `_START/_START` value like `CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=15` is the
default brightness at boot (percent).

The dongle also has its own extra conf,
`boards/shields/splitkb_aurora_sofle/splitkb_aurora_sofle_dongle_pro_micro.conf`,
which sets central-specific things (peripheral count, more BLE connection slots,
its display). Half-only vs dongle-only settings belong there, shared settings
belong in `config/`.

## boards/shields/splitkb_aurora_sofle/ (the hardware)

You touch this only for physical changes. What each file is:
- `Kconfig.shield`: declares the shield names that exist (left, right,
  left_peripheral, dongle_pro_micro).
- `Kconfig.defconfig`: sets defaults per shield. This is where **who is the BLE
  central** is decided: the dongle and the `_left` shield are marked central; the
  peripherals are not. It also wires up display and RGB defaults.
- `splitkb_aurora_sofle.dtsi`: shared hardware description. Defines the key matrix
  transform, the two encoders (both `disabled` by default), and the **128x32
  OLED** on the halves. Change the OLED `width`/`height`/`multiplex-ratio` here if
  you ever swap the half screens.
- `splitkb_aurora_sofle_left.overlay`: left half as a standalone central (no
  dongle). Defines the left key matrix pins and enables `&left_encoder`.
- `splitkb_aurora_sofle_left_peripheral.overlay`: left half as a peripheral (what
  you build). Same pins; the left encoder enable was removed because you do not
  have a left encoder.
- `splitkb_aurora_sofle_right.overlay`: right half pins, enables `&right_encoder`,
  and shifts its columns with `col-offset = <6>` so both halves map into one
  matrix.
- `splitkb_aurora_sofle_dongle_pro_micro.overlay`: the dongle. Uses a mock kscan
  (it has no keys), defines the full matrix transform, and the **128x64 OLED**.
- `boards/nice_nano_v2.overlay`: the WS2812 underglow LED strip wiring for the
  nice!nano v2.
- `splitkb_aurora_sofle.zmk.yml`: metadata listing the shield and its siblings.

### Recipe: add the left encoder later
1. In `splitkb_aurora_sofle_left_peripheral.overlay`, add back at the end:
   `&left_encoder { status = "okay"; };`
2. The left encoder is already index 0 in `sensor-bindings`, so it will start
   working with the existing bindings.

### Recipe (future): add a Cirque trackpad
The full template is already in the repo, commented out, in three places. To
turn it on, uncomment all three together, then rebuild and flash the half:
1. `config/west.yml`: the `cirque-input-module` remote + project.
2. `boards/shields/splitkb_aurora_sofle/splitkb_aurora_sofle_left_peripheral.overlay`:
   the trackpad node (I2C Option A or SPI Option B) and the `zmk,input-listener`.
   If your trackpad is on the right half, move that block into
   `splitkb_aurora_sofle_right.overlay` instead.
3. `config/splitkb_aurora_sofle.conf`: the `CONFIG_ZMK_POINTING` block.

Before it will build you must replace the placeholder pins (`pro_micro N`,
`NRF_PSEL(..., X)`) with the GPIOs you actually solder to. I2C (Option A) reuses
the existing OLED bus and only needs the extra data-ready (DR) pad. On ZMK v0.3+
pointing is `CONFIG_ZMK_POINTING=y` (it supersedes `CONFIG_ZMK_MOUSE`). Driver:
https://github.com/petejohanson/cirque-input-module

Do this as its own change so you can flash and test just that half.

## Flashing

1. Open the GitHub Actions run, download the artifacts.
2. Put a board in bootloader mode (double-tap reset). It mounts as a USB drive.
3. Drag the matching `.uf2` onto the drive. It reboots automatically.
4. First time or after changing BLE settings: flash `settings_reset` to each
   half first, then flash the real firmware, then power all three on together to
   pair.

## Gotchas

- Every layer must have exactly 60 bindings or the build fails with a matrix
  error. Count them.
- `config/west.yml` ZMK version and the `@v0.3.0` in `build.yml` must match.
- Only the central advertises a Bluetooth name; naming a peripheral does nothing.
- After editing the keymap, the keymap-drawer workflow regenerates the SVG and
  commits it back. That extra commit is expected.
- Changing `chain-length` wrong will make RGB glitch; 29 matches the per-key LED
  count on this board.
