# Trinity avionics — PCB projects

KiCad projects, production files and reference material for the Trinity
avionics stack. Two boards:

| Directory | Board |
|---|---|
| `PCB2.3_JF/` | Main board — RP2040, BMP388 barometer, pyro channel |
| `Sensor_board_1.2/` | Extension board — BNO085 breakout, SD card, LEDs, buzzer |
| `datasheets+components+min-design/` | Datasheets, component lists, RP2040 minimal reference design |

## Main board pin map (PCB2.3_JF)

**Real RP2040 GPIO numbers.** See the silkscreen warning below.

| Signal | GPIO | Notes |
|---|---|---|
| BMP388 SDA / SCL | 6 / 7 | I2C1 → `Wire1`, address `0x76`, pull-ups R23/R24 |
| BNO085 | 6 / 7 | same bus, address `0x4A` |
| LED (D5) | 20 | via R15 — **not** GPIO25 as on a Pico |
| Pyro ("palnik", J5) | 24 | via R18 to Q1 gate |
| ADC_0 / ADC_3 | 26 / 29 | to J4.7 / J4.6 |
| J4.4 / J4.5 | 18 / 19 | silkscreen correct here |

### ⚠ J3 silkscreen labels are shifted by two

The nets named `/GPIO2`…`/GPIO7` land on RP2040 **GPIO0–GPIO5**:

| J3 pin, silkscreen | Real GPIO |
|---|---|
| J3.1 "GPIO2" | GPIO0 |
| J3.3 "GPIO3" | GPIO1 |
| J3.5 "GPIO4" | GPIO2 |
| J3.7 "GPIO5" | GPIO3 |
| J3.9 "GPIO6" | GPIO4 |
| J3.11 "GPIO7" | GPIO5 |

J4's labels are correct. **Fix this on the next revision**, on the main and
extension board together — correcting only one recreates the same bug with a
new offset.

## RP2040 SPI pin rule

SPI function is fixed by GPIO number mod 4. This constrains any SD card wiring:

| n mod 4 | Role |
|---|---|
| 0 | MISO (RX) |
| 1 | CS |
| 2 | SCK |
| 3 | MOSI (TX) |

SPI0 = GPIO0–7 and 16–23. SPI1 = GPIO8–15 and 24–29.

GPIO6/7 are the I2C bus and must not be reused for SPI.

## Firmware build

```
arduino-cli compile -b rp2040:rp2040:generic:flash=16777216_12582912,freq=125 <sketch>
```

Flash is a **W25Q128 (16 MB)**, split 4 MB code / 12 MB filesystem.

`freq=125` is deliberate: arduino-pico defaults to 200 MHz, above the RP2040's
133 MHz datasheet maximum.

## Hardware status

**Verified on hardware**

- BMP388 barometer — 25 Hz logging, ±0.1 m noise
- BNO085 orientation — ±0.01 ° stable, two units tested
- Both sensors sharing `Wire1` simultaneously (0x76 + 0x4A)
- Battery power from a 2S pack

**Open issues**

- **SD card** — the tested module (GME 1508311) carries an AMS1117-3.3
  regulator needing ~1.1 V headroom, so it only works from 5 V. There is no
  5 V rail in flight. Replace with a 3.3 V module or put a bare microSD socket
  on the board.
- **No SWD pads** — pads 24/25 unconnected, so no hardware debugging. Add them.
- Pyro switch is an **AO3400A** (30 V V_DS, SOT-23). There is **no flyback
  diode** anywhere in the design — fine for a resistive igniter, not for
  anything inductive.

## For the next revision

1. Make silkscreen labels match real GPIO numbers (both boards together)
2. Add SWD pads
3. Resolve SD card power — bare socket at 3.3 V preferred
4. Consider **RP2350B** if sensor fusion is planned: the RP2040 is Cortex-M0+
   with no FPU, so all floating-point maths is software-emulated
