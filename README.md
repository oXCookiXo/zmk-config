# zmk-config: splitkb.com Aurora Sofle (dongle)

ZMK firmware for a splitkb.com Aurora Sofle v2 running in a dongle setup.

## Hardware

- **Dongle**: nice!nano v2, BLE central, 128x64 OLED
- **Left half**: nice!nano v2, peripheral, 128x32 OLED
- **Right half**: nice!nano v2, peripheral, 128x32 OLED, one EC11 encoder
- RGB underglow, ZMK Studio, and mouse emulation enabled

## Build

GitHub Actions builds every target in [`build.yaml`](build.yaml) on push. Download
the artifacts from the workflow run and flash each UF2 to the matching board:

| Artifact | Flash to |
| --- | --- |
| `aurora_sofle_dongle` | dongle |
| `aurora_sofle_left` | left half |
| `aurora_sofle_right` | right half |
| `aurora_sofle_settings_reset` | either half, to clear BLE bonds when re-pairing |

## Layers

Seven layers, defined in [`config/splitkb_aurora_sofle.keymap`](config/splitkb_aurora_sofle.keymap):

- **BASE** (0): QWERTY
- **SEL** (1): held from the left inner thumb; its number row toggles any layer
  on or off, home row selects BT profiles
- **L2 - L6** (2-6): toggle-layer placeholders, currently transparent

Every non-base layer has `&to BASE` on its top-left key to jump back home.

## Modules

- [zmk](https://github.com/zmkfirmware/zmk) `v0.3.0`
- [zmk-dongle-display](https://github.com/englmaxi/zmk-dongle-display) `v0.3` (128x64 dongle status screen)

## Keymap drawing

[`.github/workflows/keymap-drawer.yaml`](.github/workflows/keymap-drawer.yaml)
renders the keymap to `keymap-drawer/splitkb_aurora_sofle.svg` on each keymap change.
