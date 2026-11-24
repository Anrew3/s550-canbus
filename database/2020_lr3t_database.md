# 2020+ Digital Cluster (LR3T-14C026-JE)

**Label:** 2020+ digital (LR3T)

The 2020+ S550 12.4" digital instrument cluster, MCU software **LR3T-14C026-JE**. GT, EcoBoost, Bullitt and Mach 1
all run it. The Shelby GT500 MCU (LR3V-14C026-BF) is in [gt500_lr3v_database.md](gt500_lr3v_database.md), which
lists only what differs from this file.

Where the data came from: customipc firmware trace of the MCU, 22 Nov 2026, plus an emulator running the real
firmware; nothing here is bench tested. Each received field was followed from the code that reads it to whatever
it drives (lamp, pop-up, gauge, chime, a frame the cluster sends), then the real MCU firmware and the real display
program were run in an emulator and fields were set one at a time. Example frames are built from those field
tables, they are not captures from a car. Every byte not mentioned is `00`.

It may not hold for the analog FR33 / KR3T ([database.md](database.md)) or the 2018 - 2019 digital JR3T / KL3T
([2018_digital_database.md](2018_digital_database.md)). Where this file reads an ID differently from those files,
it says how the LR3T firmware reads it.

## Labels and conventions

Every row or claim inside an ID section carries a label:

- **exact**: read from the firmware code.
- **seen**: seen in an emulator running the real firmware (MCU plus display program).
- **inferred**: likely, from how the code uses it, not proven.

Every ID's Status is `From firmware` (or `Cluster sends` for IDs the cluster transmits): none of it is tested on a
bench or in a car yet.

- HS3 bus, 500 kbit/s, 11 bit IDs. FORScan finds the IPC there.
- Bytes 0 - 7, bit 0 = `0x01`, bit 7 = `0x80`. Multi-byte values are big-endian: the first byte named holds the high
  bits.
- **Run** means `0x3B2` byte 0 = `0x4_` (power mode Run). Most gauges, lamps and timeouts only work in Run, see
  [0x3B2](#0x3b2---body-info-power-mode-doors-lamps).
- "Option N" is an EEPROM (as-built) option of this cluster. Many features only exist with their option set.
- "Missing" means no frame for 5 s while in Run, see the Timeouts topic.

## Modules

| Module | Part number | Notes |
|---|---|---|
| IPC (cluster) | LR3T-14C026-JE | firmware only, not bench tested |

# 0x04C - Airbag / Seat Belts

**Sample:** `00 50 00 00 00` (built from the firmware field table: airbag lamp off, both belts not unbuckled) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 5.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 6 | Airbag lamp | `0` off, `1` on, `2` flashing, `3` on (inferred values; seen: these bits light the airbag lamp) |
| 0 | 5 - 4 | Passenger seat occupied | `1` occupied (exact) |
| 1 | 7 - 6 | Driver buckle | `0` / `1` buckled (not unbuckled), `2` unbuckled, `3` fault (exact) |
| 1 | 5 - 4 | Front passenger buckle | same coding (exact) |

- Belt-Minder (chime and belt lamp) uses the buckle states with options 28 / 29 / 30 / 58 / 80 (exact). The
  passenger counts as unbuckled only when the seat is occupied (byte 0 bits 5 - 4 = 1) and the belt is 2 (exact).
- Missing: airbag lamp on (seen), buckles read as not unbuckled, U0151-00 (RCM) logged (exact).
- The cluster echoes the airbag lamp state on [0x350](#0x350---airbag-lamp-echo-cluster-tx) (inferred).

```
All fine                                      00 50 00 00 00
Airbag lamp on                                40 50 00 00 00
Driver unbuckled                              00 90 00 00 00
```

# 0x076 - Track Apps G Meter And Speed

**Sample:** `00 00 2F FB FE 25 B8` (built from the firmware field table: g meter centred, 60 mph, speed valid) · **Rate:** missing after 5.0 s · **Status:** From firmware

The 2020 MCU reads two g meter values as 10 bit fields, value = `raw - 511` (`0x3FE` / `0x3FF` invalid), which is
the `mustang.c` encoding in [database.md](database.md) bit by bit. It also reads a speed. DLC 7.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 4 - 0, + byte 3 bits 7 - 3 | G meter axis A | 10 bit, `raw - 511` (exact); which axis is which is inferred |
| 3 | 2 - 0, + byte 4 bits 7 - 1 | G meter axis B | 10 bit, `raw - 511` (exact) |
| 2 | 5 (`0x20`) | Speed valid | **`1`** or the speed below doesn't count (exact) |
| 5 - 6 | | Track Apps speed | 0.01 km/h per count, `0xFFFE` / `0xFFFF` invalid. Runs the 0-60 / 0-100 timers, the brake test and the lap timer (exact) |

- The g meter page needs EEPROM option 206 (Accelerometer Display, as-built byte `0x070` bit 5) and both axes valid
  and fresh (exact).
- Axes timing out set U0121-00 (ABS); the speed timing out (no frames, or the speed valid bit clear) sets U0415-82
  (exact).
- Track Apps needs an admin key, or no MyKey feature (exact).

```
G meter centred, 60 mph (9656), speed valid   00 00 2F FB FE 25 B8
G meter centred, stopped, speed valid         00 00 2F FB FE 00 00
```

# 0x077 - Cruise / Steering / Hill Start Status

**Sample:** `70 00 00 00 00 85 41 00` (built from the firmware field table: every group valid, no warnings; seen clean in the emulator) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

On HS3 the 2020 MCU reads this frame as cruise / steering / hill start status. It has five signal groups, each
with its own valid bit and its own 5 s deadline (exact, steering messages seen in the emulator):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 4 (`0x10`) | Valid, set speed group (byte 4) | `1` = valid. Group timing out sets U0401-82 (exact) |
| 0 | 5 (`0x20`) | Valid, steering lamp group (byte 6 bits 5 - 4) | `1` = valid. `0`: after 5 s the steering lamp is forced to state 1, and with EEPROM option 51 (EPAS) set **U0420-82** is logged (exact) |
| 4 | | Set speed | `0` none, `0x01 - 0xFD` speed in the cluster's unit, `0xFE` / `0xFF` invalid (exact; seen: the set speed bug moves) |
| 5 | 0 (`0x01`) | Valid, second cruise group | `1` = valid (exact) |
| 5 | 1 (`0x02`) | ACC set speed style | only used in adaptive cruise modes (inferred) |
| 6 | 0 (`0x01`) | Valid, hill start group (byte 6 bits 3 - 1) | `1` = valid. Group timing out sets U0415-82 (exact) |
| 6 | 3 - 1 | Hill start lamp | **`6` = Hill Start Assist Not Available**, every other value off (seen) |
| 6 | 5 - 4 | **Steering lamp** | `0` off, `1` Steering Assist Fault Service Required, `2` Steering Fault Service Now, `3` Steering Loss Stop Safely (seen) |
| 6 | 6 (`0x40`) | Valid, fifth group | `1` = valid, no visible use (exact) |

- **U0420-82**: on the 2020 MCU it comes only from the steering lamp group timing out (byte 0 bit 5 clear, or no
  frame), and only with option 51 (EPAS) set (exact).
- Speed doesn't take part in it on the 2020 MCU; bytes 0 - 1 hold a 12 bit value nothing reads (exact).
- The steering-warning fix frame `FF FF FF FF FF FF 00 FC` in [database.md](database.md) has byte 0 bit 5 set and
  steering value 0, so on LR3T it keeps the steering group valid. Its byte 6 bit 0 is clear, so on the 2020 MCU the
  hill start group would time out (U0415-82) (inferred from the DTC map).
- Missing in Run: **Steering Assist Fault Service Required** (seen).

```
Clean, every group valid, no set speed         70 00 00 00 00 85 41 00
Set speed 65                                   70 00 00 00 41 85 41 00
Hill Start Assist Not Available                70 00 00 00 00 85 4D 00
Steering Assist Fault Service Required         70 00 00 00 00 85 51 00
Steering Fault Service Now                     70 00 00 00 00 85 61 00
Steering Loss Stop Safely                      70 00 00 00 00 85 71 00
```

# 0x081 - Steering Wheel Keys

**Sample:** `00 00 00` (built from the firmware field table: all keys released), `10 00 00` (OK), `08 00 00` (Up), `01 00 00` (Down), `00 40 00` (Back), `00 01 00` (Menu), `00 08 00` (Media), `00 04 00` (Navi), `00 00 80` (Pony menu, Cobra menu on a Shelby) · **Rate:** missing after 5.0 s · **Status:** From firmware

A set bit = key held. DLC 3. The keys the 2020 MCU passes to the display, with the display program's own name for
each (exact):

| Byte | Bits | Key | Display name |
|---|---|---|---|
| 0 | 4 (`0x10`) | OK | `vehicle_hk_ok` (exact) |
| 0 | 3 (`0x08`) | Up | `vehicle_hk_up` (exact) |
| 0 | 0 (`0x01`) | Down | `vehicle_hk_down` (exact) |
| 1 | 6 (`0x40`) | Back | `vehicle_hk_back` (exact) |
| 1 | 0 (`0x01`) | Menu | `vehicle_hk_settings`, firmware name SETTINGS (exact) |
| 1 | 3 (`0x08`) | Media | `vehicle_hk_media` (exact) |
| 1 | 2 (`0x04`) | Navi | `vehicle_hk_navi` (exact) |
| 1 | 5 (`0x20`) | HUD | `vehicle_hk_hud`: forwarded by the cluster (exact), no visible effect on an S550, which has no HUD |
| 2 | 7 (`0x80`) | Pony menu key (Cobra menu on a Shelby) | `vehicle_hk_view`, firmware name VIEW (exact) |

- Byte 0 `0x02` / `0x04` reach the MCU but it has no key for them, so the display never sees them (exact). The
  digital menu uses Up / Down / OK / Back.
- A frame like `46 01` also sets those two bits and byte 0 `0x40` (unmapped); `00 01 00` alone is the Menu key
  (exact).
- **Phone** doesn't come from `0x081`: it is built from [0x2A1](#0x2a1---phone-key). **Voice** has no field in the
  MCU (exact).
- Missing sets U0212-00 (steering column module) (exact). Send zeros when released.

**Holding a key.** The MCU turns each held key into a hold level the display reads (short press vs long press,
e.g. hold OK to reset a trip). Counts are exact; the ms assume the 4 ms timer tick (inferred):

| Level | After (counts) | About |
|---|---|---|
| 1 | press | 0 ms |
| 2 | 37 | 148 ms |
| 3 | +87 (124) | 0.5 s |
| 4 | +125 (249) | 1.0 s |
| 5, 6, 7, 8 | +250 each (499, 749, 999, 1249) | 2.0, 3.0, 4.0, 5.0 s |
| 9 | +750 (1999) | 8.0 s |

- Auto repeat while held: first repeat after 75 counts (~300 ms), then every 37 (~148 ms). Release resets the
  level to 0 (exact). A bench tool only needs to keep the bit set; it doesn't have to repeat frames faster.

# 0x084 - Date / Time

**Sample:** `1A 00 01 46 1E 00 0E` (built from the firmware field table: 22 Nov 2026 14:30) · **Rate:** missing after 5.0 s · **Status:** From firmware

From the head unit / GPS clock. Date stamps Track Apps results (drag / acceleration log, lap timer) and gives the
hour for the Quiet Start exhaust window. DLC 7.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | | Year | `2000 + raw`, `0xFF` invalid (exact) |
| 2 - 3 | | Day of year | `1 - 366` (Jan 1 = 1), else invalid. The cluster works out month / day, leap years included (exact) |
| 4 | | Minute | `0 - 59` (inferred, from the range check) |
| 5 | | Likely seconds | copied, not range checked (inferred) |
| 6 | | Hour | `0 - 23`, 24 h (exact) |

- Missing: no date on Track Apps records, and Quiet Start doesn't fire (exact). Nothing visible otherwise; no clock
  display on this cluster is fed from `0x084` (exact).

# 0x090 - Clock Format

**Sample:** `00 00 00 00 00 00 10` (built from the firmware field table: 24 hour) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 6 | 5 - 4 | Clock format request | `1` 24 hour, `2` 12 hour, `0` / `3` keep (exact) |

- The cluster stores it and sends it back out on [0x2FD](#0x2fd---settings-units-exhaust-request-cluster-tx) byte 0
  bits 1 - 0 (inferred values).

```
24 hour                                       00 00 00 00 00 00 10
12 hour                                       00 00 00 00 00 00 20
```

# 0x108 - Brake Test Trigger (Track Apps)

**Sample:** `00 06 0F 00 80 00 00 00` (built from the firmware field table: valid, brake test trigger) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 | 2 (`0x04`) | Frame valid | `1`, or the frame doesn't reload the deadline in Run (exact) |
| 4 | 7 - 6 | Brake state | `2` starts the Track Apps **Brake Performance** test (exact), likely "brake applied" (inferred); other values no trigger |

- The test must be armed and above its start speed (speed from [0x076](#0x076---track-apps-g-meter-and-speed)), and
  needs option 208 (exact).
- Missing: the brake test resets and reports a fault (exact).

# 0x123 - Language Change Answer (cluster TX)

**Sample:** `40` (built from the firmware field table: language changed) · **Rate:** on event · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 6 | Answer to a language request on [0x191](#0x191---units--language-request) | `0` idle / already active, `1` changed, `2` not available (exact) |

# 0x14C - Wrench Lamp Request

**Sample:** `18 00 00 03 00 03 40 00` (built from the firmware field table: lamp off, valid, every update bit set) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 0 (`0x01`) | Lamp request | `1` lights the wrench (lamp slot 6) (exact) |
| 3 | 0 (`0x01`) | Qualifier | `1` = valid. With it clear the frame doesn't reload the deadline in Run (exact) |

- The message timing out, or frames with byte 3 bit 0 clear, light the wrench and set U0401-82 (invalid data from
  the PCM) (exact). On the 2020 MCU EEPROM option 2 = 0 gates it (exact).
- The lamp logic is shared with the 4x4 / AWD options (5, 258), so the sender is likely a driveline module
  (inferred).
- Missing in Run: **See Manual** (seen).

# 0x156 - Coolant And Oil Temp

**Sample:** `96 A0 00 00 03 00 00 00` (built from the firmware field table: coolant 90 C, oil 100 C, coolant valid) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | | Coolant temp | `raw - 60 = °C` (exact, seen) |
| 1 | | Oil temp | `raw - 60 = °C`, `0xFE` / `0xFF` invalid (exact) |
| 4 | 1 - 0 | Coolant valid | **must be `3`**. `0 - 2`: after 5 s the coolant reads invalid (hot lamp on, needle to its fault position) (exact) |
| 4 | 4 - 3 | Engine fail-safe cooling | `1` or `2` = fail-safe active (overheat message) (exact bits, inferred name) |

- Byte 4 is two fields: `03` = valid, no fail-safe; `08` = fail-safe 1 with the valid bits clear; `0B` = valid plus
  fail-safe 1 (exact).
- The needle has a wide flat normal band: 82 - 110 °C all sit at half scale (exact, from the gauge curve).
- Hot engine lamp: on at 121 °C (raw 181), off at 119 °C (raw 179) with option 260 = 0 (exact). Other option 260
  values move it (116 / 113 / 127 °C).
- Missing: U0100-00 (lost communication with the PCM) (exact); hot lamp, needle to the fault position and
  **Engine Coolant Over Temperature** (seen).
- Bytes 2, 3 and 7 aren't read (exact). Where oil temp shows on screen is open.

```
Coolant 90 C, oil 100 C, coolant valid         96 A0 00 00 03 00 00 00
Same, engine fail-safe cooling 1               96 A0 00 00 0B 00 00 00
```

# 0x166 - Odometer From Another Module

**Sample:** `00 00 00 FF FF FF 00` (built from the firmware field table: odometer invalid, safe on a bench) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 - 5 | | Odometer | 24 bit count from another module, `0xFFFFFF` invalid (exact). Unit likely km (inferred) |
| 1 | 3 - 0 | Status | only tested for non-zero, by one EEPROM option 21 check (inferred) |

- With EEPROM option 90 (Odometer Delta) the cluster compares this with its own stored odometer and, when they
  disagree, **rewrites its stored odometer**, flagged on [0x434](#0x434---fuel-mykey-cluster-tx) byte 0 bit 2
  (exact). **On a bench this can change the cluster's mileage**, so only the invalid frame is given here.
- Missing or invalid: the odometer check is blocked (exact).

# 0x167 - Engine Run State

**Sample:** `20 00 00 00 00 00 00` (built from the firmware field table: engine running) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7. The cluster uses only byte 0.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 5 - 4 | Engine run state | `0` not running, **`2` running**, `1` / `3` other states (likely cranking / stalled) (exact field, inferred names for 1 and 3) |
| 0 | 3 - 1 | Start-stop state | `3` is treated as running too, likely auto start-stop with the engine stopped (inferred) |

- It gates the power mode logic, the adaptive cruise and forward collision lamps, the engine-on chime and
  [0x430](#0x430---odometer-settings-out-cluster-tx) byte 0 bit 7 (exact).
- Missing or never received: every user treats the engine as not running (exact).
- Bytes 1 - 2 and 5 - 6 are latched but never read (exact).

# 0x168 - Shift By Wire Warning A

**Sample:** `00 00 00` (built from the firmware field table: no warning) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 3.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 7 (`0x80`) | Warning request | `1` lights lamp slot 6 (the wrench) only with option 117 (Shift By Wire) = 1 (exact) |

- Missing also lights it with option 117 = 1 (exact). No visible change in the emulator with a normal EEPROM (seen).
  See [Wrench / See Manual inputs](#wrench--see-manual-inputs).

# 0x169 - Shift By Wire Warning B

**Sample:** `00 00 00` (built from the firmware field table: no warning) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 3.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 3 (`0x08`) | Warning request | `1` lights lamp slot 6 (the wrench) only with option 117 (Shift By Wire) = 1 (exact) |

- Same lamp logic as [0x168](#0x168---shift-by-wire-warning-a) (exact).

# 0x171 - Gear / PRNDS

**Sample:** `04 00 00` (built from the firmware field table: lever P, lever valid) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 3, all exact:

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 1 - 0 | Transmission mode | `0` normal, `1` manual (M), `2` SelectShift (S), `3` invalid (exact) |
| 0 | 3 - 2 | Lever valid | **must be `1`** or no gear is shown (exact, seen) |
| 0 | 7 - 4 | Gear number | `1 - 10` for the M / S number, `0` none, `11 - 15` invalid (exact) |
| 1 | 7 - 5 | Lever | `0` P, `1` R, `2` N, `3` D, `4` S (exact) |

- `00 00` has lever valid = 0, so no PRNDS letter shows (exact, seen).
- The M / S gear number needs option 61 (exact).
- A manual car in 3rd gear sent as `35 00 00` shows "GEAR 3" in the tach centre (seen). A frame with mode 0 on a
  manual shows no gear number (seen).
- Missing: no PRNDS letter (exact).
- The 2020 firmware's gear logic reads EEPROM option 2. Elsewhere in the firmware option 2 = 1 reads as manual; check
  the EEPROM before relying on it (open).

```
P                                             04 00 00
R                                             04 20 00
N                                             04 40 00
D                                             04 60 00
S                                             04 80 00
D, 4th gear                                   44 60 00
S with gear 3 (SelectShift)                   36 80 00
M with gear 1                                 15 80 00
Manual car, 3rd gear                          35 00 00
```

# 0x178 - Launch Control / Speed Limiter

**Sample:** `80 00 00 00 00 00 00 00` (built from the firmware field table: launch control state 4, launch lamp) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 5 | Launch control state | `0` off, `1` / `2` active, `3` off / idle, `4` active + launch lamp (likely ready / engaged), `5 - 7` fault (exact). `0x2_` = 1, `0x8_` = 4, `0xA_` = 5 |
| 0 | 3 - 2 | Speed limiter state | `0` off, `1` limiter on, `2` / `3` active with a set speed (inferred) |
| 2 | 6 - 4 | Lamp request | `1` / `2` = a lamp in two states, other values off (exact; which lamp is open) |
| 4 | 7 | Speed limiter flag | for the set speed indicator (inferred) |

- Needs EEPROM option 226 (launch control); the speed limiter needs option 12 (exact). Missing frame: launch control
  shows "not active" and both lamps go off (exact).
- The driver's launch rpm and on / off go out on [0x3C8](#0x3c8---mycolor-drive-mode-launch-requests-cluster-tx)
  (exact).

# 0x179 - Shift Arrow, Distance Counter

**Sample:** `00 00 00 00 10 00 00 00` (built from the firmware field table: byte 4 as in the emulator's healthy car) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 5 | Shift indicator (manual) | `0` off, `1 - 3` upshift arrow (option 123), `4` downshift arrow (option 312), `5 - 7` off (exact) |
| 2 | | Four single bits | inputs to the wrench logic (inferred) |
| 4 | 0 (`0x01`) | Lamp request | `1` lights one telltale (exact; which one is open) |
| 5 | | Distance counter | rolling count that drives the odometer; `0xFF` = invalid (exact). Distance unit open |
| 6 - 7 | 1 - 0 of byte 6, byte 7 | 10 bit value | fed with the distance count into the fuel economy model, likely fuel used (inferred) |

- Missing frame: the odometer stops counting and the shift arrow goes off (exact).
- Bytes 2 - 3 are never read as a number (exact).
- Byte 4 = `0x10` cleared a warning in the emulator's healthy car probe (seen).

```
Shift arrow up                                 20 00 00 00 10 00 00 00
Shift arrow down                               80 00 00 00 10 00 00 00
```

# 0x18A - Adaptive Cruise / Forward Collision Warning

**Sample:** `00 00 00 00 00 00 00` (built from the firmware field table: no FCW lamp, no ACC message), `40 00 00 00 00 00 00` (FCW lamp on, FCW menu locked) · **Rate:** missing after 5.0 s · **Status:** From firmware

From the radar / ACC module. DLC 7. The cluster reads about a third of its fields:

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 5 | FCW state | **`2` = FCW lamp on and the FCW menu items locked**; `6` / `7` invalid (exact). Option 53 |
| 1 | 0 (`0x01`) | Forward collision alert | `1` = alert sound (exact) |
| 2 | 1 (`0x02`) | Set speed indicator variant | (exact) |
| 3 | 7 - 4 | ACC status / message code | values 1, 3, 7, 9, 10, 14 light ACC messages; 2 = set speed indicator style (exact; lamp names open) |
| 4 | 2 - 0 | ACC gap level | `1 - 4` shown (5 = 4); `6` / `7` invalid (exact) |
| 4 | 4 (`0x10`) | Cruise on | sent on to [0x430](#0x430---odometer-settings-out-cluster-tx) byte 4 bit 3 (inferred) |
| 4 | 7 - 6 | ACC alert request | `1 - 3` = alert sounds (exact) |

- Missing: FCW lamp on (state 2), FCW menu locked, gap level shows 3 (exact).
- It is the only collision warning message between `0x14D` and `0x194` on the 2020 MCU, so it is the first candidate
  for "Collision detection unavailable" from zeros in that range (inferred). `0x167` also gates the FCW lamp
  (inferred).

# 0x18B - Shift Light Frame (cluster TX)

**Sample:** `00 00 00 1E 00 00 00 00` (seen: what the 2020 MCU sends) · **Rate:** 1000 ms · **Status:** Cluster sends (from firmware). Don't send it

- Constant on the 2020 MCU: its shift light fields are never written (exact, seen). The GT500 MCU fills them, see
  [gt500_lr3v_database.md](gt500_lr3v_database.md).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 | 5 - 0 | Shift rpm field | `0x1E`, constant on the 2020 MCU (seen) |

# 0x191 - Units / Language Request

**Sample:** `3C 00 00` (built from the firmware field table: no request, keep the stored units) · **Rate:** missing after 5.0 s · **Status:** From firmware

A 3 byte settings request, most likely from SYNC. On each new frame the cluster adopts it into its own settings,
which is why it is stored (exact):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 2 | Distance / fuel economy unit | `0` metric (from Miles & Gallons it switches to l/100 km or km/l, by EEPROM option 14), `1` Miles & Gallons, `2` / `3` no request (exact) |
| 0 | 5 - 4 | Temperature unit | `0` and `1` are the two units, `2` / `3` no request (exact) |
| 2 | 7 - 1 | Language | language slot number (e.g. 2 English UK, 3 English US); `0` / `1` = no request. Taken only if this cluster has that language (exact) |

- **`00` in byte 0 means "metric"** and overrides the EEPROM units (seen). Send `3C` when no change is meant.
- A change made in the cluster menu wins over `0x191` (exact).
- The units are echoed on [0x2FD](#0x2fd---settings-units-exhaust-request-cluster-tx) byte 2 bits 7 / 6, and a
  language change is answered on [0x123](#0x123---language-change-answer-cluster-tx) byte 0 bits 7 - 6 (exact).
- Missing frame: nothing changes, the cluster keeps its units and language (exact).

```
No request                                    3C 00 00
Metric distance, temperature unchanged        30 00 00
English US, units unchanged                   3C 00 06
```

# 0x193 - Nav / Phone / Language Buttons (cluster TX)

**Sample:** `00 06 00 00 00 00 00 00` (built from the firmware field table: language code 3, en-US, no buttons) · **Rate:** on event · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 (`0x80`) | Cancel waypoint | `1` request, answered on [0x2EE](#0x2ee---navigation-guidance) byte 3 bit 0 (exact) |
| 0 | 6 - 5 | Phone mute | `1` / `2` (exact) |
| 1 | 7 - 1 | Language | Ford language code, `3` = en-US (exact) |
| 4 | 7 - 5 | Phone call buttons | `1` swap call, `2` conference, `3` / `4` privacy (exact) |
| 5 | 3 (`0x08`) | Repeat prompt (nav) | (exact) |
| 5 | 2 (`0x04`) | Cancel route (nav) | (exact) |

- Event frames. If [0x2EE](#0x2ee---navigation-guidance) is missing the nav requests end when their timers run out
  (exact).

# 0x195 - List Request (cluster TX)

**Sample:** `00 00 00 00 4B 03 00 01` (built from the firmware field table: fetch the first page of list 3) · **Rate:** on event · **Status:** Cluster sends (from firmware). Don't send it

Page requests for the cluster's SYNC lists (phone book / call history style).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | List index | start position (inferred) |
| 2 - 3 | | Entry ID | `0xFFFF` in two request types (inferred) |
| 4 | 7 - 6 | Request kind | `1` fetch a page, `2` select an entry (exact) |
| 4 | 5 - 0 | Entry count | `11` per fetch, `0` for select (exact) |
| 5 | | List ID | `1 - 3` (exact) |
| 6 - 7 | | Fetch flag | `1` with fetch, `0` with select (exact) |

- No reply in time: all fields go back to 0 (exact). What the replies look like is open.

# 0x19A - MyKey Restrictions (cluster TX)

**Sample:** `00 00 00 02 40` (built from the firmware field table: Do Not Disturb and 911 Assist not forced) · **Rate:** 1000 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 | 2 - 1 | Do Not Disturb | `1` not forced, `2` forced by MyKey (exact) |
| 4 | 7 - 6 | 911 Assist | `1` not forced, `2` forced by MyKey (exact) |

- The analog bench frame `00 00 00 02 40` in [database.md](database.md) decodes the same way: "not forced" for
  both.

# 0x1D6 - Phone Call Acknowledge

**Sample:** `00 00 00 00 20 00` (built from the firmware field table: active call request acknowledged) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 6. Needs option 191 (phone pages) and the [0x2E3](#0x2e3---infotainment-state) link.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 4 | 5 - 4 | Active call acknowledge | `2` acknowledges the cluster's active call request on [0x287](#0x287---phone-call-buttons-cluster-tx) byte 0 bits 3 - 2 (exact) |

# 0x1E6 - Audio Source Response

**Sample:** `00 00 04 03` (built from the firmware field table: source active, tuner band 1) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 4. The head unit's answer to a source request from the cluster.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 3 - 1 | Source state | `2` / `4` / `5` active, `3` changing, `1` idle (exact) |
| 3 | 3 - 0 | Current source | `3` tuner band 1, `7` band 0, `8` type 1 (likely SiriusXM), `9` / `10` bands 2 / 3, `11` type 2 (likely SYNC media) (exact codes, inferred names) |
| 0 - 1 | | Ends a [0x1F2](#0x1f2---audio-request-2-cluster-tx) request | byte 0 bits 4 - 2 = 1, byte 0 bits 7 - 5 = 1 - 3, byte 1 bits 3 - 0 echoing the requested code, byte 1 bit 4 = 0 (exact) |

# 0x1E9 - Station Text

**Sample:** `4B 49 53 53 20 46 4D 20` (built from the firmware field table: text "KISS FM ") · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 7 | | Text | eight ASCII characters, byte 0 first, shown on the cluster's audio page; a `00` byte ends the text early (exact) |

- Likely the radio station name (RDS) (inferred). Missing: the last text stays (seen).
- Needs option 189 (entertainment pages) and the [0x2E3](#0x2e3---infotainment-state) link (exact). What sets the
  cluster's "text valid" flag is open, so a bare `0x1E9` may not show.

# 0x1EE - Audio Request (cluster TX)

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: no action) · **Rate:** on event · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 0 | Tuner action | action code from the cluster's audio screen (exact field; codes open) |
| 0 | 7 - 4 | Audio action | action code (exact field; codes open) |
| 3 | 7 - 3 | Argument | preset / source index (inferred) |

- Each field is written inverted, then real, so every press sends (exact). Ended by
  [0x2F5](#0x2f5---audio-source--band) / [0x2DC](#0x2dc---audio-preset-echo) or a timer (exact).

# 0x1F2 - Audio Request 2 (cluster TX)

**Sample:** `0C 0E 00 00 00 00 00 00` (built from the firmware field table: idle) · **Rate:** on event · **Status:** Cluster sends (from firmware). Don't send it

- A second audio request, only for two audio actions (10 and 11), idle `0C 0E 00 ...`. Cleared by
  [0x1E6](#0x1e6---audio-source-response) (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Request | `0C 0E` idle (exact) |

# 0x202 - Vehicle Speed

**Sample:** `00 00 00 00 40 00 27 10` (built from the firmware field table: 100.00 km/h, quality good) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 6 - 7 | | Speed | 16 bit, **0.01 km/h per count**, so raw = mph x 160.9344 (exact, seen) |
| 4 | 6 - 5 | Speed quality | `2` or `3` = good, speed accepted; `0` or `1` = bad, speed held and after 5 s faulted (exact, seen). So `0x6_` and `0x4_` both work |
| 0 | 4 - 2 | Gear range code | `3` = reverse: backup chime (option 27) and reverse flag (exact) |
| 0 | 7 - 5 | Warning code | values 2 / 4 / 5 / 6 each light one bit of a warning slot (exact; which message is open) |

- Needs Run. Missing: U0100-00 (lost communication with the PCM) and speed reads 0 (exact).
- The needle reads about 1 % high on a normal cluster (about 4 % with option 120): the Ford over-read is built into
  the firmware (exact, seen). Dial face and unit come from option 1 (metric) and option 3 (180 mph / 300 km/h)
  (exact, seen).
- Byte 4 bit 0 and byte 5 aren't read (exact). Track Apps uses its own speed on [0x076](#0x076---track-apps-g-meter-and-speed).

```
0 km/h, quality good                           00 00 00 00 40 00 00 00
60 mph (9656), quality good                    00 00 00 00 40 00 25 B8
100.00 km/h, quality good                      00 00 00 00 40 00 27 10
Reverse (backup chime), 0 km/h                 0C 00 00 00 40 00 00 00
```

# 0x204 - Engine Speed (RPM)

**Sample:** `00 00 00 01 90` (built from the firmware field table: 800 rpm) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 5.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 - 4 | byte 3 bits 4 - 0, byte 4 | Engine speed | 13 bit, **rpm = raw x 2** (exact, seen) |

- The 2020 MCU reads only these 13 bits. It doesn't read the pedal bytes (exact).
- It is the tach source **only while EEPROM option 279 (Tach Signal, as-built byte `0x07F` bit 3) is clear**. With
  option 279 set the cluster takes the tach from [0x42F](#0x42f---engine-speed-alternate-source) instead (exact,
  seen). See [0x204 / 0x42F - Tach source](#0x204--0x42f---tach-source).
- Below 190 counts (~380 rpm) the chime logic treats the engine as not running; below 200 (~400 rpm) the
  engine-not-running warning path runs (exact).
- Missing (while it is the source): the needle parks at a fallback point and U0100-00 is logged (exact).

```
800 rpm                                       00 00 00 01 90
3000 rpm                                      00 00 00 05 DC
```

# 0x20A - Drive Mode Confirm (Powertrain)

**Sample:** `01 04 00 00 00 00 00 02` (built from the firmware field table: Normal reported back, update bits set) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 0 (`0x01`) | Update bit | `1` = counts (exact) |
| 0 | 7 - 4 | Drive mode from the powertrain | `0` Normal, `1` Sport+ / Track (and Drag Strip on a manual), `2` Snow / Wet (inferred name), `7` Drag Strip on an automatic, `15` invalid (exact) |

- Missing: "Drive Mode Selection Not Available", and U0401-82 (exact). One of the five confirmations a drive mode
  change waits for, see [Drive mode handshake](#0x20a--0x38a--0x431--0x46a--0x27a---drive-mode-handshake).

```
Normal reported                                 01 04 00 00 00 00 00 02
Sport+ reported                                 11 04 00 00 00 00 00 02
Drag Strip reported (automatic)               71 04 00 00 00 00 00 02
```

# 0x213 - Electric Park Brake Lamps

**Sample:** `00 00 00 00 20` (built from the firmware field table: EPB lamp on) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 5. Only with option 50 = 1 (electric park brake). A Mustang with a hand brake gets its brake lamp from
[0x3C3](#0x3c3---body-lighting-status) byte 2 bit 4 instead.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 | 6 - 5 | Brake warning lamp | `0` off, `1` on, `2` / `3` two flash patterns (exact; patterns inferred) |
| 4 | 6 - 5 | EPB lamp | same coding (exact). Off when option 50 is clear |

- Missing with option 50: EPB lamp on (exact). No screen change with a normal EEPROM (seen).

# 0x219 - Entertainment Display Flags

**Sample:** `01` (built from the firmware field table: first display flag set) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 1.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 2 - 0 | Status code A | `1 - 3` set one entertainment display flag (exact; meaning open) |
| 0 | 7 - 6 | Status code C | `2` and `3` set two other display flags (exact; meaning open) |

- Missing: the cluster takes the same flags from [0x2D8](#0x2d8---audio-status) / [0x2DC](#0x2dc---audio-preset-echo)
  (exact). No screen change in the emulator (seen).

# 0x220 - Chime Request, Channel 1 (cluster TX)

**Sample:** `10 01 00 00 00 00 00 00` (exact: idle) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 4 - 3 | Request mode | `2` idle, `1` a chime playing in repeat mode (`08 01`) (exact) |
| 1 | | Request state | `1` idle (exact) |
| 0 - 7 | | Chime data | while a chime plays, the bytes come from that chime's ROM table entry (exact: idle / repeat state; inferred: what the table bytes mean) |

- This is a chime request, not blind spot. [0x223](#0x223---chime-module-feedback) byte 2 = 1 ends it (exact).

# 0x223 - Chime Module Feedback

**Sample:** `10 00 00 00 00` (built from the firmware field table: chime path available, nothing playing) · **Rate:** missing after 5.0 s · **Status:** From firmware

From the module that plays the cluster's chimes when the cluster isn't its own chime generator (option 32).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 5 - 4 | Chime path status | `1` = available; other values: the cluster falls back (exact) |
| 0 | 2, 3 | Per channel flags | channel 1 / channel 2 (inferred) |
| 2 | | Channel 1 status | `1` = chime done; stops re-sending the [0x220](#0x220---chime-request-channel-1-cluster-tx) request (exact) |
| 4 | | Channel 2 status | `1` = chime done (exact) |

- All zeros reads as "nothing playing", which is why zeros work (exact).
- The 2020 DTC table has no U0184 (exact), so `0x223` isn't needed to clear that code on this cluster.

# 0x225 - Chime / MyKey Audio Status (cluster TX)

**Sample:** `20 02 00 00 00 00 00 00` (exact: boot payload) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 6 - 5 | Audio mute request | `1` none, `2` mute (MyKey with option 80, or Belt-Minder) (inferred) |
| 0 | 4 - 2 | Chime level | highest attribute of the active chimes (inferred) |
| 0 | 0 (`0x01`) | Chime starting | (inferred) |
| 1 | 7 - 6 | Chime request state | `1` serviced, `2` not serviceable, i.e. no chime module (inferred) |
| 1 | 1 - 0 | MyKey volume limit | `1` none, `2` limit, option 84 (exact) |
| 2 | 7 (`0x80`) | Chime generator state | with option 32 = 0, a closure change in Off asks the chime module for a sound (exact bit, purpose inferred) |

- That explains the analog bench's `20 81` in [database.md](database.md) (inferred).

# 0x263 - AWD / 4x4 Fault

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: no fault) · **Rate:** missing after 5.0 s · **Status:** From firmware

Expected only when option 5 (2WD / 4x4 / AWD) says 4x4 or AWD (exact). Not fitted on a Mustang, so a normal EEPROM
ignores it.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 0 (`0x01`) | Driveline fault | `1` = AWD / 4WD / 4x4 malfunction warning, text by options 5 / 258 (exact) |

- Missing also raises that warning, with option 5 set (exact). No screen change with a normal EEPROM (seen).

# 0x274 - Rev Match Status

**Sample:** `00 00 00 05 18 80` (built from the firmware field table: rev match available, update bits set) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 6. Needs option 375 and a manual (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 4 | 5 - 4 | Rev match state | `1` / `2` available, `0` / `3` not (exact) |
| 4 | 3 (`0x08`) | Update bit | `1` = counts. Timing out sets U0401-82 (exact) |

- The rev match request goes out on [0x3C8](#0x3c8---mycolor-drive-mode-launch-requests-cluster-tx) byte 7 bits
  2 - 1 (inferred values).

# 0x277 - Memory Preset Command

**Sample:** `58 00 00 00 04` (built from the firmware field table: recall preset 1, update bits set) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 5. Option 238 (MyColor).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 4 | Memory preset command | `1 - 3` save preset 1 - 3, `5 - 7` recall preset 1 - 3, `4` a warning only (exact) |
| 0 | 3 (`0x08`) | Update bit, preset command | (exact) |
| 4 | 4 - 3 | Warning | a 3 way warning (exact; texts open) |
| 4 | 2 (`0x04`) | Update bit, warning | (exact) |

- On recall the cluster sends the stored MyColor levels again on
  [0x3C8](#0x3c8---mycolor-drive-mode-launch-requests-cluster-tx) (exact).

# 0x27A - Exhaust Mode Status

**Sample:** `55 00 00 25` (built from the firmware field table: Normal), `53 00 00 25` (Quiet), `57 00 00 25` (Sport), `59 00 00 25` (Track) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 4. Needs option 356 (active exhaust).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 0 (`0x01`) | Update bit, exhaust mode | `1` = counts (exact) |
| 0 | 3 - 1 | Exhaust mode in effect | `0` not available, `1` Quiet, `2` Normal, `3` Sport, `4` Track, `5` / `6` invalid, `7` invalid for the display but "any mode" for the drive mode check (exact) |
| 0 | 4 (`0x10`) | Update bit for byte 3 bit 7 | (exact) |
| 3 | 7 (`0x80`) | Indicator request | one telltale (inferred) |

- Shown in the Exhaust menu and used to confirm drive mode changes. With an invalid mode the cluster asks again on
  [0x2FD](#0x2fd---settings-units-exhaust-request-cluster-tx) (exact).

# 0x287 - Phone Call Buttons (cluster TX)

**Sample:** `04 00 00 00 00 00 00 00` (built from the firmware field table: active call request) · **Rate:** 200 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 2 | Active call request | `1` requested, probably hang up (exact field, inferred meaning) |
| 2 | 7 - 6 | Incoming call reply | `1` / `2`, likely reject / accept (inferred) |

- Held until [0x2D0](#0x2d0---phone-call-state) / [0x1D6](#0x1d6---phone-call-acknowledge) confirm, a timer runs out
  or power drops (exact).

# 0x2A1 - Phone Key

**Sample:** `45 00 00 00 10 00` (built from the firmware field table: slot 1 tag E, state 1 = Phone held) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 6. Four slots of an ASCII tag in bytes 0 - 3, each with a 4 bit state in bytes 4 - 5 (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 3 | | Slot tags | ASCII; tag `E` (`0x45`) is the Phone key (exact) |
| 4 - 5 | | Slot states | 4 bits per slot, slot 1 = byte 4 bits 7 - 4. State `1 - 4` with tag `E` = Phone held (exact) |

- The display gets it as `vehicle_hk_phone` (exact).

# 0x2D0 - Phone Call State

**Sample:** `00 00 00 06` (built from the firmware field table: call state 6, caller record 1) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 4. Needs option 191 (phone pages) and the [0x2E3](#0x2e3---infotainment-state) link.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 | 4 - 0 | Call state | picks the phone page and caller record: 2, 3, 5, 6, 7, 9, 10, 12 - 16 do something, others clear. `6` shows caller record 1, `14` caller record 2 (exact; state names open) |
| 1 | 2 - 0 | Phone status | every value but 4 sets phone status bit 0 (exact) |
| 1 | 5 - 4 | Mute acknowledge | acknowledges the cluster's mute request on [0x193](#0x193---nav--phone--language-buttons-cluster-tx) byte 0 bits 6 - 5; `1` also sets phone status bit 1 (exact) |

- Where the caller name / number text comes from is open (no CAN message in the trace carries it).

# 0x2D5 - Radio Frequency

**Sample:** `00 00 00 00 01 F6 00` (built from the firmware field table: FM 101.1 with an FM band), `00 00 00 00 03 59 00` (AM 1010 with an AM band) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 4 - 5 | byte 4 bits 2 - 0, byte 5 | Tuner frequency | 11 bit. **FM: MHz = 76.0 + raw x 0.05** (max 108.0). **AM: kHz = raw + 153** (max 1710) (exact) |

- FM or AM comes from the band on [0x2F5](#0x2f5---audio-source--band) (exact). Shown on the cluster's radio page.
- A frequency change raises a "station changed" flag for a moment (exact).

# 0x2D8 - Audio Status

**Sample:** `80 00` (built from the firmware field table: radio page flag set) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 (`0x80`) | Radio page flag | `1` sets a radio page flag (exact; meaning open) |
| 1 | 2 - 1 | Display flags | fallback source for two entertainment flags while [0x219](#0x219---entertainment-display-flags) has never arrived: `2` / `3` set one each (exact) |
| 1 | 7 - 3 | Validity | `0x1F` = invalid, after a 5 s confirm (exact) |

# 0x2DB - CD Number

**Sample:** `00 00 00 00 00 00 00 05` (built from the firmware field table: number 5) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 6 - 7 | | CD number | likely the track; `0` hides it (inferred) |

# 0x2DC - Audio Preset Echo

**Sample:** `02 00 00 00` (built from the firmware field table: selection 2) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 4.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 4 - 0 | Selection number | likely the tuner preset the head unit echoes; when it matches the cluster's request on [0x1EE](#0x1ee---audio-request-cluster-tx) the request ends (inferred meaning, exact match logic) |

# 0x2E0 - Phone Call Info

**Sample:** `00 3C 00 00` (built from the firmware field table: value 60) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 4. A phone message on this cluster (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Phone value | 16 bit, shown on the phone screen, likely call duration (inferred) |
| 3 | 4 - 2 | Acknowledge | acknowledges [0x193](#0x193---nav--phone--language-buttons-cluster-tx) byte 4 bits 7 - 5 (`1 - 4`); `1` during two calls swaps the shown caller (exact) |

# 0x2E3 - Infotainment State

**Sample:** `03` (built from the firmware field table: link up), `00` (inactive) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 1.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 2 - 0 | Infotainment state | **`2 - 4` = link up**: the cluster keeps its entertainment, navigation and phone pages. `0`, `1`, `6`, `7` inactive; `5` inactive for the display, also a chime input (exact) |

- If no frame comes for 5 s, or the state leaves 2 - 4, those pages are cleared (exact). An APIM sending `02` is
  "link up".
- The pages also need options 189 (entertainment), 190 (navigation) and 191 (phone), as-built byte `0x069` bits
  5 / 4 / 3 (exact).

# 0x2E6 - CD Value

**Sample:** `00 00 00 0A` (built from the firmware field table: value 10) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 - 3 | | CD value | track or time; `0` hides it (inferred) |

# 0x2EE - Navigation Guidance

**Sample:** `00 0C 19 02 03 00` (built from the firmware field table: guidance on, value 12, 25 min, code 3) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 6. Needs option 190 (navigation pages) and the [0x2E3](#0x2e3---infotainment-state) link.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Distance value | 16 bit, `0xFFFE` / `0xFFFF` invalid (inferred: distance to the next turn or destination; unit open) |
| 2 | 5 - 0 | Minutes | `0 - 60` shown (inferred: ETA / time to go) |
| 2 | 7 - 6 | Nav state | copied to the screen (exact) |
| 3 | 0 (`0x01`) | Acknowledge | answers the cluster's "cancel waypoint" request on [0x193](#0x193---nav--phone--language-buttons-cluster-tx) (exact) |
| 3 | 1 (`0x02`) | Route guidance active | enables the nav entries of Info on Demand (inferred) |
| 3 | 7 - 3 | Hours | `0 - 24` shown (inferred) |
| 4 | 3 - 0 | Maneuver code | `0 - 13` shown (14 shown as 13), `15` invalid (exact; likely the turn arrow, inferred) |
| 4 | 5 - 4 | Guidance suppress | `2` clears the maneuver data (exact) |

- Invalid values must last 5 s before a field is hidden (exact).

# 0x2F4 - Chime Request, Channel 2 (cluster TX)

**Sample:** `10 01 00 00 00 00 00 00` (exact: idle) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 7 | | Chime data | same layout as [0x220](#0x220---chime-request-channel-1-cluster-tx) (exact) |

# 0x2F5 - Audio Source / Band

**Sample:** `10 00` (built from the firmware field table: FM), `40 00` (AM) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 2.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 4 | Source / band | `1`, `2`, `3`, `6` = FM; `4`, `5` = AM (exact) |

- Other codes, or `0x2F5` missing, turn the frequency off, so [0x2D5](#0x2d5---radio-frequency) isn't shown (exact).

# 0x2F7 - HD Radio Text

**Sample:** `00 00 4B 49 53 53 00 00` (built from the firmware field table: label "KISS") · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 - 5 | | Short label | 4 ASCII characters, an HD Radio / short station label (inferred) |
| 0 | 4 - 2 | Radio status code | feeds the HD Radio state machine, option 186 (inferred) |

# 0x2F8 - Preset Number (Older Source)

**Sample:** `00 03 00 00 00 00 00` (built from the firmware field table: number 3) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 | | Number | 8 bit preset / channel number, `0xFF` invalid; used when [0x2FC](#0x2fc---preset-number) says so (inferred) |

# 0x2FC - Preset Number

**Sample:** `00 00 00 00 00 00 C0 00` (built from the firmware field table: number 3) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 5 - 6 | byte 5, byte 6 bits 7 - 6 | Number | 10 bit preset / channel number (preferred source); `0x3FE` = use [0x2F8](#0x2f8---preset-number-older-source) instead (inferred) |

# 0x2FD - Settings, Units, Exhaust Request (cluster TX)

**Sample:** `0D 00 C2 38 FC A0 40 00` (seen: boot payload in the emulator, en-US, exhaust request Normal) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 2 | Language | Ford language code, `3` = en-US. `0C` = 3 (exact) |
| 0 | 1 - 0 | Clock | 12 / 24 h, from EEPROM option 47 or [0x090](#0x090---clock-format) (inferred) |
| 2 | 6 (`0x40`) | Distance unit | `0` miles, `1` km (exact) |
| 2 | 7 (`0x80`) | Temperature unit | metric flag (inferred) |
| 2 | 3 - 2 | Line lock request | Track Apps line lock (inferred) |
| 6 | 7 - 5 | **Exhaust mode request** | `1` Quiet, `2` Normal, `3` Sport, `4` Track. From the Exhaust menu, Quiet Start and drive modes (exact) |

# 0x30A - Driver Assist / Steering Requests (cluster TX)

**Sample:** `C7 00 00 00 00 00 00 00` (exact: reset default, blind spot 3, cross traffic 3, rear park aid 1, steering Normal) · **Rate:** 200 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 6 | Blind spot on / off | `0` on, `1` off, `3` not configured / module missing (inferred) |
| 0 | 3 - 2 | Rear park aid | (inferred) |
| 0 | 1 - 0 | Cross traffic alert on / off | as blind spot (inferred) |
| 1 | 2 - 0 | **Steering feel request** | `0` Normal, `1` Sport, `2` Comfort, `3` / `4` drive mode steering (Sport+ / Track) (exact) |

- A frame with byte 0 `D7` has bits 7 - 6 = 3 and bits 1 - 0 = 3: no blind spot / cross traffic modules (inferred).
- The steering feel request changes from the console button ([0x361](#0x361---console-buttons-fcim)) or the
  Steering Feel menu (exact).

# 0x318 - Trip Computer Value (cluster TX)

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: value 0) · **Rate:** 100 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 5 - 6 | byte 5 bits 5 - 0, byte 6 | Trip computer value | 14 bit, from the distance to empty / fuel economy task (inferred) |

- On the 2020 MCU this, not `0x353`, is the distance to empty like value (inferred). `0x353` is never written
  (exact).

# 0x331 - Key Warning

**Sample:** `00 00 00 00 60` (built from the firmware field table: Key Battery Low Replace Soon) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 5. The cluster reads only byte 4.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 4 | 7 - 5 | Key warning code | `1`, `2`, `3` = three pop-ups (exact); **`3` = Key Battery Low Replace Soon** (seen); texts for 1 and 2 open |

- A logged byte 4 of `02` is code 0, no warning (exact). Other modules may read the rest of the frame.

# 0x350 - Airbag Lamp Echo (cluster TX)

**Sample:** `40` (built from the firmware field table: airbag lamp on / default) · **Rate:** 1000 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 6 | Airbag lamp state | echo of [0x04C](#0x04c---airbag--seat-belts): `1` on / default, `2` flash (inferred) |

# 0x361 - Console Buttons (FCIM)

**Sample:** `00 00 00 00 00 00 00` (built from the firmware field table: released), `00 00 00 00 00 00 80` (drive mode button held), `00 00 00 00 00 00 40` (steering feel button held) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7. The cluster reads only byte 6.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 6 | 7 (`0x80`) | Drive mode button | `1` pressed; a 0 → 1 change is one press (exact) |
| 6 | 6 (`0x40`) | Steering feel button | `1` pressed, one press per 0 → 1 change; needs [0x431](#0x431---steering-mode-transmission-messages) valid (exact) |
| 6 | 4 (`0x10`) | Exhaust button | not read by the 2020 MCU (exact); the GT500 MCU reads it |

- Drive mode button: the first press opens the selector, each next press steps the mode, and the choice commits
  when the window runs out (seen: presses about 1 s apart cycle Normal, Sport+, Track, Drag Strip). Needs option 230
  and Run (exact).
- What the cluster sends back: exhaust mode on [0x2FD](#0x2fd---settings-units-exhaust-request-cluster-tx) byte 6
  bits 7 - 5, steering feel on [0x30A](#0x30a---driver-assist--steering-requests-cluster-tx) byte 1 bits 2 - 0, drive
  mode on [0x430](#0x430---odometer-settings-out-cluster-tx) byte 7 bits 5 - 2 and
  [0x3C8](#0x3c8---mycolor-drive-mode-launch-requests-cluster-tx) byte 2 (exact).
- Missing: drive mode selection unavailable, a press shows "Drive Mode Selection Not Available" (exact).

# 0x38A - Gear Lever / Drive Mode (Automatic)

**Sample:** `40 10 F0 07 08 C7 00 00` (built from the firmware field table: lever P, drive mode Normal, update bits set) · **Rate:** missing after 5.0 s, per signal · **Status:** From firmware

Powertrain status on automatics, one update bit per signal (mask in [Update bits](#update-bits)) (exact):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 0 | Lever position | `0` P (clears Transmission Not in Park), `1` R, `2` N, ...; `12 - 15` invalid. Update bit byte 0 bit 6 (exact) |
| 1 | 7 - 5 | Drive mode from the powertrain | `1` Sport+, `3` Track, `4` Drag Strip; confirms a drive mode change. Update bit byte 2 bit 6 (exact) |
| 0 | 5 (`0x20`) | High beam feature flag | with option 319 = 1, update bit byte 1 bit 4 (exact bit; what it lights is open) |
| 2 | 3 - 0 | Warning code | `4` lights a warning, update bit byte 5 bit 7 (exact) |
| 3 | 3 (`0x08`) | Warning request | one warning, update bit byte 3 bit 2 (exact) |
| 4 | 4 (`0x10`) | Memory Recall Not Permitted While Driving | update bit byte 4 bit 3 (inferred name) |

```
Lever P, Normal                               40 10 F0 07 08 C7 00 00
Lever P, Sport+ reported                      40 30 F0 07 08 C7 00 00
```

# 0x38D - Key / Start Messages, MyKey

**Sample:** `00 00 00 00 00 00 00` (built from the firmware field table: no messages) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7. Byte 4 bits 7 - 4 pick key / keyless pop-ups (exact). The byte 4 codes line up with the table in
[database.md](database.md). Four codes from the 2020 firmware that table doesn't have yet:

| Byte | Bits | Code | 2020 firmware |
|---|---|---|---|
| 4 | 7 - 4 = `2` | `0x20` | a key / start message of its own (exact) |
| 4 | 7 - 4 = `5` | `0x50` | a key message shown **only while the power mode is not on** (exact; try with `0x3B2` byte 0 `1_`) |
| 4 | 7 - 4 = `13` | `0xD0` | a key / start message of its own (exact) |
| 4 | 7 - 4 = `15` | `0xF0` | a key message shown **only while the power mode is not on** (exact) |

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 0, 7 - 4 | Key counts | `0 - 8` for the MyKey menu (inferred: admin / MyKey counts) |
| 3 | 1 - 0 | Two warnings | `1` / `2` each light one message (exact); likely steering lock messages (inferred) |
| 3 | 3 - 2 | Keyless backup slot status | with EEPROM option 100 (keyless) (exact) |
| 5 | 5 - 4 | MyKey in use | `1` = MyKey in use (exact) |

- The cluster waits for this frame's MyKey state before it trusts the key type from
  [0x3B2](#0x3b2---body-info-power-mode-doors-lamps). While `0x38D` is missing the MyKey state is unknown, and with
  EEPROM option 239 drive mode selection is blocked (inferred). That fits "sending `0x38D` at all shows Track Apps"
  in the other files. Track Apps hides while a restricted MyKey is in use (inferred).

# 0x3A6 - Side Radar A (Blind Spot / Cross Traffic)

**Sample:** `00 60 03 00 00 00 00` (built from the firmware field table: blind spot and cross traffic on / OK), `00 40 03 00 00 00 00` (blind spot active, no alert), `00 48 03 00 00 00 00` (blind spot alert on this side) · **Rate:** missing after 1.6 s · **Status:** From firmware

Left side radar (inferred). DLC 7. Same layout as [0x3A7](#0x3a7---side-radar-b-blind-spot--cross-traffic); the two
are combined, see [0x3A6 / 0x3A7 - Blind spot radars together](#0x3a6--0x3a7---blind-spot-radars-together). Blind
spot needs EEPROM option 118, cross traffic option 37.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 | 7 - 5 | Blind spot status | `3` on / OK, `2` active (alerts from byte 1 bits 4 - 3), `0` off, `4 - 7` fault (exact) |
| 1 | 4 - 3 | Blind spot alert | `1` alert on this side, `2` / `3` fault (exact) |
| 1 | 0 (`0x01`) | Cross traffic alert | `1` alert (exact) |
| 2 | 2 - 0 | Cross traffic status | `3` on / OK, `2` active, `0` off, `4 - 7` fault (exact) |
| 3 | 1 - 0 | Cross traffic alert state | `1` indication, `2` / `3` fault (exact) |

- **Timeout 1.6 s**, not 5 s (exact). Missing sets U0232-00 (side obstacle module A, pairing inferred).

# 0x3A7 - Side Radar B (Blind Spot / Cross Traffic)

**Sample:** `00 60 03 00 00 00 00` (built from the firmware field table: blind spot and cross traffic on / OK), `00 40 03 00 00 00 00` (blind spot active, no alert), `00 48 03 00 00 00 00` (blind spot alert on this side) · **Rate:** missing after 1.6 s · **Status:** From firmware

Right side radar (inferred). DLC 7.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 | 7 - 5 | Blind spot status | as [0x3A6](#0x3a6---side-radar-a-blind-spot--cross-traffic) (exact) |
| 1 | 4 - 3 | Blind spot alert | `1` alert on this side, `2` / `3` fault (exact) |
| 1 | 0 (`0x01`) | Cross traffic alert | `1` alert (exact) |
| 2 | 2 - 0 | Cross traffic status | as `0x3A6` (exact) |
| 3 | 1 - 0 | Cross traffic alert state | `1` indication, `2` / `3` fault (exact) |

- **Timeout 1.6 s** (exact). Missing sets U0233-00 (side obstacle module B, pairing inferred).

# 0x3AA - Park Aid Tones / Status

**Sample:** `00 00 03 00 00 00 00 00` (built from the firmware field table: no tones, system normal) · **Rate:** missing after 5.0 s · **Status:** From firmware

From the parking aid module (inferred).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 0 | Park aid tone, channel 1 | `0` none, `1 - 15` one of 15 tone patterns, likely the front sensors (exact; front inferred) |
| 0 | 7 - 4 | Park aid tone, channel 2 | `0` none, `1 - 15` tone pattern, likely the rear sensors (exact; rear inferred) |
| 1 | 0 (`0x01`) | MyKey park aid can't be turned off | `1` with a MyKey in use: "Park Aid Cannot be Deactivated" (exact) |
| 2 | 3 - 0 | Park aid system status | `3` normal; `8` rear and front blocked; `9`, `10` fault type warnings; `11`, `12` rear blocked (exact codes, inferred names) |

- Tones play only in Run (exact). The blockage warnings need options 347 / 348 (exact).
- Missing: tones stop, and with option 99 U0159-00 (parking aid) is logged (exact).

# 0x3AB - Rear Park Aid On / Off

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: rear park aid on) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 4 | 7 - 6 | Rear park aid state | `0` on, `1` off, `2` / `3` unavailable (greyed in Settings) (exact field, inferred names) |

- Mirrors the Rear Park Aid toggle in Settings, option 99 (exact). Missing: the toggle is greyed, and with option 99
  U0159-00 is logged (exact).
- The other 13 fields are never read (exact).

# 0x3B2 - Body Info, Power Mode, Doors, Lamps

**Sample:** `40 00 00 0F 10 00 00 00` (built from the firmware field table: Run, dimming step 15, admin key) · **Rate:** missing after 5.0 s · **Status:** From firmware

**Byte 0 bits 7 - 4 are the power mode** (exact). The whole cluster's power state comes from it:

| Byte | Bits | Power mode | What the cluster does |
|---|---|---|---|
| 0 | 7 - 4 = `1` (`0x1_`) | Off | Gauges and most telltales inactive. Turn arrows only as hazards (both bits set). Message timeouts aren't checked (exact) |
| 0 | 7 - 4 = `2` (`0x2_`) | Accessory | As Off for gauges and timeouts; hazards only (exact) |
| 0 | 7 - 4 = **`4` (`0x4_`)** | **Run** | Gauges and lamps work, every message's 5 s timeout is checked, lost-comm DTCs can set, each turn arrow shows on its own (exact) |
| 0 | 7 - 4 = `8` (`0x8_`) | Start (crank) | Treated as Run with a "starting" flag once a timer runs out (the timer restarts whenever the value isn't 8) (exact) |
| 0 | 7 - 4, anything else | Off | (exact) |

- Until the first `0x3B2` arrives the cluster stays Off. If `0x3B2` stops, the power mode signal times out after
  5 s and the cluster goes to Off (exact).
- The ABS lamp only works in Run / Start (exact); most other telltales and gauges too (inferred from the timeout
  rule).

On the 2020 MCU every body bit comes from `0x3B2`; [0x3B3](#0x3b3---outside-temperature) carries only the outside
temperature. The rest of the frame:

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 0 (`0x01`) | Trunk ajar | `1` open (seen) |
| 0 | 1 (`0x02`) | Rear fog lamp | `1` on, EEPROM option 107 (exact) |
| 0 | 3 - 2 | Park / position lamps | `0` off, `1` on, `2` / `3` invalid. Telltale with option 119, only in Run / Start (exact) |
| 1 | 2 (`0x04`) | Wrench input | keep `0` (exact, see [Wrench / See Manual inputs](#wrench--see-manual-inputs)) |
| 1 | 3 (`0x08`) | Remote start active | feeds the remote start state machine; with option 141 the needles stay parked until it is done (inferred name, exact effect) |
| 1 | 5 - 4 | Key warning | `1` = one key warning, depends on the keyless / shift by wire options (exact bits, text open) |
| 1 | 7 - 6 | Day / night | used only when byte 3 is `0` or faulted: `1` day (step 18), other night (step 12) (inferred) |
| 2 | 2 - 1 | Dimmer switch activity | a new non-zero value shows the dimming pop-up again (inferred) |
| 3 | | Dimming step | `0` none (day / night fallback), **`1 - 12` night, `13 - 18` day**, `19+` invalid (then 12 after 5 s) (exact). A change shows the Dimming pop-up (seen) |
| 4 | 3 (`0x08`) | Right turn arrow | `1` on. In Off / Accessory only shown when both arrows are on (hazard) (exact, seen) |
| 4 | 7 - 4 | Key type | `1` admin key, `2` MyKey (option 80), latched once valid until power down (exact) |
| 6 | 0, 1 | Two more closure switches | chime inputs, nothing on screen (seen); which door is open (inferred) |
| 6 | 5 - 2 | Vehicle life mode | **keep `0`**. `1` shows Factory Mode Contact Dealer (seen); `3` another special mode that mutes chimes (exact) |
| 6 | 6 (`0x40`) | Left turn arrow | as the right one (exact, seen) |
| 7 | 0 (`0x01`) | Front fog lamps | `1` on, option 55, Accessory / Run / Start (exact, seen) |
| 7 | 1 (`0x02`) | Closure change detector | the real car sends `1` (inferred meaning) |
| 7 | 2 (`0x04`) | One more closure switch | nothing on screen (seen) |
| 7 | 3 (`0x08`) | Hood ajar | `1` open (seen) |
| 7 | 4 (`0x10`) | Passenger door ajar | `1` open (seen) |
| 7 | 5 (`0x20`) | Driver door ajar | `1` open (seen) |

- Doors, hood and trunk sit at the same byte / bit as in the analog `0x3B3` byte 7 / byte 0 table in
  [database.md](database.md), so on a 2020 cluster send them on `0x3B2` (exact for the 2020 MCU).
- The BCM blinks the turn bits itself; toggle them at the flasher rate.
- Byte 4 bits 7 - 4 = `2` (`0x20`) is MyKey: Track Apps hides while a restricted MyKey is in use, with MyKey set up
  (option 80) (inferred).
- Not read by the 2020 MCU: byte 5, byte 6 bit 7, byte 1 bits 1 - 0, byte 7 bits 7 - 6 (exact). The parking brake
  comes from [0x3C3](#0x3c3---body-lighting-status) byte 2 bit 4 or [0x213](#0x213---electric-park-brake-lamps).
- A driver (or passenger) door opening cancels remote start (exact for the driver door, inferred for the passenger
  door).
- Missing frame: all door, fog and mode lamps clear, dimming falls back to step 12, and U0140-00 (BCM) is logged
  (exact).

```
Power mode Off                                 10 00 00 0F 10 00 00 00
Power mode Accessory                           20 00 00 0F 10 00 00 00
Power mode Run                                 40 00 00 0F 10 00 00 00
Power mode Start (crank)                       80 00 00 0F 10 00 00 00
Driver door ajar (Run)                         40 00 00 0F 10 00 00 20
Driver + passenger + hood + trunk ajar         41 00 00 0F 10 00 00 38
Left turn arrow (Run)                          40 00 00 0F 10 00 40 00
Right turn arrow (Run)                         40 00 00 0F 18 00 00 00
Hazards (Run)                                  40 00 00 0F 18 00 40 00
Hazards (Off, the only arrows shown in Off)    10 00 00 0F 18 00 40 00
Front fog + park lamps, night dimming 6        44 00 00 06 10 00 00 01
Night dimming step 5                           40 00 00 05 10 00 00 00
MyKey driver (Run)                             40 00 00 0F 20 00 00 00
Remote start active                            40 08 00 0F 10 00 00 00
```

# 0x3B3 - Outside Temperature

**Sample:** `00 00 00 00 78 00 01` (built from the firmware field table: 20 C, update bit set) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7. On the 2020 MCU `0x3B3` carries only these two fields (exact):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 4 | | Outside temperature | **`raw x 0.5 - 40 = °C`**; `0xFE` = sensor fault, `0xFF` = invalid (exact, seen) |
| 6 | 0 (`0x01`) | Update bit | **`1`** or the value isn't refreshed in Run (exact, seen) |

- The frost / ice warning comes on at **+4 °C or below** (raw `0x58` or less), and the cluster sends a frost flag
  on [0x430](#0x430---odometer-settings-out-cluster-tx) byte 5 bits 5 - 4 = 2 (exact).
- `0xFE` and `0xFF` each log their own outside temp DTC with options 94 / 85 (exact).
- Missing: no value shown ("--"), and the frost logic counts it as possible ice (exact).
- No turn signals or doors on this ID on the 2020 MCU: those are on [0x3B2](#0x3b2---body-info-power-mode-doors-lamps)
  (exact).

```
20 C / 68 F                                   00 00 00 00 78 00 01
0 C / 32 F                                    00 00 00 00 50 00 01
-5 C                                          00 00 00 00 46 00 01
```

# 0x3B4 - TPMS Status

**Sample:** `04 11 11 00 23 23` (built from the firmware field table: TPMS lamp on, all tyres normal, placard 35) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 6. Needs EEPROM option 133 (TPMS).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 4 | Message code | `1`, `2`, `3`, `5`, `6`, `7`, `10`, `13`, `14` each pick their own pop-up; **`4` = sensor relearn / training active** (only below a low speed); other values none (exact; texts open). Code 1 is also what a missing frame shows |
| 0 | 3 - 2 | Lamp | `0` off, `1` solid, `2` flashing, `3` invalid (after 5 s the lamp flashes, then stays solid) (exact) |
| 0 | 1 - 0 | Reset / relearn status | shown in Settings; forced to `3` while the frame is missing (exact) |
| 1 - 2 | | Tire status nibbles | four nibbles, `1 - 4` valid, `2` and `4` = warning colour; `0` or `5 - 15` start a 5 s fault debounce (the last good value is kept) (exact) |
| 4 | | Placard pressure, front | psi, valid `25 - 115` (exact range, inferred unit). A logged `20` = 32 psi |
| 5 | | Placard pressure, rear | same |

- The lamp and system status are separate fields (exact).
- A TPMS reset from the cluster menu goes out on [0x434](#0x434---fuel-mykey-cluster-tx) byte 2 bit 2 = 1 (exact).

# 0x3B5 - Tire Pressures

**Sample:** `00 F0 00 F0 00 F0 00 B4` (built from the firmware field table: 240 / 240 / 240 / 180 kPa) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Pressure 1 | 16 bit, **1 kPa per count**, `0xFFFD - 0xFFFF` invalid (exact) |
| 2 - 3 | | Pressure 2 | same (exact) |
| 4 - 5 | | Pressure 3 | same (exact) |
| 6 - 7 | | Pressure 4 | same (exact) |

- The firmware reads all four 16 bit pairs; values under 256 kPa just have a `00` high byte (exact). The cluster
  converts to psi / kPa / bar for the screen. Which pair is which wheel is open.
- Missing for 5 s: every tyre shows "no value" (exact).

# 0x3C3 - Body Lighting Status

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: all off) · **Rate:** missing after 5.0 s · **Status:** From firmware

From the BCM.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 1 (`0x02`) | High beams | `1` on, with EEPROM option 319 = 0 (exact). `0x_E` has it set, `0x_C` not |
| 0 | 0 (`0x01`) | Chime sequence enable | enables a 3 stage chime / pop-up sequence (exact bit, meaning open) |
| 2 | 4 (`0x10`) | Brake lamp request | `1` brake telltale on, in Run / Start (exact bit; parking brake applied, inferred) |
| 2 | 3 - 2 | Perimeter alarm status | `2` / `3` = the two menu states, `0` / `1` = none, option 97 (exact) |
| 3 | 3 (`0x08`) | Key in ignition | key-in chime, EEPROM option 68, Off / Accessory (inferred) |
| 3 | 5 (`0x20`) | Lamps-on reminder | chime in power mode Off (inferred) |
| 4 | 0 (`0x01`) | LED low beam fault | `1` fault telltale, EEPROM option 153 (exact) |

- Bytes 1 and 6 aren't read by the 2020 MCU (exact).
- **Send it even to keep things quiet**: missing frame turns on the brake telltale and the LED low beam fault
  telltale, and logs U0140-00 (BCM) (exact).

```
All off                                        00 00 00 00 00 00 00 00
High beams on (option 319 = 0)                 02 00 00 00 00 00 00 00
Brake telltale request                         00 00 10 00 00 00 00 00
LED low beam fault                             00 00 00 00 01 00 00 00
```

# 0x3C8 - MyColor, Drive Mode, Launch Requests (cluster TX)

**Sample:** `00 00 70 00 00 00 00 00` (built from the firmware field table: ambient preset 7, drive mode Normal) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 2 - 0 | Drive mode selected | `0` Normal, `1` Sport+, `2` Snow / Wet (inferred), `3` Track, **`4` Drag Strip** (exact) |
| 2 | 7 - 4 | Ambient light preset | `0` ambient off, `1 - 7` preset colour (exact). `0x70` = preset 7, drive mode Normal |
| 3 | 1 - 0 | MyColor primary is custom | `0` preset, `1` custom (exact) |
| 5 | 3 - 0 | MyColor primary preset | `0 - 7` (exact) |
| 4 - 5 | | MyColor custom colour | three 4 bit levels: byte 5 bits 7 - 4, byte 4 bits 3 - 0, byte 4 bits 7 - 4 (inferred) |
| 0 - 1 | | Ambient custom colour + dim | byte 0 bits 7 - 6 custom flag, byte 0 bits 1 - 0 dim level, colour levels in byte 1 and byte 0 bits 5 - 2 (inferred) |
| 6 | | Launch control rpm | `raw * 50` rpm (exact). `3C` reads 3000 rpm |
| 7 | 7 - 6 | Launch control on / off | `1` on (exact) |
| 7 | 5 - 3 | Damper (MagneRide) request | `0` Normal, `1` Sport, `3` Track, `4` Drag (exact) |
| 7 | 2 - 1 | Rev match request | (inferred) |

- On a memory recall ([0x277](#0x277---memory-preset-command)) the stored levels go out again (exact).

# 0x3D8 - Front Camera Status

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: no lane or auto high beam requests) · **Rate:** missing after 5.0 s · **Status:** From firmware

From the front camera (lane keeping / auto high beam) module (inferred).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Setting value | value word of a camera setting (inferred), see [Settings requests and replies](#0x3e0--0x3e3--0x3e6---settings-requests-and-replies) |
| 2 - 3 | | Setting ID | `08 07` lane assist mode, `08 0C` auto high beam, `08 08` driver alert, `08 0B` lane assist intensity, `08 06` lane assist related (exact) |
| 4 | 3 - 2 | Lighting assist state | `1 - 3` each one driver assist indication (inferred) |
| 4 | 0 (`0x01`) | Lane assist warning | (inferred) |
| 6 | 7 - 3 | Lane keeping display | left / right line colours by lane assist mode, option 70, Run / Start (inferred) |
| 6 | 2 - 1 | Lane keeping warning | `1` / `2` two warnings (inferred) |
| 6 | 0 (`0x01`) | Auto high beam telltale | `1` on, with option 320 = 1 (inferred) |

- Missing: lane display and auto high beam telltale clear, and U023A-00 (image processing module) is logged with
  options 70, 44, 16, 268 or 320 (exact).

# 0x3E0 - Settings Request (cluster TX)

**Sample:** `04 03 00 00 84` (built from the firmware field table: read auto lock) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

On the 2020 MCU these requests are **settings reads**. The cluster keeps a 33 entry table of personalization
settings (the Settings menu items that live in other modules) and asks for each one on `0x3E0`; the module that
owns it answers with the current value (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Setting ID | 16 bit, e.g. `04 03` auto lock, `07 01` Easy Entry / Exit (exact) |
| 2 - 3 | | Value to write | `00 00` for a read (exact) |
| 4 | 7 - 5 | Constant | always 4 (`0x80`) (inferred) |
| 4 | 4 - 2 | Request type | `0` idle, **`1` read** (`0x84` = 4 + read), `2` write (exact) |

- Setting IDs and which reply ID carries each: [Settings requests and replies](#0x3e0--0x3e3--0x3e6---settings-requests-and-replies).

# 0x3E1 - Settings Reply (Easy Entry / Exit)

**Sample:** `00 00 01 07 01 00 00 00` (built from the firmware field table: value 1 for setting `07 01`) · **Rate:** missing after 5.0 s · **Status:** From firmware

The words sit one byte later than on the other settings feeds (exact):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 - 2 | | Setting value | (inferred word order) |
| 3 - 4 | | Setting ID | `07 01` Easy Entry / Exit, option 49 (exact) |

- Missing sets U0256-00 (front controls interface module) with option 111 (exact). The only setting it carries is
  easy entry / exit, which points to a seat / memory module (inferred). See
  [Settings requests and replies](#0x3e0--0x3e3--0x3e6---settings-requests-and-replies).

# 0x3E3 - Settings Reply (BCM)

**Sample:** `00 01 04 03 00` (built from the firmware field table: auto lock, raw value 1) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Setting value | value of the setting named in bytes 2 - 3 (inferred word order) |
| 2 - 3 | | Setting ID | BCM settings, e.g. `04 03` auto lock, `04 0D` auto lamp delay (exact). Full list in [Settings requests and replies](#0x3e0--0x3e3--0x3e6---settings-requests-and-replies) |

- Missing sets U0140-00 (BCM) when any BCM setting option is on (exact), and the BCM settings fall back to their
  defaults (exact).

# 0x3E5 - Settings Reply (Driver Assist)

**Sample:** `00 01 08 02 00` (built from the firmware field table: FCW sensitivity, raw value 1) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Setting value | (inferred word order) |
| 2 - 3 | | Setting ID | `08 0F` FCW distance alert, `08 02` FCW sensitivity, `08 40` FCW active braking (exact) |

- Missing sets U023A-00 (image processing module) with option 53 = 1 or 2 (exact).

# 0x3E6 - Settings Reply (Wipers)

**Sample:** `00 01 04 07 00` (built from the firmware field table: courtesy wipe, raw value 1) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Setting value | (inferred word order) |
| 2 - 3 | | Setting ID | `04 07` courtesy wipe, `04 08` rain sensing (exact) |

- Missing sets U0212-00 (steering column module, which carries the wipers) with option 36 or 104 (exact).

# 0x3E7 - Settings Reply (Remote Start)

**Sample:** `00 01 05 03 00` (built from the firmware field table: remote start, raw value 1) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | | Setting value | (inferred word order) |
| 2 - 3 | | Setting ID | `05 03` remote start, `05 04` rear defrost, `05 01` / `05 02` driver / passenger seat, `05 09` climate, `04 0A` steering wheel (exact) |

- Missing sets U0256-00 (front controls interface module) with option 111 (Remote Start) (exact).

# 0x3EC - Ambient Profile Mode

**Sample:** `00 00 04 00 00 00 00 00` (built from the firmware field table: apply to every profile) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 2 - 0 | Profile mode | `4` = a menu change to ambient colour / zones / MyColor is copied into all four profile slots; other values only the active one (exact) |

- Needs option 238 (MyColor). Until the first frame the copy-to-all path is blocked (exact).

# 0x3F1 - SYNC Page Numbers

**Sample:** `00 00 00 01 00 02` (built from the firmware field table: value 1 and value 2) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 3 | | Value 1 | 32 bit; `0` = not present, otherwise shown (low 16 bits) on the SYNC page (exact; meaning open) |
| 4 - 5 | | Value 2 | 16 bit; `0` = not present (exact; meaning open) |

- Missing: the display keeps the last values (exact).

# 0x416 - ABS / Stability / Brake Lamps

**Sample:** `00 00 00 00 00 00 00` (built from the firmware field table: all lamps off), `00 00 00 00 00 02 40` (ABS steady + IVD / ESC steady), `00 20 00 00 00 00 00` (red BRAKE request) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7. From the ABS / ESC module.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 | 5 (`0x20`) | Brake system warning | `1` red BRAKE lamp, option 8 (exact) |
| 1 | 6 (`0x40`) | Stability (RSC / AdvanceTrac) warning | `1` warning lamp and chime, EEPROM options 52 / 130 (exact) |
| 4 | 1 - 0 | Hill start assist status | `0 - 3`, shown in Settings; forced to `3` when missing, EEPROM option 321 (exact) |
| 5 | 2 - 1 | Traction / stability (IVD / ESC) malfunction lamp | `0` off, `1` steady, `2` slow flash (2048 ms period), `3` fast flash (512 ms), option 130 (exact) |
| 6 | 7 - 6 | ABS lamp | `0` off, `1` steady, `2` fast flash (512 ms period), `3` slow flash (2048 ms), option 8 (exact) |

- The ABS lamp only works in power mode Run / Start (exact).
- Missing frame: ABS, traction and brake lamps on steady, and U0121-00 (ABS) is logged (exact).
- On the 2020 MCU the TC OFF telltale comes from [0x46A](#0x46a---damper-mode--tc-off-lamp) byte 2 bits 7 - 6, not
  from this frame. Byte 5 is the IVD / ESC malfunction lamp, not the "off" lamp (exact). Bytes 0, 2, 3 and 7 aren't
  read (exact).

```
All lamps off                                  00 00 00 00 00 00 00
ABS steady + IVD / ESC steady                  00 00 00 00 00 02 40
ABS fast flash                                 00 00 00 00 00 00 80
ABS slow flash                                 00 00 00 00 00 00 C0
IVD / ESC slow flash                           00 00 00 00 00 04 00
IVD / ESC fast flash                           00 00 00 00 00 06 00
Red BRAKE request                              00 20 00 00 00 00 00
```

# 0x421 - Engine Lamps

**Sample:** `00 00 00 00 00 00 00 00` (built from the firmware field table: engine lamps off) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 3 - 2 | Lamp request (check engine) | `0` off, `1` / `3` steady, `2` flashing (exact bits, inferred name) |
| 1 | 1 - 0 | Auto stop status | `1` / `2` = the two auto start-stop indications, EEPROM options 6 / 7 (exact) |
| 2 | 2 (`0x04`) | Warning (oil pressure) | `1` = warning after a debounce tied to engine speed; **forced on when `0x421` is missing** (exact bit, inferred name) |
| 7 | | Fuel economy / range model input | `0xFF` invalid (the model then uses 100) (exact) |

- That is why a missing `0x421` reads as Oil Pressure Low. A byte 2 of `0x50` / `0x58` has bit 2 clear (no
  warning); `FF` sets it (exact).
- Missing frame: check engine lamp on, auto stop lamp off, the auto stop menu shows "unavailable" (exact). That
  "lamp on when the PCM goes silent" is why byte 0 bits 3 - 2 is the likely check engine lamp (inferred).

```
Check engine lamp steady                       04 00 00 00 00 00 00 00
Check engine lamp flashing                     08 00 00 00 00 00 00 00
```

# 0x424 - PCM Fuel Warning Level

**Sample:** `00 00 00 00 00 00 00` (built from the firmware field table: level 0) · **Rate:** missing after 5.0 s · **Status:** From firmware

DLC 7. Used only with **option 310 (FuelLvl_PCM)** (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 3 - 1 | Fuel warning level | `0` / `1` / `2` go straight to the low fuel warning; `3 - 7` depend on internal state; missing forces level 2 (exact). Which level is which pop-up stage is inferred |

- It sets the low fuel warning, not the needle. See [Fuel level](#fuel-level).

# 0x42C - Cruise / ACC Status

**Sample:** `14 B0 84 30 00 00 15` (built from the firmware field table: every update bit set, cruise off, no warnings), `54 B0 84 30 00 00 15` (cruise standby, grey SET), `94 B0 84 30 00 00 15` (cruise engaged, green) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

DLC 7. Every signal group in this frame has its own **update bit**. A frame with the bit clear doesn't refresh
that group, so it times out after 5 s even while frames arrive (exact):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 4 (`0x10`) | Update bit, cruise / ACC state | **`1`** (exact) |
| 0 | 7 - 5 | Cruise / ACC state | `0` / `1` off, `2` / `3` standby (grey, SET), `4` / `5` engaged (green), `6` / `7` invalid (exact) |
| 0 | 2 (`0x04`) | Update bit for byte 0 bit 3 | `1` (`8C` has it, `88` doesn't) (exact) |
| 0 | 3 (`0x08`) | Warning (Occupant Sensor Blocked) | `1` on (exact bit; name from the analog bench in [database.md](database.md)) |
| 2 | 7 (`0x80`) | **Update bit for the byte 1 warning code** | **`1`**. Clear: after 5 s the code times out and its lamp is **forced on** (exact) |
| 1 | 3 - 0 | Warning code | `1`, `2`, `4`, `5`, `8` light that lamp (2, 4, 5, 8 add a second slot); `3`, `9`, `6`, `13`, `14` other lamps (exact) |
| 1 | 4 (`0x10`) | Update bit for byte 3 bits 1 - 0 | (exact) |
| 2 | 2 (`0x04`) | Update bit for byte 6 bits 7 - 5 | no lamp (exact) |
| 6 | 2 (`0x04`) | Update bit for byte 6 bit 3 | (exact) |
| 6 | 3 (`0x08`) | Warning (spin out message) | `1` on (exact bit; name from the analog bench) |

- **The battery icon: byte 2 bit 7.** "`FF` turns it off within 2 s, `00` brings it back 6 s later" on the analog
  bench is this update bit: with it clear the byte 1 code times out and the lamp it drives is forced on. So the lamp
  is very likely the red battery / charging lamp (mechanism exact, name inferred). Sweeping only `0x80` would
  confirm it.
- `FF` in byte 6 sets both bit 2 and bit 3, which is why it shows the spin out message (exact).
- On the 2020 MCU U0401-82 comes from other frames (see [DTC map](#dtc-map)); `0x42C` isn't mapped to it there
  (exact).
- The ACC display needs option 10 (exact). The ACC status group uses a shorter deadline: 125 counts (exact count,
  1.0 s inferred).

```
Cruise off, no warnings                        14 B0 84 30 00 00 15
Cruise standby (grey SET)                      54 B0 84 30 00 00 15
Cruise engaged (green)                         94 B0 84 30 00 00 15
```

# 0x42D - Inlet Air Temp

**Sample:** `00 02 78 00 00 00 00 00` (built from the firmware field table: 30 C) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 1 - 2 | byte 1 bits 1 - 0, byte 2 | Inlet air temp | 10 bit, **`raw x 0.25 - 128 = °C`**, `0x3FE` / `0x3FF` invalid. Gauge with option 217 (exact) |
| 1 | 3 - 2 | Lamp request | `1` lights a lamp (exact; name open) |
| 1 | 7 (`0x80`) | Lamp request | `1` lights another lamp (exact; name open) |

- Invalid values log U0401-00 (exact).
- Cylinder head temp is not on this ID on the 2020 MCU: it is [0x43E](#0x43e---afr-cht-boost-oil-pressure) byte 4
  (exact).

# 0x42F - Engine Speed (Alternate Source)

**Sample:** `00 00 00 00 00 00 01 90` (built from the firmware field table: 800 rpm) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 6 - 7 | byte 6 bits 4 - 0, byte 7 | Engine speed | 13 bit, **rpm = raw x 2** (exact) |

- The same 13 bit field as in [2018_digital_database.md](2018_digital_database.md). The 2020 cluster uses `0x42F`
  **only when EEPROM option 279 (Tach Signal, as-built byte `0x07F` bit 3) is set**; with it clear the tach comes
  from [0x204](#0x204---engine-speed-rpm) (exact, seen). Bullitt-style EEPROMs have option 279 set (seen).
- With option 279 set it also feeds the shift light and the engineering rpm (exact).
- A missing `0x42F` (in use) sets U0100-00 (exact).

```
800 rpm                                       00 00 00 00 00 00 01 90
3000 rpm                                      00 00 00 00 00 00 05 DC
```

# 0x430 - Odometer, Settings Out (cluster TX)

**Sample:** `05 00 00 00 00 00 00 00` (built from the firmware field table: automatic, odometer 0, drive mode request Normal) · **Rate:** 100 ms · **Status:** Cluster sends (from firmware). Don't send it

On the 2020 MCU `0x430` follows the DBC `Cluster_Info1` layout of the analog `0x430` in [database.md](database.md):
the odometer is in bytes 1 - 3 (exact field, inferred unit). Byte 0 `05` / `04`:

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 0 (`0x01`) | Transmission display flag | `1` on automatics, from [0x171](#0x171---gear--prnds) (exact) |
| 0 | 2 (`0x04`) | Constant | always `1` (exact) |
| 0 | 3 (`0x08`) | Auto idle shutdown (AEIS) setting | `1` on (exact) |
| 0 | 6 (`0x40`) | Copy of EEPROM option 1 | metric bit (exact) |
| 0 | 7 (`0x80`) | Engine running | from [0x167](#0x167---engine-run-state) (exact) |
| 1 - 3 | | Odometer | 24 bit, `0xFFFFFF` invalid (exact; unit open) |
| 4 | 3 (`0x08`) | Adaptive cruise selected | from [0x18A](#0x18a---adaptive-cruise--forward-collision-warning) (inferred) |
| 5 | 5 - 4 | Frost warning | `2` = outside temp at or below +4 °C (or invalid) (exact) |
| 5 | 1 - 0 | MyKey programming request | `1` create, `2` clear (inferred) |
| 6 | 6 - 5 | Hill start assist setting | `0` feature absent, `3` reset default (exact) |
| 6 | 3 - 2 | Perimeter alarm | `1` off, `2` on (exact) |
| 7 | 5 - 2 | **Drive mode request** | `0` Normal, `1` Sport+ / Track (and Drag Strip on a manual), `7` Drag Strip on an automatic (exact) |

- Drive mode handshake: see [0x20A / 0x38A / 0x431 / 0x46A / 0x27A - Drive mode handshake](#0x20a--0x38a--0x431--0x46a--0x27a---drive-mode-handshake).

# 0x431 - Steering Mode, Transmission Messages

**Sample:** `00 00 A0 87 08 C8 00 86` (built from the firmware field table: no message, steering mode Normal, every update bit set) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

Byte 3 and byte 4 each hold a **message number**, each with its own update bit (exact). The 2020 firmware numbers
the messages only as bits of one warning slot, so the names in the code table come from the analog bench table in
[database.md](database.md):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 | 2 (`0x04`) | Update bit for the byte 3 code | `1` = the code counts (exact) |
| 3 | 6 - 3 | Message code A | see below (exact) |
| 3 | 1 (`0x02`) | Update bit for the byte 4 code | `1` = the code counts (exact) |
| 4 | 7 - 5 | Message code B | see below (exact) |
| 5 | 2 - 0 | Steering mode status | for Info on Demand and the drive mode check: `0` Normal, `1` Sport+ / Track (`3` / `4` with option 210) (exact) |
| 5 | 3 (`0x08`) | Update bit, steering mode | (exact) |

| Code A (byte 3 bits 6 - 3) | Slot bit | Analog bench name | Code B (byte 4 bits 7 - 5) | Slot bit | Analog bench name |
|---|---|---|---|---|---|
| `0`, `3`, `4`, `8` | none | (nothing, `04`) | `0`, `7` | none | |
| `1` | 0 | Transmission Malfunction (`0C`) | `1` | 8 | Transmission Too Hot, Press Brake (`37`) |
| `2`, `5`, `9` | 1 | Transmission Service Required (`14`) | `2` | 9 | untested |
| `6`, `7` | 2 | untested | `3` | 10 | untested |
| `10` | 3 | untested | `4` | 11 | Transmission Overheat, Stop (`97`) |
| `11` | 13 | same bit as code B 5 | `5` | 13 | Transmission Warming Up (`B7`) |
| `12` | 4 | Transmission Adapt Mode (`E3`) | `6` | 12 | untested |
| `13` | 5 | Transmission Indicator Mode Lockup (`EF`) | | | |
| `14`, `15` | 6, 7 | untested | | | |

- `EB` / `EA` in byte 3 are code 13 too, but with bit 2 clear: on the 2020 MCU that code wouldn't count, and the
  Warming Up message comes from byte 4 = `B7` (code 5, with byte 3 bit 1 set) (exact). `E3` also has bit 2 clear,
  yet showed Adapt Mode on the analog bench, so the analog cluster may not check this update bit (inferred).
- A value stays until its group times out (5 s) (exact).
- Steering feel button presses on [0x361](#0x361---console-buttons-fcim) need this frame valid (exact).

```
No message, steering mode Normal               00 00 A0 87 08 C8 00 86
Code A 1 (Transmission Malfunction)            00 00 A0 8F 08 C8 00 86
Code B 5 (Transmission Warming Up)             00 00 A0 87 A8 C8 00 86
Steering mode Sport+ reported                   00 00 A0 87 08 C9 00 86
```

# 0x432 - Axle Temp, Brake / Axle Messages

**Sample:** `00 51 11 14 18 37 00 01` (built from the firmware field table: no requests, update bits set) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

Each signal group has its own update bit and 5 s timeout (exact).

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | byte 0, byte 1 bit 7 | Axle (rear diff) oil temp | 9 bit, gauge with option 223; the scale lives in a gauge curve (inferred, scale open) |
| 1 | 6 (`0x40`) | Update bit, axle temp | `1` = counts (exact) |
| 1 | 5 (`0x20`) | Brake Applied Power Reduced | `1` shows the message. Update bit byte 1 bit 4 (exact) |
| 3 | 3 (`0x08`) | Axle Over Temperature Reduce Speed | `1` shows the message. Update bit byte 3 bit 2 (exact) |
| 5 | 3 (`0x08`) | MyKey traction control message | `1` shows "Traction Control On - MyKey Setting", option 130 (exact bit, inferred name). Update bit byte 5 bit 2 |

- The `0x432` groups timing out set U0401-82; invalid values set U0401-00 (exact).

```
No requests                                    00 51 11 14 18 37 00 01
Axle Over Temperature Reduce Speed             00 51 11 1C 18 37 00 01
Brake Applied Power Reduced                    00 71 11 14 18 37 00 01
```

# 0x434 - Fuel, MyKey (cluster TX)

**Sample:** `00 00 00 00 00 FF F0 00` (built from the firmware field table: MyKey max speed off, other fields 0) · **Rate:** 100 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 - 1 | byte 0 bits 1 - 0, byte 1 | Fuel level (combined) | 10 bit, `0x3FF` = fuel fault (exact). About 48 is empty and 968 full (from the 16 bit filter values 3072 / 61952 shifted right by 6, exact curve) |
| 2 - 3 | byte 2 bits 1 - 0, byte 3 | Fuel sender B | 10 bit, after the tank curve (exact) |
| 6 - 7 | byte 6 bits 1 - 0, byte 7 | Fuel sender A | 10 bit (exact) |
| 2 | 2 (`0x04`) | TPMS reset request | `1` requested (exact) |
| 4 | 6 (`0x40`) | MyKey traction control | `0` locked on by MyKey, `1` not locked (exact) |
| 5 - 6 | byte 5, byte 6 bits 7 - 4 | MyKey max speed | about 0.1 km/h per count, **`0xFFF` = off**, 1271 = 80 mph default (exact) |
| 0 | 2 (`0x04`) | Odometer delta flag | see [0x166](#0x166---odometer-from-another-module) (inferred) |

- The analog bench frame `40 00 10 00 28 FF F0 00` in [database.md](database.md): bytes 5 - 6 `FF F_` = MyKey max
  speed off (exact).

# 0x43E - AFR, CHT, Boost, Oil Pressure

**Sample:** `00 00 00 93 96 56 30 00` (built from the firmware field table: AFR 14.7, CHT 90 C, boost +69.0 kPa, oil pressure 0) · **Rate:** missing after 5.0 s · **Status:** From firmware

Each gauge needs its EEPROM option and Run.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 3 | | Air fuel ratio | `raw / 10`, gauge with EEPROM option 213 (exact) |
| 4 | | Cylinder head temp | `raw - 60 = °C`, option 216 (exact) |
| 5 - 6 | byte 5, byte 6 bits 7 - 3 | Boost / vacuum | 13 bit, 0.1 kPa per count, **zero at 2068** (below = vacuum, above = boost), option 215 (exact). The readout uses 0.2961 inHg / 0.145 psi per kPa (exact factors) |
| 6 - 7 | byte 6 bits 1 - 0, byte 7 | **Engine oil pressure** | **10 bit** (not only byte 7), gauge with option 228; the scale lives in the gauge curve table (inferred, scale open) |

- Missing frame: the gauges go to their fallback positions and show "not available"; with a gauge enabled it counts
  toward U0100-00 (exact).

# 0x46A - Damper Mode / TC OFF Lamp

**Sample:** `19 04 22 09 80 00` (built from the firmware field table: update bits set, dampers Normal, TC OFF lamp off) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

On the 2020 MCU `0x46A` is the **selectable damper (MagneRide) status**. DLC 6. The analog and 2019 digital files
read bytes 3 and 5 as gateway presence bits; the 2020 MCU doesn't read them, and its DTC table has no U0146 (exact).
The TC OFF lamp in byte 2 agrees with the 2018 digital file.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 5 | Damper mode | `0` Normal, `1` Sport, `2` shown as Normal, `3` Track, `4` Drag, `7` invalid. Confirms drive mode changes with EEPROM option 210 (Selectable Dampers) (exact) |
| 0 | 4 (`0x10`) | Update bit, damper mode | `1` = counts. Group timing out sets U0401-82 (exact) |
| 1 | 7 - 3 | Damper warning | `1 - 5` each one message, other values none (exact) |
| 1 | 2 (`0x04`) | Update bit, damper warning | (exact) |
| 2 | 7 - 6 | **TC OFF lamp** | `0` off, `1` steady, `2` slow flash (2048 ms), `3` fast flash (512 ms), option 130 = 2 or 3, Run / Start (exact) |
| 2 | 5 (`0x20`) | Update bit, TC OFF lamp | (exact) |

- The 2018 file's "byte 1 bit 3 TCS off message" is damper warning value 1 on the 2020 MCU (exact).
- `0x46A` is not a wrench input on the 2020 MCU (inferred, see
  [Wrench / See Manual inputs](#wrench--see-manual-inputs)).

```
Dampers Normal, no warning, TC OFF lamp off    19 04 22 09 80 00
TC OFF lamp steady                             19 04 62 09 80 00
TC OFF lamp slow flash                         19 04 A2 09 80 00
TC OFF lamp fast flash                         19 04 E2 09 80 00
Dampers Sport                                  39 04 22 09 80 00
Dampers Track                                  79 04 22 09 80 00
```

# 0x46B - High Beam / Auto High Beam / Park Lock

**Sample:** `00 00 00 00 00 00 00 53` (built from the firmware field table: update bits set, all off), `00 00 00 00 00 00 00 73` (high beams on, option 319 = 2), `00 00 00 00 00 00 00 D3` (auto high beam on) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 7 | 7 (`0x80`) | Auto high beam active | `1` telltale on, with option 320 = 2 (exact) |
| 7 | 5 (`0x20`) | High beams on | `1` on, used instead of [0x3C3](#0x3c3---body-lighting-status) with EEPROM option 319 = 2 (exact) |
| 7 | 3 (`0x08`) | Park lock status | with option 167 (inferred) |
| 7 | 0, 4, 6 | Update bits | park lock, high beam, auto high beam (exact) |

- A timed out auto high beam group counts toward U0401-82 (exact).

# 0x470 - Line Lock, Transmission Temp

**Sample:** `00 00 C6 00 01 19 20 00` (built from the firmware field table: line lock ready, TOT group valid, update bits set) · **Rate:** missing after 5.0 s, per group · **Status:** From firmware

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 6 (`0x40`) | Update bit, transmission oil temp | **the only bit TOT needs**; `0xFF` works because it includes it (exact) |
| 3 | | Transmission oil temp | `0xFE` / `0xFF` invalid. Gauge with option 220, automatics only; the scale lives in a gauge curve (exact field, scale open) |
| 2 | 5 - 3 | Line lock state | `0` ready, `1` holding (brake percent shown), `2 - 4` other line lock screens, `5 - 7` none. Option 209 (exact) |
| 2 | 2 (`0x04`) | Update bit, line lock state | (exact) |
| 6 - 7 | byte 6 bits 4 - 0, byte 7 | Line lock held brake pressure | 13 bit, `0x1FFE` / `0x1FFF` invalid (exact; unit open) |
| 6 | 5 (`0x20`) | Update bit, line lock pressure | (exact) |

- The line lock group timing out (option 209) sets U0401-82 (exact).
- Bytes 0 - 1 are latched but never read (exact).
- The cluster's own line lock request goes out on [0x2FD](#0x2fd---settings-units-exhaust-request-cluster-tx) byte 2
  bits 3 - 2 (inferred values).

```
Line lock ready, TOT group valid               00 00 C6 00 01 19 20 00
Line lock holding                              00 00 CE 00 01 19 20 00
```

# 0x591 - Network Management (cluster TX)

**Sample:** `91 00 FF FF FF FF FF FF` (seen in the emulator) · **Rate:** 1000 ms while awake · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | | Node | `0x91` (exact) |
| 1 | | Control | `0` (exact) |

# Topics

## Timeouts and bad values

- **5 s timeout (exact).** Almost every received message, or each signal group inside one, has a deadline of
  625 ticks x 8 ms = **5.0 s**. Each frame reloads it. The side radars `0x3A6` / `0x3A7` use 200 ticks = **1.6 s**.
  This matches "must arrive at least every 5 s" in the other files for `0x14C`, `0x46A` and `0x3B2`.
- **Only in Run (exact).** Deadlines are only checked while `0x3B2` says Run (and the network is up). In Off or
  Accessory a missing message usually does nothing. Before the first frame most messages count as missing.
- **Bad values (exact).** An out of range value (e.g. `0xFE` / `0xFF` on a temperature) is ignored at once, the last
  good value stays, and after 1250 ticks x 4 ms = 5 s of bad values the signal is marked faulted.
- **Lost communication DTCs.** A timeout logs a DTC in the cluster (read with FORScan or UDS `0x19`), see
  [DTC map](#dtc-map).
- The real car's send periods were not read (only the cluster's own TX periods and its RX deadlines). Sending every
  message at least every 1 s is enough for the deadlines; 20 - 100 ms for speed, rpm and keys gives smooth needles
  (inferred).

## Update bits

Many frames carry one update bit per signal group. A frame with the bit clear doesn't reload that group's
deadline, so the group times out in Run even while frames arrive (exact). That is the mechanism behind "frames with
it clear don't count as received" in the other files (`0x14C` byte 3 bit 0, `0x46A` byte 3 bit 3) and `0x42C`
byte 0 bit 4. With these masks set, all seven drive mode "missing" flags clear in the emulator (seen).

OR these masks (byte 0 first) into a frame to set every update bit (exact):

| ID | Mask | Covers |
|---|---|---|
| `0x076` | `80 00 20 00 00 00 00` | byte 2 bit 5 = Track Apps speed valid |
| `0x077` | `70 00 00 00 00 85 41 00` | the five group valid bits |
| `0x108` | `00 06 0F 00 00 00 00 00` | byte 1 bit 2 = frame valid (brake test) |
| `0x14C` | `18 00 00 03 00 03 40 00` | byte 3 bit 0 = valid |
| `0x20A` | `01 04 00 00 00 00 00 02` | byte 0 bit 0 = drive mode confirm |
| `0x274` | `00 00 00 05 08 80` | byte 4 bit 3 = rev match state |
| `0x277` | `08 00 00 00 04` | byte 0 bit 3 preset command, byte 4 bit 2 warning |
| `0x279` | `00 00 00 00 00 00 40 82` | no visible use on the 2020 MCU |
| `0x27A` | `51 00 00 25` | byte 0 bit 0 = exhaust mode, byte 0 bit 4 |
| `0x38A` | `40 10 F0 07 08 C7 00 00` | byte 0 bit 6 = lever, byte 2 bit 6 = drive mode, others |
| `0x3B3` | `00 00 00 00 00 00 01` | byte 6 bit 0 = outside temp |
| `0x3CA` | `2B 01 15 00 C0` | no visible use |
| `0x3D0` | `01 0F 00 00 80` | no visible use |
| `0x42C` | `14 B0 84 30 00 00 15` | byte 0 bit 4 cruise state, byte 2 bit 7 warning code, others |
| `0x431` | `00 00 A0 87 08 C8 00 86` | byte 5 bit 3 steering mode, byte 3 bits 2 / 1 message codes, others |
| `0x432` | `00 51 11 14 18 37 00 01` | byte 1 bit 6 axle temp, byte 1 bit 4, byte 3 bit 2, byte 5 bit 2, others |
| `0x46A` | `19 04 22 09 80 00` | byte 0 bit 4 damper mode, byte 1 bit 2 damper message, byte 2 bit 5 TC OFF lamp |
| `0x46B` | `00 00 00 00 00 00 00 53` | byte 7 bits 0 / 4 / 6 |
| `0x470` | `00 00 C6 00 01 19 20 00` | byte 2 bit 2 line lock, byte 2 bit 6 TOT, byte 6 bit 5 line lock pressure |

Other frames use one plain valid value instead: `0x202` byte 4 bits 6 - 5 = 2 or 3, `0x156` byte 4 bits 1 - 0 = 3
(exact).

## DTC map

Which message sets which DTC on the 2020 MCU (the timeout of the message or group, unless noted; exact):

| DTC | Set by |
|---|---|
| U0100-00 lost comm PCM | `0x156`, the rpm source (`0x204` / `0x42F`), speed `0x202`, `0x43E` (with a gauge enabled) |
| U0401-82 invalid data PCM | `0x077` set speed group, `0x14C` (qualifier), `0x20A`, `0x274`, `0x432`, `0x46A` damper group, `0x46B`, `0x470` |
| U0401-00 invalid data PCM | `0x42D` inlet air `0x3FE` / `0x3FF`, invalid `0x432` / `0x470` values |
| U0121-00 lost comm ABS | `0x416`; `0x076` g meter axes (option 206) |
| U0415-82 invalid data ABS | `0x077` hill start group; `0x076` speed |
| U0420-82 invalid data PSCM | `0x077` steering lamp group, only with option 51 (EPAS) |
| U0140-00 lost comm BCM | `0x3B2`, `0x3C3`, `0x3E3` |
| U0151-00 lost comm RCM | `0x04C` |
| U0212-00 lost comm SCCM | `0x081`, `0x3E6` |
| U0159-00 lost comm parking aid | `0x3AA`, `0x3AB` (option 99) |
| U0199-00 lost comm door module | `0x3DE` (option 75, mirror autofold) |
| U023A-00 lost comm image processing module | `0x3D8`, `0x3E5` |
| U0256-00 lost comm FCIM | `0x3E1`, `0x3E7` (option 111) |
| U0232-00 / U0233-00 lost comm side obstacle A / B | `0x3A6` / `0x3A7` (pairing inferred) |

- The 2020 table has **no U0146 (GWM) and no U0184 (radio)**, so a 2020 cluster doesn't need `0x46A` / `0x223`
  for those two (exact).
- Zeros work as presence frames because a frame resets its message's timeout and zero values mostly mean "nothing
  to show" (inferred).

## Healthy car baseline

This set keeps a parked, running car free of lost communication lamps in the emulator with a normal car EEPROM
(seen; built from the field tables plus the update bit masks). Send each at 100 ms or faster.

```
0x3B2  40 00 00 0F 10 00 00 00   Run, dimming 15, admin key
0x3B3  00 00 00 00 78 00 01      outside air 20 C, update bit
0x202  00 00 00 00 40 00 00 00   0 km/h, quality good
0x204  00 00 00 01 90            800 rpm
0x42F  00 00 00 00 00 00 01 90   800 rpm (only read with option 279)
0x156  96 A0 00 00 03 00 00 00   coolant 90 C, oil 100 C, coolant valid
0x171  04 00 00                  lever P, lever valid (automatic)
0x167  20 00 00 00 00 00 00      engine running
0x179  00 00 00 00 10 00 00 00   byte 4 = 0x10
0x04C  00 50 00 00 00            airbag lamp off, belts not unbuckled
0x416  00 00 00 00 00 00 00      ABS / IVD / brake requests off
0x421  00 00 00 00 00 00 00 00   engine lamps off
0x3C3  00 00 00 00 00 00 00 00   body lighting
0x213  00 00 00 00 00            EPB / brake lamp off
0x077  70 00 00 00 00 85 41 00   cruise: groups valid, no set speed
0x42C  14 B0 84 30 00 00 15      ACC: groups valid, ACC off
0x14C  18 00 00 03 00 03 40 00   wrench lamp off, valid
0x38A  40 10 F0 07 08 C7 00 00   lever P, drive mode Normal
0x20A  01 04 00 00 00 00 00 02   drive mode Normal reported
0x431  00 00 A0 87 08 C8 00 86   steering mode Normal, no messages
0x46A  19 04 22 09 80 00         dampers Normal, no warnings
0x27A  55 00 00 25               exhaust Normal (only read with option 356)
0x432  00 51 11 14 18 37 00 01   axle / brake status, no requests
0x470  00 00 C6 00 01 19 20 00   line lock ready, TOT group valid
0x46B  00 00 00 00 00 00 00 53   lighting groups valid, all off
0x361  00 00 00 00 00 00 00      console buttons released
0x081  00 00 00                  wheel keys released
0x191  3C 00 00                  no units request (00 would force metric)
```

What drops look like, 3 s after the message stops in Run (seen):

- `0x156` stops: coolant needle to the fault position, hot engine lamp, **Engine Coolant Over Temperature**.
- `0x04C` stops: airbag lamp.
- `0x077` stops: **Steering Assist Fault Service Required**.
- `0x14C` stops: **See Manual**.

## EEPROM options that gate features

If a frame does nothing, check its option first (exact):

| Option | As-built | Gates |
|---|---|---|
| 1 | `0x020` bit 6 | metric (km/h, km, l/100 km, °C) |
| 2 | `0x020` bit 5 | transmission type; see the note in [0x171](#0x171---gear--prnds) |
| 3 | `0x020` bit 4 | 180 mph / 300 km/h dial |
| 80 | `0x052` bit 2 | MyKey |
| 133 | | TPMS |
| 189 / 190 / 191 | `0x069` bits 5 / 4 / 3 | entertainment / navigation / phone pages |
| 206 | `0x070` bit 5 | g meter |
| 209 | `0x070` bit 2 | Line Lock |
| 210 | `0x070` bit 1 | selectable dampers (MagneRide) |
| 213 / 215 / 216 / 217 / 220 / 223 / 228 | | AFR, boost, CHT, inlet air, trans fluid temp, axle oil temp, oil pressure gauges |
| 226 | `0x074` bit 7 | Launch Control |
| 230 | `0x074` bit 3 | drive mode selection |
| 279 | `0x07F` bit 3 | tach from `0x42F` instead of `0x204` |
| 356 | `0x092` bit 5 | active exhaust |

## 0x204 / 0x42F - Tach source

The 2020 MCU reads the same 13 bit rpm field (rpm = raw x 2) on both IDs and picks one by EEPROM option 279 (exact,
seen):

- Option 279 clear: tach from [0x204](#0x204---engine-speed-rpm) byte 3 bits 4 - 0 + byte 4.
- Option 279 set (Bullitt-style EEPROMs): tach, shift light and engineering rpm from
  [0x42F](#0x42f---engine-speed-alternate-source) byte 6 bits 4 - 0 + byte 7. With only `0x204` on the bus such a
  cluster's tach stays at 0 (seen).
- Sending both is safe (exact).
- The 2018 digital file finds `0x204` dead and `0x42F` working; that fits if that cluster has option 279 set
  (inferred).

## 0x20A / 0x38A / 0x431 / 0x46A / 0x27A - Drive mode handshake

1. **The button:** [0x361](#0x361---console-buttons-fcim) byte 6 bit 7, a 0 → 1 change is one press (exact).
2. **Availability:** needs option 230 and Run. Every module below must be present, with its update bits set, or a
   press only shows **Drive Mode Selection Not Available** (exact): `0x38A`, `0x20A`, `0x361`, `0x431`, `0x46A`
   (with option 210), `0x27A` (with option 356), `0x178` (with option 226).
3. **Request:** the cluster sends the mode on [0x3C8](#0x3c8---mycolor-drive-mode-launch-requests-cluster-tx) byte 2
   bits 2 - 0 and [0x430](#0x430---odometer-settings-out-cluster-tx) byte 7 bits 5 - 2 (exact).
4. **Confirmation:** it commits the mode only when every module reports it back (exact):

| Answer | Bits | Normal | Sport+ | Track | Drag Strip |
|---|---|---|---|---|---|
| `0x20A` | byte 0 bits 7 - 4 (+ update bit byte 0 bit 0) | 0 | 1 | 1 | 7 automatic / 1 manual |
| `0x38A` | byte 1 bits 7 - 5 | 0 | 1 | 3 | 4 (or 0) |
| `0x431` | byte 5 bits 2 - 0 (+ update bit byte 5 bit 3) | 0 | 1 (3 with option 210) | 1 (4 with option 210) | open |
| `0x46A` | byte 0 bits 7 - 5 (option 210) | 0 | 1 | 3 | 4 |
| `0x27A` | byte 0 bits 3 - 1 (option 356) | 2 | 3 | 4 | open; **7 = accepted for every mode** |

- If the answer never comes, the cluster falls back to Normal about 0.34 s later (seen).
- Snow / Wet (likely mode 2) and MyMode use the same functions; not checked in the emulator.

Answer Sport+ (automatic, no dampers, no active exhaust), built from the field tables:

```
0x20A  11 04 00 00 00 00 00 02
0x38A  40 30 F0 07 08 C7 00 00
0x431  00 00 A0 87 08 C9 00 86
```

## 0x3A6 / 0x3A7 - Blind spot radars together

- The two radars are combined: the cluster looks up the pair (`0x3A7` value, `0x3A6` value). Both `3` = on / OK,
  both `2` = active (the alert bits are used), both `0` = off. If the sides disagree, or one is missing for 1.6 s,
  the system shows a fault (exact).
- Send the same frame on both IDs and change one side for an alert, e.g. `00 48 03 00 00 00 00` on one ID and
  `00 40 03 00 00 00 00` on the other (built from the field table).
- A frame of `00 40 02 00 ...` is blind spot status 2 and cross traffic status 2: "active", no alert (exact).
- The cluster's blind spot / cross traffic on / off requests go out on
  [0x30A](#0x30a---driver-assist--steering-requests-cluster-tx) (inferred values).

## 0x3E0 / 0x3E3 / 0x3E6 - Settings requests and replies

About 0.8 s after start the cluster asks the other modules for their settings on
[0x3E0](#0x3e0---settings-request-cluster-tx). On the 2020 MCU these are **settings reads**: the cluster keeps a
33 entry table of the Settings menu items that live in other modules, and each owner answers with the current
value (exact).

- **Reply layout**: bytes 0 - 1 = value, bytes 2 - 3 = setting ID on `0x3E3`, `0x3E5`, `0x3E6`, `0x3E7` and `0x3D8`
  (exact for the ID word, inferred for the value word). A reply `00 01 04 xx 80` is value 1 for setting `04 xx`.
- On `0x3E1` the two words sit one byte later: value in bytes 1 - 2, ID in bytes 3 - 4, so a reply there looks like
  `00 00 01 07 01 00 00 00` (layout exact, module inferred).
- Each reply ID feeds only its own settings (exact):

| Reply ID | Setting IDs |
|---|---|
| `0x3E3` (BCM) | `04 0D` auto lamp delay, `04 03` auto lock, `04 04` auto unlock, `04 15` daytime running lamps, `04 02` / `04 01` remote window close / open, `04 19` / `04 1A` locking feedback audible / visual, `04 10` (unnamed), `04 11` mislock, `04 12` intelligent access, `04 05` remote unlock, `04 14` (park lock related), `04 06` remote start system and duration, `04 18` silent mode |
| `0x3D8` (front camera) | `08 07` lane assist mode, `08 0C` auto high beam, `08 08` driver alert, `08 0B` lane assist intensity, `08 06` (lane assist related) |
| `0x3E5` (driver assist) | `08 0F` FCW distance alert, `08 02` FCW sensitivity, `08 40` FCW active braking |
| `0x3E1` | `07 01` Easy Entry / Exit |
| `0x3E7` (remote start) | `05 03` remote start, `05 04` rear defrost, `05 01` / `05 02` driver / passenger seat, `05 09` climate, `04 0A` steering wheel |
| `0x3E6` (wipers) | `04 07` courtesy wipe, `04 08` rain sensing |
| `0x3DD`, `0x3DE`, `0x3DF` | no settings on this cluster: their IDs never match |

- So a `04 07` request going to `0x3E6` fits: on the 2020 MCU `0x3E6` is the wiper / rain sensor settings feed
  (its timeout sets U0212, the steering column module, which carries the wipers) (exact DTC, inferred module).
- **`07 01` is Easy Entry / Exit**, and its answer comes on `0x3E1` (exact). A reply with the ID in bytes 3 - 4 and
  a non-zero value is what the 2020 MCU expects there.
- **A setting that keeps being asked for**: the cluster keeps asking until it gets a value that is valid for that
  setting; auto lamp delay has many valid values (inferred).
- Most on / off items take raw 0 or 1; for many, raw 1 maps to menu status 0 and raw 0 to status 1. Status 2 is the
  default, "not reported yet" (inferred). Each item also needs its EEPROM option, e.g. auto lock = option 18
  (exact).
- A reply ID timing out puts its settings back to their defaults (exact). Which DTC each one sets is in the
  [DTC map](#dtc-map).

## Wrench / See Manual inputs

The wrench inputs differ by firmware. On the 2020 MCU they are (each must arrive at least every 5 s). In the 2020
firmware the wrench is lamp slot 6; its name comes from the analog bench in [database.md](database.md), not from the
firmware.

| ID | 2020 firmware | Label |
|---|---|---|
| `0x14C` | byte 2 bit 0 = 0, byte 3 bit 0 = 1 (qualifier) | exact |
| `0x3B2` | byte 1 bit 2 = 0 | exact |
| `0x179` | byte 2 bits clear | inferred |
| `0x168` / `0x169` | byte 2 bit 7 / byte 2 bit 3 clear, only with option 117 (Shift By Wire) | exact |
| `0x263` | only expected with EEPROM option 5 = 4x4 / AWD | inferred |

- `0x46A` is not a wrench input on the 2020 MCU (it is the damper message there), unlike the analog and 2019 digital
  readings (inferred: the wrench routine was traced, not fully decompiled).

## Fuel level

- **Fuel level is not on CAN** on this cluster. It reads two resistive float senders (saddle tank) on its own ADC
  (exact). Both are needed; the firmware combines them and flags a fault if one is out of range (exact).
- Low resistance = full (exact). On a bench, put a resistance or a DAC voltage on both sender inputs. The pull-up
  value and connector pins are open.
- `0x207` is not in the cluster's receive table (exact).
- [0x424](#0x424---pcm-fuel-warning-level) is a PCM low fuel warning level, used only with option 310 (exact).
- The cluster reports its own fuel reading on [0x434](#0x434---fuel-mykey-cluster-tx) (exact).

## Infotainment pages

The cluster's audio, navigation and phone pages need their option (189 / 190 / 191) and the link state on
[0x2E3](#0x2e3---infotainment-state) = 2 - 4 (exact).

- Audio: band on [0x2F5](#0x2f5---audio-source--band), frequency on [0x2D5](#0x2d5---radio-frequency), station
  text on [0x1E9](#0x1e9---station-text), short label on [0x2F7](#0x2f7---hd-radio-text), preset number on
  [0x2FC](#0x2fc---preset-number) / [0x2F8](#0x2f8---preset-number-older-source), CD numbers on
  [0x2DB](#0x2db---cd-number) / [0x2E6](#0x2e6---cd-value), SYNC page numbers on
  [0x3F1](#0x3f1---sync-page-numbers). The cluster's audio requests go out on [0x1EE](#0x1ee---audio-request-cluster-tx)
  / [0x1F2](#0x1f2---audio-request-2-cluster-tx).
- Navigation: [0x2EE](#0x2ee---navigation-guidance), with requests on
  [0x193](#0x193---nav--phone--language-buttons-cluster-tx).
- Phone: [0x2D0](#0x2d0---phone-call-state), [0x2E0](#0x2e0---phone-call-info), [0x1D6](#0x1d6---phone-call-acknowledge),
  the Phone key on [0x2A1](#0x2a1---phone-key), requests on [0x287](#0x287---phone-call-buttons-cluster-tx),
  `0x193` and [0x195](#0x195---list-request-cluster-tx).
- Open: what makes `0x1E9` text "valid" for display; where song title / artist and caller name text come from (no
  CAN message in the trace carries longer text).

## What the 2020 cluster sends

The 2020 MCU sends 48 IDs. Its transmit table gives each a fixed period (exact). Cyclic frames also go out on a
change, at most every 24 ms. IDs with their own section above have their fields there.

| ID | Period | 2020 firmware |
|---|---|---|
| `0x430`, `0x434`, `0x318` | 100 ms | see their sections |
| `0x346` | 100 ms | constant: three 2 bit fields at 3 (byte 3 bits 1 - 0, byte 4 bits 1 - 0, byte 6 bits 7 - 6), boot `00 00 00 03 03 00 C0 00` (exact) |
| `0x348` | 100 ms | never written (exact) |
| `0x287`, `0x30A` | 200 ms | see their sections |
| `0x18B`, `0x19A`, `0x220`, `0x225`, `0x2F4`, `0x2FD`, `0x350`, `0x3C8`, `0x3E0` | 1000 ms | see their sections |
| `0x591` | 1000 ms while awake | network management |
| `0x313`, `0x315`, `0x316`, `0x353`, `0x3FB`, `0x45A`, `0x30F` | 1000 ms | never written at run time, all zeros (exact) |
| `0x317` | 1000 ms | only its init bytes `03 FE` (exact) |
| `0x08C`, `0x45C` | 1000 ms | no signals written (exact) |
| `0x123`, `0x193`, `0x195`, `0x1EE`, `0x1F2` | on event | see their sections |
| `0x087`, `0x190`, `0x1F8 - 0x1FC`, `0x363`, `0x4A4` | on event | no run-time producer, so never sent in practice; only init bytes (exact) |
| `0x728` | on request | UDS answers to `0x720` / `0x7DF` (exact) |
| `0x2A4`, `0x2AF`, `0x2B9`, `0x2BE`, `0x2C7`, `0x4A1` | on request | ISO-TP channels, not cyclic frames (exact) |

- The analog cluster in [database.md](database.md) also writes `0x315`; the 2020 MCU never does (exact).

## Messages the 2020 cluster ignores

These are in the cluster's receive table and the MCU unpacks them, but **no code reads the result** on
LR3T-14C026-JE (exact; "no visible effect" also seen for the ones the emulator swept). Another Ford module, or
another cluster software, may use them.

`0x082` `0x086` `0x091` `0x163` `0x175` `0x17C` `0x17D` `0x188` `0x192` `0x194` `0x19B` `0x19D` `0x19E` `0x1E2`
`0x221` `0x224` `0x229` `0x22A` `0x261` `0x262` `0x279` `0x2D1` `0x2D2` `0x2D9` `0x2DA` `0x2DE` `0x2F6` `0x2F9`
`0x303` `0x30B` `0x30C` `0x312` `0x320` `0x347` `0x349` `0x360` `0x365` `0x366` `0x367` `0x368` `0x369` `0x3B8`
`0x3C9` `0x3CA` `0x3CD` `0x3D0` `0x3DD` `0x3DF` `0x3E2` `0x3EA` `0x3F0` `0x412` `0x420` `0x440` `0x443` `0x445`
`0x44E` `0x471` `0x474` `0x476` `0x477` `0x4A2` `0x4D0`

- `0x082`, `0x086`, `0x091` and some others still have a deadline: missing them can count toward a lost
  communication DTC, but nothing shows on screen (exact).
- `0x1E2`, `0x2DA`, `0x2DE` and `0x2F9` turn a field or two into internal flags that nothing reads (exact).
- `0x3DD` and `0x3DF` are settings reply IDs with no settings on this cluster (exact).
- `0x17E` is received but not read by the 2020 MCU; the GT500 MCU reads it (exact).
- **Parts of used messages are ignored too**, e.g. `0x3B2` byte 5, `0x156` bytes 2, 3 and 7, `0x202` byte 5,
  `0x470` bytes 0 - 1, `0x179` bytes 2 - 3 as a number, `0x361` all but byte 6 (exact).
- Accepted but routed nowhere: `0x001` `0x002` `0x006` `0x007` `0x100` `0x101` `0x102` `0x208` `0x402` `0x404`
  `0x408` (exact).

## Compared with the other files

Which of it may carry over (inferred, from where the bench results in the other files already fit the 2020 layout):

- **Likely the same on the analog**: `0x04C`, `0x076`, `0x077`, `0x14C`, `0x156`, `0x166`, `0x179`, `0x202`
  byte 4, `0x204`, `0x223`, `0x331`, `0x38D`, `0x3B4`, `0x3C3`, `0x416` bits, `0x421`, `0x42C`, `0x431`, `0x43E`,
  the `0x3E0` setting IDs, and the cluster's `0x220` / `0x2F4` / `0x3C8` / `0x434`.
- **Digital only, or read differently on the analog**: the body bits on `0x3B2` (the analog reads them from
  `0x3B3`), `0x3B3` (only the outside temp on the 2020 MCU), `0x46A` (damper message on the 2020 MCU), the
  `0x42F` / `0x204` choice (option 279), the `0x081` layout, `0x191`, `0x432`, `0x470`, and the IDs that are only
  in this file. The DTC map differs too: U0146 / U0184 don't exist on the 2020 MCU, and U0401-82 comes from other
  frames than `0x42C`. The `0x416` flash rates and the `0x202` scale may differ.
