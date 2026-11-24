# Fardriver parameters

Every value here lives in controller flash and is set with one 8-byte frame.
UART and Bluetooth carry the same frames, so anything the phone app changes,
any device on the controller's serial port can change (3.3 V logic, 19200 8N1).

    AA  C0+len  addr  addr  low  high  crc_lo  crc_hi

`addr` counts 16-bit words, so one frame sets one word. Two 8-bit parameters
often share a word: a name marked `+hi` is the high byte. To change one of a
pair, read its block first, change that byte, and send both bytes back.

Values below are from a bench capture in September 2026 on an NS30
(`ND72680B_24_3`, firmware HA86). Field names and scaling come from
github.com/jackhumbert/fardriver-controllers.

## Checked against the PC tool (2026-10-05)

The names above come from a community map. Two of them were **swapped**:
`0x28` is LowSpeed and `0x29` is BackSpeed (the map had them the other way
round). Found by writing `0x29` and watching BackSpeed change in the
Fardriver PC tool. The rows below were checked the same way:

| Block | PC tool name | Notes |
|---|---|---|
| `15` | MaxSpeed | speed cap in high mode (D) |
| `28` | LowSpeed Para -> Speed | speed cap in low mode (DL) |
| `29` | BackSpeed | speed cap in reverse |
| `2A` | MidSpeed Para -> Speed | speed cap in medium mode (DM) |
| `32` | LowSpeed / MidSpeed LineCurr | stored as x/128: 58 -> 45 %, 96 -> 75 % |
| `33` | LowSpeed / MidSpeed PhaseCurr | stored as x/128: 64 -> 50 %, 96 -> 75 % |
| `6C` | SpeedLimitRpm | only used when the speed-limit input is on |

**Speed caps are not in motor rpm.** With 5 pole pairs, the rpm we show
(MeasureSpeed x 4 / 5) settles at about **0.8 x** the cap. Measured: cap 500
-> ~400 rpm, 1500 -> ~1200, 3000 -> ~2400.

Throttle range in the PC tool (Paras tab): LowThrottle 1.10 V, HighThrottle
3.00 V. These are not the `C4`/`C5` words: those hold option flags (manual
9.21/9.22).

## Checked on 2026-10-07: what serial can change

Goal: never need the PC tool for settings.

**Write test.** 67 plain number settings were each written with a slightly
different value over serial, read back, and restored.

- **Accepted (60):** `0C 0F 10 11 13 14 15 16 17 18 19 1C 1E 1F 26 27 28 29 2A
  2B 2C 2D 2E 2F 30 31 32 33 34 63 64 66 6C 84 88`-`98 9E 9F C6 C7 C9 CC CD
  CE CF`.
- **The controller checks the value.** It ignores a write it does not like
  and keeps the old value, with no error. Seen: HighThrottle below 3.0 V,
  AccStep above 224, BlockTime below 10, a positive value in `nratio` 14 to
  19 (`99`-`9B`).
- **Locked (4), no value was accepted:** `25` HighVolProtect, `83`
  HighVolRestore, `85` MosTempProtect / Restore, `82`. Not tried in the PC
  tool yet.
- Not tested on purpose: sensor calibration (`00`-`04`, `0D`, `0E`), dates,
  odometer, version, password and phone words, CAN protocol and bit rate,
  learn values, unknown words.

**New addresses, found by changing one box in the PC tool and comparing two
full dumps.** All of them also accept a serial write.

| Addr | Bits | PC tool name | Values |
|---|---|---|---|
| `1A` | 15-13 (high byte) | Function > Gear | 0 DefaultN, 1 DefaultFW. Rest of the list, not tried: InverseN, ButtonLow, ButtonHigh, ButtonMid, None |
| `1A` | 3-2 (low byte) | Paras > ThrottleRsp | 0 Linear, 2 Eco. 1 is probably Sport (not tried) |
| `0B` | 2-0 (low byte) | Function > Brake | 0 Connect Stop, 1 Disconnect Stop. Rest of the list, not tried: PConnect Stop, PDisconnect Stop, Invalid, LinearBrake |
| `0B` | 10-8 (high byte) | Function > SlowDown | the number itself (3 now) |
| `7C` | low byte | Paras > BackThrottle and WeakResponse | changing 0 -> 3 and 0 -> 2 together turned `0x01` into `0x65`. Best fit: WeakResponse in bits 3-1, BackThrottle in bits 6-5. Not checked one at a time |
| `CB` | 11-8 (high byte, low nibble) | Function > HighLow | 13 = CAN3Speed (known before) |

Word values on the NS30 now: `1A` = `0x0403`, `0B` = `0x4360`, `7C` =
`0x1101`.

**More addresses, found the same way on 2026-10-07 (second round).** An
automatic script stepped each PC tool box by one item, compared all blocks,
and wrote the old word back over serial. Every one was restored by a serial
write, so every one accepts serial writes.

| Addr | Bits | PC tool name | Notes |
|---|---|---|---|
| `0B` | 2-0 | Function > Brake | 0 Connect Stop, 1 Disconnect Stop |
| `0B` | 7-4 | Paras > TempDetect | 6 = NTC10K |
| `0B` | 10-8 | Function > SlowDown | the number |
| `0B` | 12-11 | Function > PC13 | 0 = Normal Rs |
| `0B` | 14-13 | Function > Park | 2 = Invalid |
| `0B` | 15 | Paras > MotorDir | 0 now |
| `1A` | 1-0 | Function > Follow | 3 = BackBar |
| `1A` | 3-2 | Paras > ThrottleRsp | 0 Linear, 2 Eco |
| `1A` | 5-4 | Paras > WeakCharc | 0 = Fast |
| `1A` | 15-13 | Function > Gear | 0 DefaultN, 1 DefaultFW |
| `7C` | 3-1 | Paras > WeakResponse | the number |
| `7C` | 6-5 | Paras > BackThrottle | the number |
| `CB` | 11-8 | Function > HighLow | 13 = CAN3Speed |
| `CB` | 15-12 | Function > LongRE | 0 = Invalid |
| `D1` | low byte | Paras > AngleDetect | 1 = Encoder |
| `9C` | 6 | Function > NoLoad | 0 or 1 |
| `BF` | word | Function > BOOSTTime | seconds x 500 (5000 = 10 s) |
| `C0` | word | Function > BOOSTStep | seconds x 500 (17500 = 35 s) |

Pin choices (Function > IO Function Select), one 4-bit code each. Codes
seen: 15 = None, 0 = NC, 5 = PIN14, 6 = PIN3, 7 = PIN8, 10 = PIN2.

| Addr | Bits 15-12 | Bits 11-8 | Bits 7-4 | Bits 3-0 | Now |
|---|---|---|---|---|---|
| `69` | BOOST | Cruise | Side | P | `0xFFF0` |
| `6A` | FW | RE | HighSpeed | LowSpeed | `0xF76A` |
| `6B` | Charge | AntiTheft | Seat | VolSwitch | `0xF5FF` |
| `CA` | Repair | SpeedLimit | (other) | (other) | `0xF055` |

**Still unknown: ECOAccRatio and ExchangePhase.** Changing them in the PC
tool changed no word in any block the board reads (`00`-`35`, `5D`-`6E`,
`7C`-`E1`). They may live in a word the board does not read, or the tool
may only send them on Save.

## Read-only live data

Same map, but status rather than settings. Writing to these is meaningless.

| Block | Holds |
|---|---|
| `E2` | gear, direction, brake, error flags, MeasureSpeed |
| `E8` | pack voltage, line current |
| `EE` | phase A and C currents |
| `D6` | MOSFET temperature, AutoLearn and error bits |
| `F4` | motor temperature, state of charge, loop time, settings date |
| `FA` | phase zero calibration, motor running state |

## Accuracy

Rows marked `?` sit in a block that mixes bit flags with values. My offsets
there can be a byte out, and a few readings are obviously wrong
(`FAIF 12545`, `SpdPulseNum 40459`, `TorqueCoeff 60416`). Confirm any `?` row
against the app before writing to it. Unmarked rows land on clean 16-bit
boundaries and read back sensible values.

## Settings

| Addr | Parameter | Now | Scaling | |
|---|---|---|---|---|
| `00` | VolCoeff | 256 |  |  |
| `01` | Voltage2Coeff | 256 |  |  |
| `02` | PhaseACoeff | 273 |  |  |
| `03` | LineCoeff | 439 |  |  |
| `04` | PhaseCCoeff | 273 |  |  |
| `05` | SaveNum | 146 |  |  |
| `07` +hi | SpeedKI | 10 |  | ? |
| `08` | **ThrottleLow** | 22 | / 20 = 1.10 V, matches the PC tool. Set to 16 (0.80 V) with the PC tool on 2026-10-06, not stored | checked 2026-10-06 |
| `08` +hi | **ThrottleHigh** | 60 | / 20 = 3.00 V, matches the PC tool. A plain write to `08` works, but the controller ignores any HighThrottle below 3.0 V (2.2, 2.4, 2.5 ignored; 3.1 accepted) | checked 2026-10-06 |
| `09` | ? | 0 |  | ? |
| `09` +hi | FAIF | 12545 |  | ? |
| `0A` +hi | CurveTime | 24576 |  | ? |
| `0C` | PhaseOffset | 239 | / 10.0, Send(0x0A, 0x07) |  |
| `0D` | ZeroBattCoeff | 896 |  |  |
| `0E` | FullBattCoeff | 1250 |  |  |
| `0F` | StartKI | 4 |  |  |
| `0F` +hi | MidKI | 8 |  |  |
| `10` | MaxKI | 12 |  |  |
| `10` +hi | StartKP | 0 |  |  |
| `11` | MidKP | 80 |  |  |
| `11` +hi | MaxKP | 120 |  |  |
| `12` | LD | 900 |  |  |
| `13` | AlarmDelay | 500 | 0x13 |  |
| `14` | PolePairs | 5 | 0x14, Send(0x12, 0x01) |  |
| `14` +hi | unk14b | 0 | alway_s sent 0x59? |  |
| `15` | MaxSpeed | 8520 | 0x15, Send(0x12, 0x02) |  |
| `16` | RatedPower | 5000 | 0x16, Send(0x12, 0x03) |  |
| `17` | RatedVoltage | 720 | 0x17 / 10, Send(0x12, 0x04) |  |
| `18` | RatedSpeed | 5300 | Send(0x12, 0x05) | ? |
| `19` | MaxLineCurr | 360 | / 4, Send(0x12, 0x1B) | ? |
| `1B` | LQ | 329 |  | ? |
| `1C` | BattRatedCap | 30 |  | ? |
| `1D` | IntRes | 32 | Send(0x0F, 0x08) | ? |
| `1E` | FwReRatio | 64 |  | ? |
| `1F` | LowVolProtect | 605 | / 10 | ? |
| `22` | ModelYear | 26 | + 2000 | ? |
| `22` +hi | ModelMonth | 9 |  | ? |
| `23` | ModelDay | 14 |  | ? |
| `23` +hi | TimeHour | 9 |  | ? |
| `24` | TimeMin | 9 |  |  |
| `24` +hi | TimeSecond | 37 |  |  |
| `25` | HighVolProtect | 907 | / 10 |  |
| `26` | CustomMaxLineCurr | 360 | / 4 |  |
| `27` | CustomMaxPhaseCurr | 1600 | / 4 |  |
| `28` | **LowSpeed** | 4000 | speed cap in low mode (DL) | checked 2026-10-05 |
| `29` | **BackSpeed** | 600 | speed cap in reverse | checked 2026-10-05 |
| `2A` | MidSpeed | 6000 |  |  |
| `2B` | Max_Dec | 128 |  |  |
| `2C` | FreeThrottle | 100 |  |  |
| `2C` +hi | unk2Cb | 0 |  |  |
| `2D` | MaxPhaseCurr | 1600 | / 4, Send(0x12, 0x1A) |  |
| `2E` | SpeedAnalog | 10000 |  |  |
| `2F` | Max_Acc | 128 |  |  |
| `30` | StopBackCurr | 8 | Send(0x12, 0x18) |  |
| `31` | MaxBackCurr | 15 | Send(0x12, 0x19) |  |
| `32` | LowSpeedLineCurr | 58 |  |  |
| `32` +hi | MidSpeedLineCurr | 96 |  |  |
| `33` | LowSpeedPhaseCurr | 64 |  |  |
| `33` +hi | MidSppedPhaseCurr | 96 |  |  |
| `34` | BlockTime | 10 |  |  |
| `35` | SpdPulseNum | 40459 |  |  |
| `5D` | unk5D | 0 |  |  |
| `5E` | unk5E | 0 | RPM index? |  |
| `5F` | unk5F | 0 |  |  |
| `60` | unk60 | 0 |  |  |
| `60` +hi | unk60b | 0 | counter of some sort? assigned 'a', 'b', |  |
| `61` | unk61 | 0 |  |  |
| `62` | unk62 | 0 | set to 0xCC when reading ADC1? |  |
| `62` +hi | unk62b | 0 |  |  |
| `63` | MaxLineCurr2 | 1400 |  |  |
| `64` | MaxPhaseCurr2 | 2720 | 0x64 |  |
| `65` | MotorDia | 4 | 0x65,  maybe not MotorDia - values inclu |  |
| `65` +hi | unk65b | 35 | 33, 35, 36 - MaxPhaseCurr2 multipler? |  |
| `66` | TempCoeff | 300 | 0x66 |  |
| `67` | ProdMaxVol | 720 | / 10.0 67, paracnt_0 |  |
| `68` | ISMax | 20257 | 0x68, paracnt_1 |  |
| `6C` | LmtSpeed | 3000 | LmtSpeed 6C | ? |
| `6D` | DistanceLSB | 301 | / 10 6D | ? |
| `6E` | ParaIndex | 51 | ParaIndex 6E, prolly Function Code | ? |
| `81` | DistanceMSB | 0 | this << 16 / 10, km | ? |
| `82` | ThrottleVoltage | 80 | * 0.01, 0x82 | ? |
| `83` | HighVolRestore | 887 | / 10.0, 0x83 | ? |
| `84` | MotorTempProtect | 140 | 0x84 | ? |
| `84` +hi | MotorTempRestore | 120 |  | ? |
| `85` | MosTempProtect | 100 | 0x85 | ? |
| `85` +hi | MosTempRestore | 80 |  | ? |
| `87` +hi | SoftwareVersionMinor | 86 |  | ? |
| `88` | RatioMin | 100 |  |  |
| `88` +hi | Ratio500 | 100 |  |  |
| `89` | Ratio1000 | 100 |  |  |
| `89` +hi | Ratio1500 | 100 |  |  |
| `8A` | Ratio2000 | 95 |  |  |
| `8A` +hi | Ratio2500 | 90 |  |  |
| `8B` | Ratio3000 | 85 |  |  |
| `8B` +hi | Ratio3500 | 80 |  |  |
| `8C` | Ratio4000 | 75 |  |  |
| `8C` +hi | Ratio4500 | 70 |  |  |
| `8D` | Ratio5000 | 65 |  |  |
| `8D` +hi | Ratio5500 | 60 |  |  |
| `8E` | Ratio6000 | 55 |  |  |
| `8E` +hi | Ratio6500 | 50 |  |  |
| `8F` | Ratio7000 | 45 |  |  |
| `8F` +hi | Ratio7500 | 35 |  |  |
| `90` | Ratio8000 | 25 |  |  |
| `90` +hi | Ratio8500 | 10 |  |  |
| `91` | Ratio9000 | 0 |  |  |
| `91` +hi | RatioMax | 0 |  |  |
| `92` | nratio_0 | -10 |  |  |
| `92` +hi | nratio_1 | -10 |  |  |
| `93` | nratio_2 | -10 |  |  |
| `93` +hi | nratio_3 | -10 |  |  |
| `94` | nratio_4 | -10 |  |  |
| `94` +hi | nratio_5 | -10 |  |  |
| `95` | nratio_6 | -10 |  |  |
| `95` +hi | nratio_7 | -10 |  |  |
| `96` | nratio_8 | -10 |  |  |
| `96` +hi | nratio_9 | -10 |  |  |
| `97` | nratio_10 | -8 |  |  |
| `97` +hi | nratio_11 | -6 |  |  |
| `98` | nratio_12 | -4 |  |  |
| `98` +hi | nratio_13 | -2 |  |  |
| `99` | nratio_14 | 0 |  |  |
| `99` +hi | nratio_15 | 0 |  |  |
| `9A` | nratio_16 | 0 | 9A | ? |
| `9A` +hi | nratio_17 | 0 |  | ? |
| `9B` | nratio_18 | 0 | 9B | ? |
| `9B` +hi | nratio_19 | 0 |  | ? |
| `9D` | InitVol | 0 | 9D | ? |
| `9E` | Stage1Curr | 53 | TurtleSpeedCurrCoeff | ? |
| `9F` | VolSelectRatio | 106 | 9F | ? |
| `9F` +hi | unk9Fb | 0 |  | ? |
| `AC` | unkAC | 0 | more password, pin states? |  |
| `AD` | unkAD | 176 | phone number start1[14] |  |
| `AE` | unkAE | 0 | phone number start0[14] |  |
| `AF` | unkAF | 0 |  |  |
| `B0` | unkB0 | 0 |  |  |
| `B1` | unkB1 | 0 |  |  |
| `B9` | unkB9 | 50710 |  | ? |
| `BA` | Pulse | 0 |  | ? |
| `BA` +hi | SQH | 0 |  | ? |
| `BB` | OnelineCurrCoeff | 64 | CurrentCoeff | ? |
| `BD` +hi | Stage1Soc | 64 |  | ? |
| `BE` +hi | BstTime | 34946 | / 500, seconds | ? |
| `BF` +hi | BstRelease | 23571 | / 500, seconds | ? |
| `C0` +hi | ParkTime | 34884 | / 500, seconds | ? |
| `C2` +hi | TorqueCoeff | 60416 | C3 | ? |
| `C4` +hi | LearnVolLow | 72 | LearnVol, VQL | ? |
| `C5` +hi | LearnVolHigh | 95 | LearnVoh, VQH, related to A11 | ? |
| `C6` | SlowDownRpm | 512 | ParkDiff | ? |
| `C7` | StartIs | 512 | IsInStart | ? |
| `C8` | ThrottleInsert | 256 | & 0x20 TCS | ? |
| `C9` | ExitFollowSpeed | 0 |  | ? |
| `C9` +hi | ReCurrRatio | 32 |  | ? |
| `CA` | AngleLearn | 85 |  | ? |
| `CB` | NoCanCnt | 75 | CAN Detect, ms, if >= 6 then * 2 else * | ? |
| `CC` | LearnThrottle | 24 |  | ? |
| `CC` +hi | SpeedLowCap | 0 | LmtSpdMinCap | ? |
| `CD` | MidSpeedCap | 0 | LmtSpdStartCap | ? |
| `CD` +hi | SpeedLimitByCap | 26 |  | ? |
| `CE` | MinSpeedCapCoeff | 128 | LmtSpdMaxCoeff | ? |
| `CF` | ReIsinAcc | 64 |  | ? |
| `D0` | Data0 | 8 |  | ? |
| `D0` +hi | Data1 | 97 |  | ? |
| `D1` | BMQHALL | 1 |  | ? |
| `D1` +hi | AVGPower | 1 | * 4, Wh/km, DataVoltage1 | ? |
| `D2` | WheelRatio | 70 |  | ? |
| `D2` +hi | WheelRadius | 12 |  | ? |
| `D3` | AVGSpeed | 129 | km/h, DataVoltage4 | ? |
| `D3` +hi | WheelWidth | 120 |  | ? |
| `D4` | RateRatio | 4000 | SpeedRatio | ? |
