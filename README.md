# fardriver-can-notes

Field notes for controlling a **Fardriver** motor controller over **CAN** and
**UART**: speed mode, forward / neutral / reverse, cruise, status frames, and
a map of the settings you can change without the PC tool.

Everything here was tested on a real electric motorbike controller on the
bench. Where something was not tested, it says so.

Not made by or with Fardriver. Use at your own risk: these settings make a
motor move. Test with the wheel off the ground.

## Tested hardware

| Item | Value |
|---|---|
| Controller | Fardriver NS30, model `ND72680B_24_3` |
| Firmware | `HA86` (most tests), `HA91`, `HB38` |
| Motor | 5 pole pairs, encoder |
| Battery | 72 V nominal |
| CAN | 250 kbit/s, 120 Ω at both ends |
| Our side | ESP32-S3 with a CAN transceiver (any CAN node works) |

## Contents

1. [Speed mode and gear over CAN (protocol 60)](#1-speed-mode-and-gear-over-can-protocol-60)
2. [Full control over CAN (protocol 61)](#2-full-control-over-can-protocol-61)
3. [Cruise](#3-cruise)
4. [Stock dashboards need the factory layout](#4-stock-dashboards-need-the-factory-layout)
5. [Throttle](#5-throttle)
6. [Using the PC tool](#6-using-the-pc-tool)
7. [UART parameter map](PARAMETERS.md)

---

## 1. Speed mode and gear over CAN (protocol 60)

Protocol 60 lets you choose the CAN layout yourself. With the settings
below, one frame (`0x100`) sets the speed mode (low / mid / high) and the
gear (N / F / R).

### Controller settings

| What | Where in the PC tool | Value |
|---|---|---|
| CAN protocol | `0x86`, low 6 bits | **60** (configurable layout) |
| CAN bit rate | `0xBC` | 250 kbit |
| Speed mode source | Function → Special → **HighLow** | **CAN3Speed** (13) |
| Direction default | Function → Special → **Gear** | **DefaultN** (0) |

- **Gear = DefaultN is the key setting.** With the factory `DefaultFW` the
  controller always drives forward, and only the reverse wire changes it.
  The CAN gear field is then ignored.
- HighLow = 14 is rejected by the controller. 13 is right.
- **Do not use protocol 59.** It held the bus dominant until a power cycle.

### CAN layout (PC tool → CAN Parameters dialog)

Open it with the big red down arrow at the top of the "Parameters File Name"
panel. Press **Read** first; it opens showing zeros. `Pos` is a **bit**
offset (16 = byte 2).

**CAN Control**

| Field | Value |
|---|---|
| DataFormat | INTEL |
| Rcv / Send frame type | Standard |
| **SpeedLimitID** (speed mode) | **0x100** |
| **GearID** (N / F / R) | **0x100** |
| Every other ID | 0x0 |
| ControlType | 0 |
| **LimitSpeed BytePos / BitPos** | **1 / 0** |
| **Gear BytePos / BitPos** | **1 / 2** |

Never leave a used field at BytePos 0 + BitPos 0. The manual treats that as
a special value.

**CAN SendID** — period = (Timer + 1) × 10 ms

| Field | Value |
|---|---|
| SendID0 / Timer | `0x101` / 4 (50 ms) |
| SendID1 / Timer | `0x102` / 4 (50 ms) |
| SendID2–5 | 0x0 / 0 |

Timers below 4 on SendID2–5 mean "send only on request".

**CAN Data** on `0x101` (`Length, Pos, Gain, ID, Valid, Offset`)

| Item | Cells | Meaning |
|---|---|---|
| Voltage | `16, 0, 1, 0, 1, 0` | bytes 0-1, 0.1 V |
| LineCurr | `16, 16, 1, 0, 1, 0` | bytes 2-3 |
| CurrentRotation | `16, 32, 1, 0, 1, 0` | bytes 4-5, rpm = raw × 4 / pole pairs |
| MosTemp | `8, 48, 1, 0, 1, 0` | byte 6, °C |
| MotorTemp | `8, 56, 1, 0, 1, 0` | byte 7, °C |

**CAN Stat** on `0x102`

| Item | Cells | Meaning |
|---|---|---|
| GearStat | `2, 0, 1, 1, 1, 0` | byte 0 bits 0-1: 0 N, 1 forward, 2 reverse |
| SpeedStat | `3, 2, 1, 1, 1, 0` | byte 0 bits 2-4: 0 low, 1 mid, 2 high |

Then **Write**, then **Save And Reset**. You should see `0x101` and `0x102`
every ~50 ms. Example at 70.6 V, 25 °C: `101 = C2 02 00 00 00 00 19 16`.

### The control frame you send

Standard ID `0x100`, byte 1:

| Bits | Meaning |
|---|---|
| 0-1 | speed mode: 0 low, 1 mid, 2 high (3 is clamped to high) |
| 2-3 | gear: 0 neutral, 1 forward, 2 reverse |

### Speed limit per mode

Each mode is a **ceiling**, not a throttle scale. Below the cap every mode
runs at the same speed.

| Mode | Parameter | Address | Factory |
|---|---|---|---|
| low | LowSpeed | `0x28` | 4000 |
| mid | MidSpeed | `0x2A` | 6000 |
| high | MaxSpeed | `0x15` | 8520 |
| reverse | BackSpeed | `0x29` | 600 |

- **`0x28` is LowSpeed and `0x29` is BackSpeed.** The community map has them
  the other way round.
- With 5 pole pairs, real rpm settles at about **0.8 ×** the cap
  (500 → ~400, 1500 → ~1200, 3000 → ~2400).

### Bench test (wheel off the ground, full throttle)

| Test | Result |
|---|---|
| neutral, any mode | 0 rpm |
| forward, low / mid / high | stops at each cap (400 / 1205 / 2390 rpm with caps 500 / 1500 / 3000) |
| reverse, low / mid / high | ~535 / **~307** / ~485 rpm |
| high → low while running | slows to the low cap, with regen |
| gear 0 while running | neutral within ~1 s |
| CAN frames stop while running | **keeps driving in the last gear** |
| forward → reverse while turning | goes to neutral and stops; reverse does not engage |

### Known behaviour

- **Lost CAN while driving keeps the last gear.** DefaultN only gives
  neutral at power-up. Your node should send neutral before it stops.
- Reverse only engages from a stop.
- Reverse in mid mode was ~40 % slower than in low and high, every time.
  Cause not found.

### What CAN can and cannot do (protocol 60)

| Function | Over CAN? |
|---|---|
| Speed mode low / mid / high | yes |
| Forward / neutral / reverse | yes (needs Gear = DefaultN) |
| BOOST | no. Value 3 is clamped to high. BOOST needs its wire |
| ECO | no. It is a throttle-curve setting, not a command |
| Side stand | no, see below |
| Brake, cruise, park, anti-theft | no. Wire inputs only; they can be reported in CAN Stat |

**Side stand:** never put SideStandID on the same ID as GearID or
SpeedLimitID. The controller then ignores the whole frame. On its own ID the
controller read the bit but did not stop driving. Leave SideStandID at 0x0.

---

## 2. Full control over CAN (protocol 61)

Protocol 61 ("CAN Command 61") lets a vehicle controller send gear, speed
mode, cruise and even throttle over CAN. IDs follow SAE J1939 style; the
default is extended frames, Intel byte order.

### Setup that worked

| Setting | Value |
|---|---|
| CAN protocol | 61 (settable over UART, `0x86` low 6 bits) |
| Frame type | Extended |
| ControlType | 48 (CAN dialog only) |
| Gear BytePos / BitPos | 0 / 0 |
| LimitSpeed BytePos / BitPos | 3 / 0 |

- Firmware **HA86**: protocol 61 sets ControlID `0x131` and status IDs
  `0x33x` by itself (speed pulse 4). With **Extended**, the control frame
  `0x10F83807` works.
- Firmware **HA91**: the control frame is `0x10F80807`. Status comes on
  `0x10F80708` and `0x10F90708`.
- One layout from a newer firmware (protocol 60 + Extended + `0x10F83807`)
  was ignored by HA86, even after a power cycle. Match the setup to your
  firmware.

### Control frame (every 20 ms)

Bytes counted from 1, as in the protocol document.

| Byte | Bits | Meaning |
|---|---|---|
| 1 | 0-1 | gear: 0 N, 1 forward, 2 reverse |
| 1 | 2 | 1 = gear from CAN (0 = from the wires) |
| 1 | 3 | CAN brake |
| 4 | 0-1 | speed mode: 0 low, 1 mid, 2 high, 3 BST |
| 4 | 2-3 | cruise: 0 off, 1 +2 km/h, 2 -2 km/h, 3 start / hold |
| 4 | 4-7 | control mode: 0x0 grip (wired throttle), 0x5 throttle over CAN, 0xA torque, 0xC speed. Change only at 0 rpm |
| 5-6 | | control data, signed (throttle: 0 to 256 = full) |
| 7-8 | | copy of bytes 5-6 |

Change throttle data in small steps (at most 8 per 20 ms). Forward and
reverse only switch below 500 rpm with zero control data.

**Safety finding:** the document says the controller stops after 150 ms
without a control frame. In **grip mode** (wired throttle) losing the frames
did **not** stop the motor on our bench. Plan your own watchdog.

### Status frames with the HA91 layout (decoded on the bench)

`0x10F80708`, every 50 ms

| Byte | Meaning |
|---|---|
| 0-1 | bus voltage, 0.1 V, little-endian (`76 02` = 63.0 V) |
| 2 | follows the current; likely 0.1 A (not proven) |

`0x10F90708`, every 50 ms

| Byte | Meaning |
|---|---|
| 0 | bits 0-1 gear (0 N, 1 D, 2 R); bits 2-3 speed mode (0 low, 1 mid, 2 high); bit 7 = in gear and standing |
| 1 | load (higher when accelerating, 0 when coasting) |
| 4-5 | rpm at 4 pole pairs, little-endian, always positive. Real rpm = value × 4 / pole pairs |
| 6 | controller temperature + 40 °C |
| 7 | motor temperature + 40 °C |

Examples of byte 0: `04` N mid · `81` D low standing · `01` D low moving ·
`09` D high moving · `82` R standing · `02` R moving.

---

## 3. Cruise

| Firmware / setup | Result |
|---|---|
| HA86, protocol 61, grip mode | cruise bits work without the CruiseEN wire. It holds only while 3 is sent again and again; 0 turns it off (and the bike brakes to a stop at idle throttle). It overshoots when it engages. +2 worked; -2 did not after +2 |
| HA91, throttle over CAN | cruise did not hold |
| HB38, protocol 60 | cruise works over CAN, but the speed drops 15-20 % at engage and then wanders about ±8 % |

## 4. Stock dashboards need the factory layout

A common stock dashboard (CAN-reading MCU) only understood the controller in
its **factory CAN18 layout**: frames `0x10F8109A` and `0x10F8108D` at
250 kbit/s. With protocol 61 or a custom layout the dashboard showed nothing.
Keep a backup of the factory settings before you change the CAN protocol.

## 5. Throttle

- PC tool Paras tab: LowThrottle 1.10 V, HighThrottle 3.00 V.
- The controller ignores any HighThrottle below 3.0 V. It keeps the old value
  with no error.
- A 3.3 V DAC output was loaded down to ~2.25 V at the throttle pin, so only
  ~60 % of the range was used. Use a 5 V buffer for a fake throttle.

## 6. Using the PC tool

- **Save a backup first** (app menu → Save As → `.heb`). The `.heb` holds the
  main parameters. The CAN layout dialog does not seem to be in it, so
  screenshot that dialog too.
- To edit a box: double-click it, type, Enter, all in one go. The tool
  refreshes boxes from the controller and overwrites slow typing. Check the
  value a few seconds later.
- While the tool is connected it owns the serial line. Writes from another
  device at the same time get lost.
- AutoLearn spins the motor.
- A parameter write is only stored after the commit frame. Read it back
  ~2 s later.

## Credits

The UART frame format and the first parameter names come from
[jackhumbert/fardriver-controllers](https://github.com/jackhumbert/fardriver-controllers).
This repo adds the CAN work, the corrections and the newly found addresses.

## License

Text: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Share it,
change it, and credit this repo.
