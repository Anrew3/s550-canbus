# Shelby GT500 Digital Cluster (LR3V-14C026-BF)

**Label:** GT500 digital (LR3V)

The 2020+ Shelby GT500 12.4" digital instrument cluster, MCU software **LR3V-14C026-BF**. This file lists only
what differs from the 2020 GT / EcoBoost MCU (LR3T-14C026-JE). Everything not here reads the same as in
[2020_lr3t_database.md](2020_lr3t_database.md): timeouts, update bits, DTC map, and every ID without a section
below.

Where the data came from: customipc firmware trace of the MCU, 22 Nov 2026, plus an emulator running the real
firmware; nothing here is bench tested. Both MCUs were compared reader by reader for every received ID and producer
by producer for every sent signal. Labels: **exact** = read from the firmware code, **seen** = in an emulator
running the real firmware, **inferred** = likely. Example frames are built from the field tables, not captured.

## Modules

| Module | Part number | Notes |
|---|---|---|
| IPC (cluster) | LR3V-14C026-BF | firmware only, not bench tested |

## What stays the same

- The GT500 software is the 2020 software rebuilt four months later, with GT500 features added and some 2020
  features removed (exact).
- **Same 185 received and 48 sent CAN IDs** in the same table order (exact). The menu keys on `0x081` are the same
  bits (exact).
- The EEPROM option layout is identical, all 392 option definitions (exact). Options 88, 232, 362, 376 and 381 are
  read only by the GT500 MCU (exact).
- 87 of the 102 sent signal producers are identical (exact). Most received ID differences are only internal
  renumbering of the same CAN bits: `0x076`, `0x077`, `0x108`, `0x168`, `0x17C`, `0x20A`, `0x274`, `0x277`, `0x2DC`
  read the same (exact).

# 0x081 - Steering Wheel Keys (GT500 keys)

**Sample:** `20 00 00` (built from the firmware field table: steering feel key), `00 80 00` (MagneRide key) · **Rate:** missing after 5.0 s · **Status:** From firmware

The GT500 MCU reads the same key bits as LR3T (OK, Up, Down, Back, Menu, Media, Navi, HUD, Pony / Cobra menu) plus
two real keys (exact):

| Byte | Bits | Key | GT500 (LR3V) |
|---|---|---|---|
| 0 | 5 (`0x20`) | Steering feel | left switch steering feel key, with its selection pop-up (exact) |
| 1 | 7 (`0x80`) | MagneRide (damper mode) | damper mode key, with the DamperSelect pop-up: Normal / Sport / Track / Drag (exact) |

- On the 2020 MCU byte 0 `0x20` has no reader and byte 1 `0x80` has one, without these pop-ups (exact). On a
  non-GT500 cluster these two show the "Steering Feel / Damper Mode Selection Not Available" pop-ups, as the 2018
  bench table in [2018_digital_database.md](2018_digital_database.md) found.
- The Cobra menu (Pony menu on other cars) opens with byte 2 `0x80`, as on LR3T (seen).

# 0x14C - Wrench Lamp Request (GT500)

**Sample:** `18 00 00 03 00 03 40 00` (built from the firmware field table: lamp off, valid) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 2 | 0 (`0x01`) | Lamp request | read, as on LR3T (exact) |
| 0 | 7 - 4 | Nibble | **no longer read** by the GT500 MCU; the 2020 MCU unpacks it (exact) |

# 0x17E - Intercooler Coolant / Manifold Charge Temp

**Sample:** `FE 78` (built from the firmware field table: IC coolant invalid, MCT 40 C) · **Rate:** missing after 5.0 s · **Status:** From firmware

Received but not read by the 2020 MCU (exact). The GT500 MCU feeds two gauges from it. DLC 2.

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 0 | 7 - 1 | Intercooler coolant temp | 7 bit, `0x7E` / `0x7F` invalid. The code adds `+ 50` (as read); gauge with option 381 (exact) |
| 1 | 7 - 1 | Manifold charge temp (MCT) | 7 bit, **`(byte 1 >> 1) - 20 = °C`**, the same as the 2018 bench's `raw x 0.5 - 20`; `0x7E` / `0x7F` invalid. Gauge with option 376 (exact) |

- A stock GT500 EEPROM has options 376 and 381 **clear**, so stock it shows neither gauge (exact).
- The GT500 display program has two gauge pages, with IC coolant, MCT, engine oil temp, vacuum / boost and oil
  pressure 0 - 150 psi (exact, seen).

```
IC coolant invalid, MCT 40 C                  FE 78
```

# 0x18B - Shift Light Frame (GT500, cluster TX)

**Sample:** `00 00 64 0A 00 00 00 00` (seen: GT500 MCU on a normal car's EEPROM), `00 00 00 1E 00 00 00 00` (seen: GT500 MCU on its own EEPROM, the same as the 2020 MCU) · **Rate:** 1000 ms · **Status:** Cluster sends (from firmware). Don't send it

Constant on the 2020 MCU; **filled** by the GT500 MCU (exact):

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 2 | 6 - 0 | Shift light value | written with option 211 (exact) |
| 3 | 5 - 0 | Shift rpm | `(rpm - 2000) / 100`, so `1E` = 5000 rpm, `0A` = 3000 rpm (exact, seen) |
| 4 | 6 - 4 | Mode | a mode field (exact; meaning open) |

- So a GT500 cluster on a normal car puts different bytes on the bus in this frame (seen). Which module reads
  `0x18B` is open.
- The GT500 shift light menu adds Change with Drive Mode, Shift Light Intensity and Display Source (HUD / Cluster)
  (exact).

# 0x191 - Units / Language Request (GT500)

**Sample:** `3C 00 00` (built from the firmware field table: no units request) · **Rate:** missing after 5.0 s · **Status:** From firmware

- In the decode slot where the 2020 MCU reads the 7 bit language field (byte 2 bits 7 - 1), the GT500 MCU reads a
  5 bit field in byte 0 bits 7 - 3 (exact; what it means is open). The units fields are in
  [2020_lr3t_database.md](2020_lr3t_database.md).

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 0 | 7 - 3 | 5 bit field | read in place of the 2020 language field (exact; meaning open) |

# 0x279 - Unknown Status (GT500)

**Sample:** `00 00 00 00 00 00 40 82` (built from the firmware field table: update bits set, values 0) · **Rate:** missing after 5.0 s · **Status:** From firmware

The 2020 MCU unpacks it but nothing reads it (exact). The GT500 MCU also reads byte 6:

| Byte | Bits | Signal | Values |
|---|---|---|---|
| 6 | 3 - 2 | 2 bit value | (exact; meaning open) |
| 6 | 1 (`0x02`) | Flag | (exact; meaning open) |

# 0x2FD - Settings, Exhaust Request (GT500, cluster TX)

**Sample:** `00 00 00 00 00 00 60 00` (built from the firmware field table: exhaust request Sport, other fields 0) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 6 | 7 - 5 | Exhaust mode request | `1` Quiet (shown as **Stealth**), `2` Normal, `3` Sport, `4` Track. Also set from the console exhaust button on [0x361](#0x361---console-buttons-gt500) (exact) |

- The other fields go out as on LR3T (exact where the producer was matched).
- The GT500 display program has an ExhaustSelect pop-up (Stealth / Normal / Sport / Track) (exact).

# 0x30A - Steering Requests (GT500, cluster TX)

**Sample:** `C7 01 00 00 00 00 00 00` (built from the firmware field table: steering feel Sport, blind spot / cross traffic not configured) · **Rate:** 200 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 1 | 2 - 0 | Steering feel request | `0` Normal, `1` Sport, `2` Comfort only. No drive mode steering codes `3` / `4`, which the 2020 MCU can send (exact) |

# 0x361 - Console Buttons (GT500)

**Sample:** `00 00 00 00 00 00 10` (built from the firmware field table: exhaust button held), `00 00 00 00 00 00 80` (drive mode button held) · **Rate:** missing after 5.0 s · **Status:** From firmware

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 6 | 4 (`0x10`) | **Exhaust button** | `1` pressed: the exhaust mode pop-up, needs EEPROM option 232 (exact). The 2020 MCU doesn't read it |
| 6 | 6 (`0x40`) | Steering feel button (2020 path) | **not read** by the GT500 MCU; it uses `0x081` byte 0 `0x20` instead (exact) |
| 6 | 7 (`0x80`) | Drive mode button | the same as LR3T (exact) |

# 0x3C8 - Drive Mode, MagneRide Request (GT500, cluster TX)

**Sample:** `00 00 03 00 00 00 00 00` (built from the firmware field table: drive mode Track) · **Rate:** 1000 ms, also on change · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 2 | 2 - 0 | Drive mode | `0` Normal, `1` Sport, `3` Track, `4` Drag (exact) |
| 7 | 5 - 3 | Damper (MagneRide) request | also set from the MagneRide key on `0x081` (exact) |

- The GT500 has **no ambient lighting menu**; MyColor (gauge colour) still goes out on `0x3C8` (exact). Rev Match
  is absent from its menus, and Launch Control moved under Track Apps (exact).

# 0x430 - Drive Mode Request (GT500, cluster TX)

**Sample:** `05 00 00 00 00 00 00 14` (built from the firmware field table: automatic, drive mode request Track) · **Rate:** 100 ms · **Status:** Cluster sends (from firmware). Don't send it

| Byte | Bits | Signal | GT500 (LR3V) |
|---|---|---|---|
| 7 | 5 - 2 | Drive mode request | Normal `0`, Sport `1`, **Track `5`**, Drag `7` (exact). The 2020 MCU sends Track as `1` and Drag `7` (automatic) / `1` (manual) |
| 0 | 1 (`0x02`) | Flag | also set by the GT500's option 88 feature, Neutral Tow (exact) |
| 6 | 4 (`0x10`) | Gear display bit | written without the 2020 condition (exact; effect open) |

- So a GT500 cluster puts a different drive mode code on the bus for Track (exact). How the modules of a non-GT500
  car answer `5` is open.

# Topics

## Drive modes on the GT500

| Mode | 2020 `0x430` byte 7 bits 5 - 2 | GT500 `0x430` byte 7 bits 5 - 2 | GT500 `0x3C8` byte 2 |
|---|---|---|---|
| Normal | `0` | `0` | `0` |
| Sport+ ("Sport" on the GT500) | `1` | `1` | `1` |
| Track | `1` | **`5`** | `3` |
| Drag Strip | `7` automatic / `1` manual | `7` (no manual test) | `4` |

- GT500 drive modes: Normal, Sport, Track, Drag, Slippery, MyMode (exact).
- One routine sends all the drive mode requests (`0x430`, `0x3C8`, steering on `0x30A`, MagneRide and exhaust); the
  2020 MCU's per-module confirm routines have no match in the GT500 code (exact). How the GT500 confirms a mode is
  open. In the emulator the modules echo the requests (seen).
- The drive mode button is `0x361` byte 6 bit 7 on both (exact).

## Other differences outside CAN

For context only (exact unless noted):

- Speedometer: 220 mph / 360 km/h only; the 180 mph feature-car bit does nothing (exact, seen).
- Tach: redline fixed at 7,500 rpm in the art (exact).
- Gear strip: PRNDM always, even with a manual EEPROM (seen).
- Start-up: sweep only; adds a Neutral Tow welcome (exact, seen).
- Warnings: adds shift by wire (DCT), clutch over-temp, oil level, damper / exhaust / suspension, neutral tow and
  EPB messages (exact).
