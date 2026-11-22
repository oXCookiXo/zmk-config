# zmk-config: splitkb.com Aurora Sofle

ZMK firmware for a splitkb.com Aurora Sofle v2 running as two halves, with the
left half as the central. A 3-controller dongle setup is staged in
[`build.yaml`](build.yaml) but commented out until the dongle case is printed.

New here or making manual edits? See [GUIDE.md](GUIDE.md) for what every file
does and step-by-step recipes for common changes.

## Hardware

- **Left half**: nice!nano v2, BLE central, 128x32 OLED, one EC11 encoder
- **Right half**: nice!nano v2, peripheral, 128x32 OLED, one EC11 encoder
- RGB underglow, ZMK Studio, and mouse emulation enabled

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

- **BASE** (0): QWERTY
- **SEL** (1): held from the left inner thumb; its number row toggles any layer
  on or off, home row selects BT profiles
- **L2 - L6** (2-6): toggle-layer placeholders, currently transparent

Every non-base layer has `&to BASE` on its top-left key to jump back home.

## Modules

- [zmk](https://github.com/zmkfirmware/zmk) `v0.3.0`
- [zmk-dongle-display](https://github.com/englmaxi/zmk-dongle-display) `v0.3` (128x64 dongle status screen, only used by the dongle setup)

## Keymap drawing

[`.github/workflows/keymap-drawer.yaml`](.github/workflows/keymap-drawer.yaml)
renders the keymap to `keymap-drawer/splitkb_aurora_sofle.svg` on each keymap change.
