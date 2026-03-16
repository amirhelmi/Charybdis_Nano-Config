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
- `config/west.yml` — West manifest (ZMK v0.3 + badjeff PMW3610 module)
- `config/charybdis.conf` — Shared config (both halves)
- `config/charybdis_left.conf` — Left half config
- `config/charybdis_right.conf` — Right half config (trackball driver enabled here)
- `config/charybdis.keymap` — Full keymap with 8 layers, combos, macros
- `boards/shields/charybdis/charybdis.dtsi` — Base devicetree (matrix, battery)
- `boards/shields/charybdis/charybdis_3610.dtsi` — PMW3610 SPI + sensor config
- `boards/shields/charybdis/charybdis_right.overlay` — Right half overlay (enables trackball)
- `boards/shields/charybdis/charybdis_left.overlay` — Left half overlay
- `boards/shields/charybdis/split_input_common.dtsi` — Input processors (snipe/scroll layers)

## PMW3610 Trackball — badjeff out-of-tree driver
**IMPORTANT**: The Zephyr upstream PMW3610 driver (`pixart,pmw3610`) crashes BLE after 10-30 min on this board.
Uses **badjeff/zmk-pmw3610-driver** instead (added in west.yml).

### Required Kconfig (in charybdis_right.conf)
```
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_PMW3610_ALT=y
CONFIG_ZMK_POINTING=y
```

### Devicetree Properties (badjeff driver)
- `compatible = "pixart,pmw3610-alt"` — badjeff compatible string (avoids upstream conflict)
- `irq-gpios` — interrupt pin (NOT `motion-gpios`, that's for Zephyr upstream)
- `cpi = <400>` — resolution (lowered from 600 to reduce BLE event pressure)
- `evt-type = <INPUT_EV_REL>`
- `x-input-code = <INPUT_REL_X>`
- `y-input-code = <INPUT_REL_Y>`

### Driver Comparison (DO NOT use Zephyr upstream)
| Property | Zephyr upstream (CRASHES) | badjeff (STABLE) |
|---|---|---|
| compatible | `pixart,pmw3610` | `pixart,pmw3610-alt` |
| interrupt | `motion-gpios` | `irq-gpios` |
| axis config | `zephyr,axis-x/y` | `evt-type`, `x/y-input-code` |
| Kconfig | `INPUT_PMW3610` (auto) | `PMW3610_ALT` (explicit) |
| west.yml | none needed | `badjeff/zmk-pmw3610-driver` |

### SPI Pin Assignments (charybdis_3610.dtsi)
| Function | nRF GPIO | Pro Micro Pin |
|----------|----------|---------------|
| SCK      | P0.08    | 0             |
| MOSI     | P0.17    | 2             |
| MISO     | P0.17    | 2 (same as MOSI — 3-wire SPI) |
| CS       | P0.20    | 3             |
| IRQ      | P0.06    | 1             |

### Critical Pin Conflict Notes
- PMW3610 uses **3-wire SPI** (SDIO): MISO MUST equal MOSI (both P0.17)
- CS must NOT be on P0.29 — that's Pro Micro 19, used by matrix column 0!
- IRQ must NOT be on P0.24 — that's Pro Micro 5, used by matrix row 1!
- Nice!Nano Pro Micro pin mapping: 0=P0.08, 1=P0.06, 2=P0.17, 3=P0.20, 4=P0.22, 5=P0.24, 6=P1.00, 7=P0.11, 8=P1.04, 9=P1.06, 10=P0.09, 18=P0.02, 19=P0.29, 20=P0.31

### Input Transform Notes (badjeff driver)
- **Main trackball** (charybdis_right.overlay): `X_INVERT | Y_INVERT | XY_SWAP`
- **Scroll layer** (split_input_common.dtsi): needs `Y_INVERT` before scalers
- **Snipe layer**: no transform changes needed
- badjeff driver reports axes differently from Zephyr upstream — when switching drivers, ALL transforms need revisiting
- Clockwise circle should draw clockwise; if anticlockwise, one axis inversion is wrong

## BLE Stability Notes
- `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y` — REMOVED, conflicts with Zephyr 4.1 BLE stack
- `CONFIG_ZMK_SLEEP=n` — sleep disabled to prevent disconnects
- Stack sizes increased for BLE + PMW3610: `SYSTEM_WORKQUEUE_STACK_SIZE=8192`, `MAIN_STACK_SIZE=8192` (4096 was too small, caused crashes)
- BLE connection intervals relaxed: `MIN_INT=12` (15ms), `MAX_INT=24` (30ms) — aggressive intervals (6/12) overwhelm BLE with trackball data
- BLE ACL buffers increased: `TX_SIZE=69`, `TX_COUNT=8`, `RX_SIZE=69`, `RX_COUNT_EXTRA=5` — needed for bursty trackball events (RX_COUNT deprecated in Zephyr 4.1, replaced by _EXTRA)
- CPI lowered to 400 (from 600) to reduce trackball event rate over BLE
- Bond reset procedure: flash settings_reset to BOTH halves, then reflash normal firmware, delete from PC Bluetooth, re-pair
- **Both** the upstream Zephyr driver AND insufficient BLE tuning can cause crashes — switching to badjeff driver alone is not enough without the buffer/stack/interval fixes

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
- West manifest tracks ZMK `v0.3` tag (pinned) + badjeff PMW3610 module
