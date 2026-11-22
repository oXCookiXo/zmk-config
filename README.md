# zmk-config: splitkb.com Aurora Sofle

ZMK firmware for a splitkb.com Aurora Sofle v2 running as two halves, with the
left half as the central. A 3-controller dongle setup is staged in
[`build.yaml`](build.yaml) but commented out until the dongle case is printed.

New here or making manual edits? See [GUIDE.md](GUIDE.md) for what every file
does and step-by-step recipes for common changes.

## Hardware

- **Left half**: nice!nano v2, BLE central, 128x32 OLED
- **Right half**: nice!nano v2, peripheral, 128x32 OLED, one EC11 encoder
- RGB underglow and ZMK Studio enabled

## Build

GitHub Actions builds every target in [`build.yaml`](build.yaml) on push. Download
the artifacts from the workflow run and flash each UF2 to the matching board:

| Artifact | Flash to |
| --- | --- |
| `aurora_sofle_left` | left half |
| `aurora_sofle_right` | right half |
| `aurora_sofle_settings_reset` | either half, to clear BLE bonds when re-pairing |

ZMK Studio connects through the left half over USB.

## Switching to the dongle setup

1. In [`build.yaml`](build.yaml), comment out the `splitkb_aurora_sofle_left`
   entry and uncomment the dongle and `splitkb_aurora_sofle_left_peripheral` entries.
2. Flash `aurora_sofle_settings_reset` to all three boards to clear old bonds.
3. Flash `aurora_sofle_dongle` to the dongle, `aurora_sofle_left_peripheral` to
   the left half, and `aurora_sofle_right` to the right half.

The dongle becomes the central and drives a 128x64 OLED; ZMK Studio then
connects through the dongle.

## Layers

Seven layers, defined in [`config/splitkb_aurora_sofle.keymap`](config/splitkb_aurora_sofle.keymap):

- **BASE** (0): QWERTY. Number row taps a digit and double-taps its shifted
  symbol; Shift + Backspace sends Delete
- **SEL** (1): held from the right thumb; its number row toggles L2 - L6 on or
  off, plus F1 - F12, BT profiles on the home row, and RGB controls
- **L2** (2): navigation, held from the left inner thumb (tap for Enter). ESDF
  arrows, Insert/Home/End/Page Up/Page Down on the right hand
- **L3** (3): numpad on the right hand and right-hand letters mirrored to the
  left, held from the right inner thumb (tap for Enter)
- **L4** (4): F-keys across the number row, held from the left thumb (tap for Space)
- **L5** (5): left-hand letters mirrored to the right, held from the right thumb
  (tap for Space)
- **L6** (6): macros and editor shortcuts on the left hand, held from the left thumb

Every non-base layer has `&to BASE` on its top-left key to jump back home.

## Modules

- [zmk](https://github.com/zmkfirmware/zmk) `v0.3.0`
- [zmk-dongle-display](https://github.com/englmaxi/zmk-dongle-display) `v0.3` (128x64 dongle status screen, only used by the dongle setup)

## Keymap drawing

[`.github/workflows/keymap-drawer.yaml`](.github/workflows/keymap-drawer.yaml)
renders the keymap to `keymap-drawer/splitkb_aurora_sofle.svg` on each keymap change.
