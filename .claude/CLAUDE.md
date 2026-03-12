# Charybdis Nano ZMK Configuration

## Hardware
- **Keyboard**: Charybdis Nano split keyboard (BastardKB)
- **MCU**: Nice!Nano v2 (nRF52840) on both halves
- **Trackball**: PMW3610 optical sensor on **right half** (central)
- **Matrix**: 10 columns x 4 rows, row2col diode direction

## Architecture
- Right half = **central** (connects to host PC)
- Left half = **peripheral** (connects to right half over BLE)
- Trackball data flows: PMW3610 → right half → host (or split bridge to left)

## Key File Paths
- `config/west.yml` — West manifest (ZMK main branch, no external modules)
- `config/charybdis.conf` — Shared config (both halves)
- `config/charybdis_left.conf` — Left half config
- `config/charybdis_right.conf` — Right half config (trackball driver enabled here)
- `config/charybdis.keymap` — Full keymap with 8 layers, combos, macros
- `boards/shields/charybdis/charybdis.dtsi` — Base devicetree (matrix, battery)
- `boards/shields/charybdis/charybdis_3610.dtsi` — PMW3610 SPI + sensor config
- `boards/shields/charybdis/charybdis_right.overlay` — Right half overlay (enables trackball)
- `boards/shields/charybdis/charybdis_left.overlay` — Left half overlay
- `boards/shields/charybdis/split_input_common.dtsi` — Input processors (snipe/scroll layers)

## PMW3610 Trackball (Zephyr Upstream Driver)
Uses the **Zephyr upstream** PMW3610 driver (NOT an out-of-tree ZMK module).

### Required Kconfig (in charybdis_right.conf)
```
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_INPUT_PMW3610=y   # auto-enabled via DT, but explicit is safer
CONFIG_ZMK_POINTING=y
```
- Zephyr upstream Kconfig symbol is `INPUT_PMW3610` (NOT `PMW3610`)
- It's `default y` when `DT_HAS_PIXART_PMW3610_ENABLED`, so it may auto-enable from devicetree
- Out-of-tree drivers (badjeff) use `CONFIG_PMW3610_ALT`, inorichi uses `CONFIG_PMW3610`

### Devicetree Properties (Zephyr upstream)
- `compatible = "pixart,pmw3610"` — Zephyr upstream compatible string
- `motion-gpios` — interrupt/motion pin (NOT `irq-gpios`, that's for out-of-tree drivers)
- `zephyr,axis-x = <INPUT_REL_X>` — required by Zephyr 4.1+
- `zephyr,axis-y = <INPUT_REL_Y>` — required by Zephyr 4.1+
- `res-cpi` — resolution in CPI
- `smart-mode` — smart motion detection

### SPI Pin Assignments (charybdis_3610.dtsi)
| Function | nRF GPIO | Pro Micro Pin |
|----------|----------|---------------|
| SCK      | 0,8      | 1             |
| MOSI     | 0,17     | 2             |
| MISO     | 0,20     | 3             |
| CS       | 0,29     | 14            |
| Motion   | 0,6      | 0             |

### Common Pitfall
- Do NOT add `irq-gpios` — that property is for out-of-tree PMW3610 drivers (badjeff/inorichi forks)
- The Zephyr upstream driver uses `motion-gpios` instead
- The Zephyr Kconfig symbol is `INPUT_PMW3610` — `CONFIG_PMW3610` does NOT exist and will cause a build error

## Layers
| Index | Name | Purpose |
|-------|------|---------|
| 0     | DEF  | Default |
| 1     | NUM  | Numbers |
| 2     | NAV  | Navigation (trackball → scroll) |
| 3     | SWT  | Switch |
| 4     | FN   | Function |
| 5     | SNI  | Snipe (trackball precision 1/3 speed) |
| 6     | SCR  | Scroll |
| 7     | BOOT | Bootloader |

## Build
- `build.yaml` targets: nice_nano/nrf52840/zmk with charybdis_left, charybdis_right, settings_reset
- Uses ZMK's official GitHub Actions workflow (`zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`)
- West manifest tracks ZMK `main` branch (Zephyr 4.1+)