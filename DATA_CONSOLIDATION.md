# WYNG Data Consolidation

Every piece of data the WYNG app handles, in two parts:

1. **[Raw data](#section-1--raw-data-from-the-puck-set)**: what the N puck devices send to the phone, taken from ICD rev 0.4, which documents the firmware as built.
2. **[App data](#section-2--app-ui-data-not-raw)**: everything the wireframes show that the pucks do not send directly. That covers derived metrics, aggregates, user-entered data, account and settings data, generated text, and app state. The section also records where the same data appears on more than one screen.

**Sources**

| Source | Location | Used for |
|---|---|---|
| Firmware Requirements Specification, WYNG-FRS-001 Draft 0.2 (3 Sep 2026) | [WYNG_Firmware_Requirements_Specification_v0.2.docx](untracked/WYNG_Firmware_Requirements_Specification_v0.2.docx) | Section 1 |
| **Interface Control Document, WYNG-ICD-001 rev 0.4 (5 Nov 2026), draft.** Documents firmware wyng-sleeve v0.7.0 as built, and governs on interface detail. | [WYNG-ICD-001_rev0.4.md](untracked/WYNG-ICD-001_rev0.4.md) | Section 1 |
| Firmware ↔ App Integration Plan (ICD scope), TigerTeam | [WYNG_FW_App_Integration_Plan.md.docx](untracked/WYNG_FW_App_Integration_Plan.md.docx) | Section 1 |
| Core Metrics & Pitch Classification, Developer Spec v3 (30 Oct 2026) | [Wyng_Core_Metrics_and_Pitch_Classification_Developer_Spec.pdf](untracked/Wyng_Core_Metrics_and_Pitch_Classification_Developer_Spec.pdf) | Section 2: definitions of Performance, CBR, Consistency, pitch types and classification |
| Core Metrics & Mathematical Data Engine Specification v3.0 (Nov 2026) | [wyng_data_engine_specification.pdf](untracked/wyng_data_engine_specification.pdf) | Section 2: formulas and stored fields |
| Screen Board Draft v0.5, all 41 screens plus Parts 1–6 (incl. Part 4 *Data contract*) | `System Design/WYNG Screen Board.html` | Section 2 |
| Prototype deck, rendered screens | [Prototype/](Prototype/) | Section 2 (chart axes, series) |
| Current mock models | `Wing/App/**/…Model.swift`, `Wing/Connectivity/DataParsing/` | Cross-checks |

**Conventions**

- `FRS-xxx-nnn` refers to a firmware requirement, `FRS §nn` to an FRS section, and `ICD §nn` to an ICD section.
- Screen IDs such as `01a` and `02d` are the Screen Board IDs. They match the file names in [Prototype/](Prototype/).
- **⚠** marks a gap, a conflict, or a decision still open. All of them are collected in [Open issues](#open-issues) at the end.
- Units: the board's data contract says *"Store SI, render imperial"*. Puck units are as ICD rev 0.4 defines them.

---

# Section 1 — Raw data from the puck set

**This section follows ICD rev 0.4, which documents the firmware as built (wyng-sleeve v0.7.0).** Rev 0.4 replaces most of rev 0.3 and is still a draft awaiting Wyng's sign-off (ICD §16). Features of the FRS and rev 0.3 that aren't in this firmware are listed in [1.13](#113-not-in-firmware-v070). Byte layouts of the diagnostic fields are in the ICD and aren't repeated here.

## 1.1 Topology and what "raw" means here

| Item | Value | Source |
|---|---|---|
| Devices per set | **3 identical pucks**, one firmware image. The firmware supports 1–8. | ICD §2 |
| Positions | Biceps, triceps and forearm flexor. **Pucks carry no position labels.** Pairing gives each one an index, 0 to n − 1, and position is worked out from recorded data ([1.7](#17-sensor-configuration-position-and-calibration)). | ICD INV-ROLE-1, §12.4 |
| Role | Each puck is a BLE **peripheral**, with one central at a time. The app connects to, commands and downloads from each puck independently and in any order. | ICD INV-CONN-1 |
| Transport | **GATT only**, for everything including bulk download. There's no L2CAP. | ICD §5, §10.3 |
| Puck↔puck link | A proprietary 2.4 GHz sync link. The app only sees its results (`PUCK.sync`, the SYNC records). | ICD §2, §3 |
| Byte order | Little-endian, packed | ICD header |
| Security | **None in this firmware.** No pairing, bonding or encryption. Every characteristic is open. | ICD §4, §16 |
| Timebase | Each puck's 64-bit **GRTC** (1 µs per tick, since boot). Pucks are aligned through **sleeve time** (1.6). **The puck has no wall clock.** | ICD §3 |
| Recording vs Bluetooth | A Bluetooth drop never stops a recording. | ICD INV-CAP-1 |

"Raw data" means anything the app **reads from, or is notified by, a puck**. **The pucks detect nothing and compute no metrics.** There is no on-puck throw detection and no on-puck orientation. Each puck records its raw sensor streams continuously between REC START and REC STOP, and the app does the rest.

The data reaches the app over these paths:

```
 ┌── Advertising (no connection) ───────────── 1.2 name, index, sleeve ID
 │
 ├── GATT reads / notifies (per puck)
 │     Puck: PUCK (1 Hz) ───────────────────── 1.3, 1.5 identity, battery, status
 │     Puck: INFO, EMG/IMU/Recorder INFO ───── 1.4 versions, scales, rates
 │     Puck: DATA (ToF MEAS) ───────────────── 1.10 live ranges
 │     EMG: STREAM ─────────────────────────── 1.10 live EMG
 │     IMU: STREAM ─────────────────────────── 1.10 one-second snapshots
 │     Recorder: CTL ───────────────────────── 1.8 start, stop, mark, erase, offload
 │     SMP ─────────────────────────────────── 1.12 firmware update
 │
 └── Recorder: DATA (GATT notify) ──────────── 1.9 recorded sessions (the payload)
```

## 1.2 Advertising (no connection)

The puck advertises whenever it is up and not connected (ICD §4).

| Field | Type / unit | Notes |
|---|---|---|
| Name | `WYNG-<d>-<tail>` | `d` = index + 1, or `N` when unpaired. `tail` = last two serial bytes in hex, e.g. `WYNG-2-8144`. |
| `idx` | u8 | Index in the sleeve; 0xFF when unpaired |
| `sleeve_id` | u32 | 0 when unpaired |
| `sleeve_n` | u8 | Pucks in the sleeve |
| RSSI | dBm (measured by the phone) | Gives a proximity estimate. The pairing list needs it (06b: *"listed with signal strength"*). |

The advertisement carries no battery, charge or unsynced-data information. The app has to connect to read those.

## 1.3 Identity and battery (PUCK, `sl_puck`)

There are no standard Device Information or Battery services. These values come from the PUCK characteristic (read, or notify at 1 Hz).

| Field | Type | Notes |
|---|---|---|
| `serial` | u8[8] | Device ID |
| `fw_major`, `fw_minor`, `fw_patch` | u8 | Shown as `Firmware 2.4.1` on 05e, 06a and 06c |
| `soc_q8` | u16 | Battery %, × 256. Shown on 06a and 06c. Valid when `gauge_ok` is set. |
| `vcell_mv` | u16 | Cell voltage, mV |
| `charge` | u8 | 0 unknown, 1 battery, 2 low, 3 charging, 4 full |

## 1.4 Capabilities (INFO characteristics)

There's no capability descriptor or handshake. Each service has an INFO characteristic to read (ICD §15).

| Characteristic | Carries |
|---|---|
| Puck INFO (`pc_info`) | ToF configuration: rate, sample period, pretrigger, thresholds, max range |
| EMG INFO (`emg_info`) | `bits` 12, `fs_hz` 2000, `lsb_nv` 878906 |
| IMU INFO (`imu_info`) | `odr_hz` 1920, scale factors (see 1.7), snapshot size |
| Recorder INFO (`rec_info`) | State, session, pages used and free, bad blocks, session length, last error |

## 1.5 Status (PUCK, 1 Hz)

| Field | Type / unit | Notes |
|---|---|---|
| `t_ms` | u32 | Since boot |
| `sync` | u8 | 0 off, 1 search, 2 track, 3 locked, 4 master |
| `res_min`, `res_max` | i8, µs | Sync residual over the last second |
| `sync_off_us` | i64, µs | Own GRTC − sleeve time |
| `rec_state` | u8 | 0 no NAND, 1 idle, 2 recording, 3 erasing, 4 offloading |
| `rec_flags` | u8 | 0x01 full, 0x02 last session ended on an error |
| `rec_session`, `rec_s` | u16 | Current or last session, and its length in s |
| `rec_free_min` | u16 | Recording time left, in minutes |
| `rec_drop` | u16 | Records lost to a full buffer |
| `imu_a_mg` | u16, mg | Peak \|a\| low-g over the last second |
| `imu_hg_dg` | u16, 0.1 g | **Peak \|a\| high-g over the last second** |
| `imu_w_dps` | u16, dps | **Peak \|ω\| over the last second** |
| `imu_clip_a`, `imu_clip_g` | u16 | **Samples at full scale** in the last second, low-g and gyro |
| `emg_rms` | u16, 0.1 mV | EMG level, 20 Hz high-pass |
| `flags` | u8 | Includes 0x01 lead-off (electrode contact lost) and 0x02 IMU ok |
| `idle_s`, `off_in_s` | u16, s | Since last motion; until power-off |
| `die_temp_q2` | i16, 0.25 °C | nRF chip temperature. **The only temperature reading.** |
| `master`, `tx`, `heard` | u8 | Sync master, the puck transmitting ultrasound, and which pucks this one heard |
| `sflags` | u8 | Includes 0x10: image on trial after a firmware update |

The rest of the structure is diagnostics (ICD §6.1). New fields are only appended, and `ver` says which are present.

## 1.6 Time

**The puck keeps no wall clock, and the app stamps wall time itself** (ICD INV-TIME-2). The firmware doesn't run an RTT exchange, and there's no wall-clock offset.

| Quantity | Definition |
|---|---|
| Own GRTC | Each puck's counter since boot, in µs. Every puck timestamp uses it. |
| Sleeve time | The master's GRTC, continuous across a change of master. **The only key for aligning pucks.** |
| Mapping | `t_sleeve = t − (off_us + drift_ppb · 1e-9 · (t − t_ref))`, using the latest SYNC record before `t` (recordings), or `PUCK.sync_off_us` (live) |
| Quality | `PUCK.sync`, `res_min/res_max`, `SYNC.valid`. Measured residual −1 to +1 µs, with at least 98% of frames within ±2 µs. |

## 1.7 Sensor configuration, position and calibration

**Sensors and rates are fixed in this firmware** (ICD §2, §8.1). There's no sample-rate service, and rates can't be configured.

| Stream | Rate | Range | Scale |
|---|---|---|---|
| Low-g accelerometer `a` | 1920 Hz | ±16 g | 488 µg/LSB |
| **High-g accelerometer `h`** | 1920 Hz | **±320 g** | 10,417 µg/LSB |
| Gyroscope `g` | 1920 Hz | **±4000 dps** | 140 mdps/LSB |
| EMG, **one channel** | 2000 Hz | 12-bit, 3.6 V full scale at the ADC | `lsb_nv` 878906 (output-referred). Input-referred ÷ AD8233 gain, 331.5 assumed. |
| Ultrasonic ToF | 200 Hz | Direct one-way range | 0.02 mm/LSB |

**Position (ICD §12.4).** The firmware stores none. The app works it out from a recording:
- The two pucks that range with each other (ToF records, "RX edges by transmitter") are the upper-arm and forearm pair. The third is the other upper-arm puck.
- The two upper-arm pucks move together in the IMU data, and the forearm puck moves differently.
- So: ranging + upper-arm motion = **biceps**; ranging + different motion = **flexor**; not ranging = **triceps**.

The ICD doesn't cover **arm side** (left or right). The app calls the result **puck placement** (A14).

**Calibration.** There's no calibration service. The pucks store no lever arms, alignment, bias or gain (ICD §15). All calibration belongs to the app.

## 1.8 Recorder: what is on the puck

A session is one recording, started and stopped by the app. Each puck has about 47 minutes of room, or about 65,000 pages (ICD §10.5).

**Commands (Recorder CTL, per puck):**

| Op | Name | Arguments | Notes |
|---|---|---|---|
| 0x01 | START | tag, up to 16 bytes of text | Must be sent to **every** puck. Refused while recording, erasing or offloading. |
| 0x02 | STOP | none | Ends the session |
| 0x03 | MARK | `id` u16, text up to 32 bytes | Writes a timestamped mark into the running session |
| 0x04 | ERASE | key, `all` u8 | Erases the recordings. Refused while recording or offloading. |
| 0x05 | OFFLOAD | `from`, `to` (pages) | Sends those pages on the DATA characteristic |
| 0x06 | CANCEL | none | Stops an offload |

- **A session ends** on STOP (reason 0), NAND full (1), low battery (2: cell below 3.4 V for 5 s) or a NAND error (3). It never ends on a Bluetooth drop.
- **The recorder does one thing at a time.** It returns "busy" while recording, erasing or offloading, so as we read the ICD, data can't be downloaded during a recording.
- **No sync pointer or commit.** The puck doesn't track what has been downloaded. The app tracks pages itself (`wp`, `s_pages` in `rec_info`) and decides when to ERASE. When the NAND is full, recording stops: nothing is overwritten.

**Offload packets:** pages are sent as chunks of MTU − 13 bytes (234 at MTU 247), and END follows with `status`, page counts, and ECC and CRC failures. The app can re-request any page range, which is how it resumes.

**Throughput:** measured at about 25–28 kB/s to a Windows PC. Not yet measured to a phone (ICD §16).

## 1.9 Recorded session (the core payload)

**This is the core payload.** It is **one continuous recording per puck, not one packet per throw.** Each 2048-byte NAND page has a 24-byte header with a CRC, and a stream of typed records runs across pages (ICD §10.4). Every record has a 4-byte header (`type`, `ver`, `len`).

### 1.9.1 Record types

| Type | Name | Payload | Rate |
|---|---|---|---|
| 0x01 | SESSION | 72 B (below) | Session start |
| 0x02 | IMU | Header + up to 32 samples (1.9.2) | 60/s |
| 0x03 | IMU_ANCH | Sample index + hardware timestamp | ~7.5/s |
| 0x04 | EMG | 20 B header + 100 samples (1.9.3) | 20/s |
| 0x05 | TOF | 28 B (1.9.4) | 200/s |
| 0x06 | SYNC | `t_us`, `off_us`, `drift_ppb`, `valid`, `master` (1.6) | 1/s |
| 0x07 | PUCK | The full PUCK structure (1.3, 1.5) | 1/s |
| 0x08 | MARK | `t_us`, `id`, text | On MARK |
| 0x09 | END | `t_us`, pages, records lost, `reason` | Session end |
| 0x0A | DROP | Records and bytes lost, types affected | After a loss |

**SESSION** carries `session`, `fw`, `rec_ver`, `serial`, `sleeve`, `t_us` (start, own GRTC), `page0`, `tag` (from the app), `imu` (the IMU INFO, with scale factors), `idx` and `n`. It carries no calibration data.

### 1.9.2 IMU samples

Each sample is 18 bytes: `a` i16[3], `g` i16[3], `h` i16[3], raw x, y, z, scaled per 1.7. The block header has `first` (sample index), `n`, `flags` (0x01: samples were lost before this block) and `partial` (a bit per sample lacking gyro or high-g).

**Timing:** every 32nd sample raises a hardware-timestamped anchor. A line through neighbouring anchors times every sample, split at resync blocks (ICD §8.3, §11). Then map to sleeve time (1.6).

### 1.9.3 EMG samples

Each block holds 100 i16 ADC codes (50 ms, one channel). The header has `seq`, `n`, `flags`, `idx`, `blk_last` and `t_us`. Flags: 0x01 lead-off, 0x02 blocks dropped.

- **Conversion:** output-referred µV = code × `lsb_nv` / 1000.
- **Timing:** `t_us` is the time of sample `blk_last`. Sample *i* is at `t_us − (blk_last − i) × 1e6 / fs_hz`.

There's no on-puck envelope in the recording, so the app computes it from the codes.

### 1.9.4 Ultrasonic ToF

One record per trigger, **200 per second**. At each trigger exactly one puck transmits and the others listen (ICD §9, §12.3).

| Field | Type | Meaning |
|---|---|---|
| `t_us` | u64 | This puck's trigger, own GRTC. For RX records the TX instant is `t_us + pretrig_us`. |
| `kind` | u8 | 0 this puck transmitted, 1 this puck listened |
| `plan` | u8 | Bits 3..0 transmitting index, 6..4 master index, 7 discovery turn |
| `m.seq` | u32 | Sync frame that scheduled the trigger |
| `m.range` | i16 | Direct range, 0.02 mm; −32768 = none |
| `m.le`, `m.le1` | i16 | Leading-edge ranges from the puck's own detector, 0.02 mm |
| `m.amp`, `m.le_amp` | u16 | Amplitudes |
| `m.flags` | u8 | Timeout, no target, predicted, leading edge found, etc. |

The transmitter is re-chosen at each discovery (3 s after the master starts, at every REC START, and every 60 s).

### 1.9.5 Volume

At about 47 kB/s per puck (IMU 35.5, ToF 6.4, EMG 4.5):

| Recording | Per puck | 3 pucks | Offload at 26 kB/s, per puck | 3 pucks, one at a time |
|---|---|---|---|---|
| 28-minute bullpen | 79 MB | 237 MB | ≈ 51 min | **≈ 2.5 h** |
| Full NAND (~47 min) | 132 MB | 398 MB | ≈ 85 min | **≈ 4.25 h** |
| 42 throws × 2 s only, for comparison | 4 MB | 12 MB | ≈ 2.5 min | ≈ 7.6 min |

The offload rate is the PC measurement. A phone may differ.

### 1.9.6 How a throw reaches the app

1. The app sends **REC START** to every puck, with a tag, and stamps the wall time itself.
2. The pucks record continuously, with no notion of a throw. A Bluetooth drop doesn't matter.
3. The app sends **REC STOP** to every puck.
4. The app **offloads each puck**, decodes the records and maps them to sleeve time.
5. The **app finds the throws** in the IMU data and works out each puck's position.

So measurement data arrives only after a session ends. During the session, the app has only the live data in 1.10.

## 1.10 Live data

These are limited views, not the recorded data.

| Source | What it gives | Limits |
|---|---|---|
| **PUCK notify, 1 Hz** | Per second: peak high-g, low-g and angular velocity, clip counts, EMG RMS, lead-off, battery, recorder state (1.3, 1.5) | One-second peaks, not timed samples |
| **EMG stream** | The 2 kHz samples, 20 packets/s, 4.4 kB/s | On request (CTL 0x01). Blocks can drop if the send queue is full. |
| **IMU snapshot** | The next 1920 samples (1 s) of `a`, `g`, `h` with anchors | On request, so it can't be aligned to a throw that hasn't happened |
| **ToF MEAS** (Puck DATA) | Each listening puck's ranges | 16 B per measurement |
| **ENV** | Amplitude envelope of one ToF measurement | On request, for diagnostics |

There is **no realtime orientation or elbow-angle stream**, no throw event, and no identify-by-tap event. A planned realtime service is listed for firmware v0.10 (ICD §15, §16).

**IDENTIFY** (Puck CTRL 0x12) makes a puck's LED flash white for 5 s.

## 1.11 Errors

Failed writes return **standard ATT error codes**. There is no WYNG status enum (ICD §14).

| Error | Code | When |
|---|---|---|
| Request Not Supported | 0x06 | Unknown CTRL command |
| Invalid Offset | 0x07 | Offset ≠ 0 on a CTL write |
| Invalid Attribute Value Length | 0x0D | Empty CTRL write |
| Unlikely Error | 0x0E | Puck rebooting, no NAND, or IMU snapshot outside data mode |
| Insufficient Resources | 0x11 | NAND full, or ring full for MARK |
| Value Not Allowed | 0x13 | Bad arguments, wrong erase key, MARK with no session, OFFLOAD without notifications on |
| Procedure Already in Progress | 0xFE | Recorder busy, or IMU snapshot running |

## 1.12 Firmware update status

Updates use SMP (MCUmgr) over BLE (ICD §13). The flow: upload → mark for test → reset → the puck boots on trial (`PUCK.sflags` 0x10) → the app reconnects and checks the version → the app writes confirm.

**The app must confirm each puck.** There's no automatic confirm, and an unconfirmed image reverts at the next reset. A set of three took about 1.5 minutes on the bench.

## 1.13 Not in firmware v0.7.0

| Item (FRS or rev 0.3) | Status | Planned |
|---|---|---|
| Throw windows, `throw_id`, on-puck throw detection, prebuffer, CaptureComplete | Replaced by continuous sessions | — |
| Encrypted, bonded links | Not implemented | Open item (ICD §16) |
| Realtime 22-byte packet, on-puck quaternions, on-puck elbow angle | Not implemented | v0.10 realtime; quaternions "planned" |
| Wall clock and RTT exchange | Dropped | — |
| Location/role enum, auto-location result | Dropped | — |
| Sample-rate and calibration services, capability descriptor | Not implemented | v0.10 config variables |
| Ring buffer, sync pointer, unsynced-overwrite counter | Replaced by page offload and ERASE | — |
| L2CAP bulk transfer | Not implemented | If phone throughput requires it |
| Magnetometer | Not implemented | v0.9 |
| Temperature sensor (beyond the chip temperature) | Not in the ICD | — |
| Tap on/off, don/doff, factory-reset gesture | Not implemented | v0.8 |
| WYNG error enum, standard DIS/BAS services | Not implemented | — |

---

# Section 2 — App UI data (not raw)

This section covers everything the wireframes show that the pucks do not send directly. **Each data element is defined exactly once**, in [2.2](#22-data-catalogue), with an ID (`D-nn`). Screens refer to those IDs. Where a screen shows a puck value unchanged, it points to that value's definition in Section 1 instead of redefining it.

Where throw metrics and aggregates are calculated (on the phone or on the server) is TBD.

## 2.1 Screen inventory: data per screen

The **IDs** column says where each item is defined. `1.n` points to a Section 1 field, shown per puck.

### 01 Home (01a–01n)

The period control scopes every number on the screen.

| Screen | Data shown | IDs |
|---|---|---|
| 01a Home, week | Period (`Week`, `AUG 17–23`) · Performance **84** · CBR **0.82** · Arm speed **78 mph** · Pitch count **525** · Insight title and message · Metric scroller (Performance, Arm Speed, CBR, Pitch Count) · Chart for the selected metric (Performance per day M–S: 80, 82, 81, 84, 83, 86, 88) · Contributors: Arm Speed 78 mph, CBR 0.82, Acceleration 39.6 g, Consistency 86%, Pitch Volume 525, each with a bar fill and a direction (up, down or flat) · Avatar | D-20, D-01, D-06, D-30, D-31, D-25, D-29, D-34, D-37, D-90 |
| 01b Arm speed sheet | Value 78 mph · Explanation · **Your range, 30 days** 74–82 mph · **Weight in score** 28% | D-30, D-33, D-28 |
| 01c Arm speed selected | Line chart M–S (76, 77, 76, 78, 77, 79, 78), dashed **AVG 77** | D-30 |
| 01d CBR selected | Line chart of CBR against the **0.80** target | D-06, D-36 |
| 01e Pitch count selected | Column chart per day. Rest days are drawn as gaps, not zeros. | D-31 |
| 01f Period menu | Day / Week / Month | D-20 |
| 01g Day | `THU, AUG 21`: Performance 88, CBR 0.79, 79 mph, 42 throws. Chart x-axis is **the day's sessions by type** (WARM [Warm-Up] 82, BULLPEN 88, LONG TOSS 85). Contributors: Acceleration 41.2 g, Consistency 89%. | D-20, D-30, D-31, D-25, D-41 |
| 01h Month | `AUGUST`: 85, 0.83, 78 mph, 2,140 throws. Chart x-axis W1–W5. | D-20, D-30, D-31, D-25, D-34 |
| 01i Previous week | `AUG 10–16`: 80, 0.79, 77 mph, 468 | D-20, D-30, D-31, D-25, D-34 |
| 01j Nothing thrown | Blurred layout · "Nothing thrown this week" | D-20, D-70 |
| 01k CBR sheet | 0.82 · three states: **Burned** under 0.80, **Ready** 0.80–0.85, **Charged** over 0.85 · **Back to charged after a heavy day: about 38 hrs** | D-06, D-36, D-53 |
| 01l Acceleration sheet | 39.6 g · 30-day range 36–42 g · weight 22% · *"a change of more than 2 g reads as a mechanical shift"* | D-30, D-33, D-28 |
| 01m Consistency sheet | 86% · 30-day range 80–92% · weight 18% | D-25, D-33, D-28 |
| 01n Pitch volume sheet | 525 throws · **by session type**: Bullpen 179, Game 115, Long toss 105, Catch play 126 · **Against your 28-day average +12%** | D-31, D-32, D-34 |

### 02 Live (02a–02f)

| Screen | Data shown | IDs |
|---|---|---|
| 02a Idle | Connection strip `LIVE · WYNG CONNECTED` · Metric scroller (Arm Speed, CBR, Arm Load, Arm Slot) · Line chart over the last 7 throws (x = pitch #, AVG line) · **Recent throws** table: pitch #, type code (4S, SL, CH, 2S), arm speed mph, arm load Nm, arm slot ° | D-11, D-13, D-14, D-16, D-17, D-18, D-30, D-61 |
| 02b Session active | Session name **Bullpen** · elapsed timer **28:04** · tiles: session arm speed **78 mph**, throws **42** · same chart and table | D-40, D-45, D-30, D-31 |
| 02c Saved | `WYNG CONNECTED · SYNCED` · "Session saved" · **Bullpen · 42 throws · 28 min** | D-40, D-31, D-45, D-61, D-62 |
| 02d Throw detail sheet | **Throw 42 · Four-seam** · 4:46 PM · 79 mph · arm slot 42.1° · acceleration 40.3 g · **type confidence 94%** · **session: Unassigned** | D-11, D-12, D-13, D-14, D-15, D-18, D-19, D-40 |
| 02e Arm slot selected | Arm slot line with no polarity, AVG 42 | D-14, D-30 |
| 02f Not connected | `OFFLINE · LOOKING FOR YOUR WYNG` · empty state | D-61 |

> **Differs from firmware v0.7.0 (A25, A26).** The wireframes assume the sleeve never stops recording and that throws appear live. As built, the pucks record only between REC START and REC STOP, and throw data arrives only after an offload (1.9.6).
>
> **Differs from the wireframes (A5).** The athlete chooses the session type when they tap **Start a session** (02a), and the session name is generated from it (D-40). The wireframes instead show the name being set at the end of the session (02c "Edit session", 03d "Rename").

### 03 Sessions (03a–03d)

| Screen | Data shown | IDs |
|---|---|---|
| 03a Sessions list | Period · per tile: **name**, start time (relative day), **sparkline** with a direction colour, **score** (e.g. 88), **throw count**, **average arm speed** | D-20, D-40, D-42, D-49, D-30, D-31 |
| 03b Session detail | Name · `Thu, Aug 27 · 4:19 PM · 28 min` · tiles for arm speed 78 mph and throws 42 · chart (Arm Speed, CBR, Arm Load, Arm Slot) · **All 42 throws** table | D-40, D-42, D-45, D-30, D-31, D-11, D-13, D-14, D-16, D-17, D-18 |
| 03c Empty | "No sessions yet" | D-70 |
| 03d Rename | Session name (editable) · throws 42 · started 4:19 PM · duration 28 min (all read-only) | D-40, D-31, D-42, D-45 |

### 04 WYNG Man (04a–04c)

| Screen | Data shown | IDs |
|---|---|---|
| 04a Conversation | Date label · greeting · suggestion chips · user messages · **structured answers** (claim, supporting numbers, link to source screen) · follow-up chips | D-80…D-83 |
| 04b Opening | Greeting · three suggestions | D-81 |
| 04c From insight | Question seeded from the Home insight · answer: *"You gained four points… arm speed added three and CBR added two; consistency gave one back"* · follow-up chips | D-29, D-82, D-34 |

### 05 Settings (05a–05e)

| Screen | Data shown | IDs |
|---|---|---|
| 05a Root | Name **Jordan Smith** · email · **position** (Pitcher) · **handedness** (Right-handed) | D-90, D-91, D-92 |
| 05b My profile | Picture · name · email · DOB 12 Mar 2009 · height 6′1″ · weight 178 lb · gender · **throwing arm** Right · **measurements** (arm, forearm) · **home location** Northgate, CA | D-90, D-92…D-97 |
| 05c Settings | Units (Imperial) · Appearance (Dark) · Language · Notifications · Health app (Off) | D-100…D-104 |
| 05d Security | Reset password · two factor (Off) · contact preferences · download my data · delete account | D-105, D-106 |
| 05e Help | App **version 1.0 (214)** · **Firmware 2.4.1** | D-107, 1.3 |

### 06 Pairing (06a–06c)

The wireframes show one device card. Per the decision to show each puck separately, every puck gets its own values.

| Screen | Data shown | IDs |
|---|---|---|
| 06a My devices | **WYNG 4821** · **Right sleeve** · firmware 2.4.1 · **Connected** · battery **84%** · **last sync 4:46 PM** · name/rename · *"One WYNG per arm. Sessions stay attached to whichever WYNG recorded them."* | D-110, D-111, D-112, D-115, 1.3, D-47 |
| 06b Pair step 1 | Illustration only. The notes add: *"Devices in range are listed with signal strength and the weak one is shown dimmed with the reason."* | 1.2 |
| 06c Paired | **WYNG 4821 is paired** · battery 84% · **Firmware 2.4.1, up to date** · **Arm: Right** · rename | D-110, D-112, D-118, 1.3 |

### 07 Reports (07a–07d)

| Screen | Data shown | IDs |
|---|---|---|
| 07a Reports | **Workload**: ratio 1.18, trend "rising", status word **Watch**, about 40 throws to the **1.30 ceiling**, chart of 7-day load against the 28-day baseline · **Durability** (last 10 sessions): arm speed holds to throw 28 then falls about 2.1% every 10 throws, chart of arm speed by 10-throw bucket (1–10 … 51–60) with AVG · **Volume by session type** (30 days): Bullpen 34%, Game 22%, Long toss 20%, Catch play 24% · **Recovery** (after a heavy day): 38 hrs, CBR line against hours with the 0.80 floor · **Personal bests**: peak arm speed 82 mph (Aug 19), best session 91 (Aug 25 bullpen), most throws per week 612 (Aug 4–10) · **Progress** (90 days): arm speed +5.3%, consistency +6 pts, CBR steady, workload +0.23 | D-50, D-51, D-32, D-53, D-35, D-34, D-29 |
| 07b Card swiped | Pin / delete | D-58 |
| 07c Workload detail | **Acute load, 7 days: 412 throws** · **Chronic load, 28 days: 349 throws** (a weekly average) · ratio 1.18 · ceiling 1.30 · "What this means" text | D-50, D-36, D-29 |
| 07d Add a report | Catalogue of 6 reports with descriptions · selection count ("Add 2 reports") | D-58 |

### 08 Arm check, and board Part 3 States

| Screen | Data shown | IDs |
|---|---|---|
| 08 Arm check | Date · **feel**: Clear, Tight, Sore or Pain (required) · muscle: Shoulder, Upper arm or Forearm (optional, multi-select) · joint: Shoulder, Elbow or Wrist (optional, multi-select) · when (Now · 4:52 PM) · note · **Recent** list (state, area, timestamp) | D-120…D-125 |
| States (Part 3) | First run with no WYNG · no throws yet · nothing this period, *"You last threw on Aug 12"* · loading · WYNG out of range · **sync failed** · Bluetooth off · notifications denied | D-61, D-62, D-63, D-70, D-71, D-72 |

## 2.2 Data catalogue

Each element appears once. The screen labels that differ from the element's name are listed under **Also shown as**, so one value never has two definitions.

### A. Throw

One `Throw` per physical throw. Every per-throw metric is defined here, and every other number in the app is built from these.

| ID | Element | Unit (stored → shown) | Definition |
|---|---|---|---|
| D-10 | Throw ID | UUID, app-generated | Identifies one throw. **The firmware has no throw concept.** The app detects a throw in the recorded IMU data and aligns the three pucks by sleeve time (1.6). The throw keeps its sleeve time, and its session and page range, so the raw data can be found again. |
| D-11 | Pitch number | int | Position of the throw within its session, assigned by the iOS app (A8) |
| D-12 | Captured at | UTC → local time | Time of **ball release** (engine spec `timestamp`). The puck has no wall clock, so: the wall time the app stamped at REC START + (release time − session start `t_us`), in sleeve time (1.6). Accurate to about the Bluetooth latency of the START command. |
| D-13 | Arm speed | mph (A17) | Linear arm velocity: `v = ω · r · C`, where ω is the peak angular velocity (deg/s), r is the athlete's arm radius (D-98) and C converts to mph. The engine spec's C ≈ 0.0568 is wrong; the correct value is ≈ 0.0390 (METRIC_CALCULATIONS 4.5). |
| D-14 | Arm slot | degrees | Forearm elevation above horizontal at ball release: the Motus/PULSE convention. Proposed from published research, not defined by the specs (METRIC_CALCULATIONS AS-3). Also an input to Consistency. |
| D-15 | Acceleration | mph/s (A17) | Linear arm acceleration: `(v(release) − v(foot plant)) / Δt`. The UI currently shows it in g (A17). |
| D-16 | Arm load | Nm | Peak elbow varus torque between foot plant and release. Proposed from published research, not defined by the specs (METRIC_CALCULATIONS AS-4). **Which scale to report is a product decision**: lab scale (≈ 50–95 Nm) or the Motus/PULSE scale the board's 36–42 Nm placeholder uses. |
| D-17 | Live CBR | 0–100 | **Not a per-throw metric but a state.** After throw *k* it is `max(0, Daily CBR (D-06) − γ · session demand (D-05))`. The Live and session charts plot this value after each throw. Shown alongside the day's starting value. |
| D-18 | Pitch type | Wyng label | One of the 16 types in the spec, in this order: **4S** four-seam, **2S** two-seam, **SI** sinker, **FC** cutter, **SL** slider, **ST** sweeper, **CU** curveball, **KC** knuckle curve, **CH** changeup, **FS** splitter, **FO** forkball, **KN** knuckleball, **SV** slurve, **SC** screwball, **CS** slow curve, **EP** eephus. Plus **General Volumetric** for a throw the classifier won't assign. The MLB/Statcast code (e.g. FF for 4S) is kept separately for integrations. Limited to the athlete's repertoire (D-99). |
| D-19 | Type confidence | 0–1 → % | The classifier's top probability (softmax). Empty for General Volumetric throws. |
| D-01 | Performance | 0–100 | Per throw: `0.45 · Execution (D-02) + 0.30 · Output (D-03) + 0.25 · Consistency (D-25)`. The weights are placeholders. |
| D-02 | Execution quality | 0–100 | `100 · (1 − Σ wᵢ · |tᵢ − tᵢ,target| / ΔT_delivery)`: how far the EMG activation times of the three muscles (biceps, flexor-pronator mass, triceps) are from the athlete's target timing (D-133) |
| D-03 | Performance output | 0–100 | `100 · exp(−λ₁ · max(0, v_baseline − v)²)`: arm speed (D-13) against the athlete's context-matched baseline (D-132). Only a drop below the baseline is penalised. |
| D-25 | **Consistency** | 0–100 | Per throw: `100 · exp(−λ₂ · D_M)`, where D_M is the throw's distance from its pitch type's cluster (D-09). **Only for classified throws.** General Volumetric throws have none. |
| D-04 | Stress unit | ≥ 0 | `SU = (ω / ω_cal_max)² · (a / a_cal_max)²`: the throw's demand relative to the athlete's calibrated maxima (D-134). A 50% effort throw scores ≈ 0.0625. **Every** throw counts, including catch play and General Volumetric. |
| D-08 | Feature vector | 4 values | `[pronation velocity, EMG timing gap (flexor-pronator mass vs triceps), slot ratio, acceleration ratio]`. Used for classification (D-18) and Consistency (D-25). |
| D-09 | Mahalanobis distance | ≥ 0 | Distance of D-08 from the pitch type's cluster centre, scaled by its covariance (D-130) |
| D-136 | Classification correction | label + history | The athlete's optional post-session correction of a pitch type, kept with its history |
| — | Measurement quality | flag/score | Whether the throw's data is good enough to count (spec: throw event field). Not defined further. |
| — | Session | reference | The session the throw belongs to, if any (D-40) |

The time the throw was synced is when the app finished offloading it (D-115). The app gives each throw its own UUID (engine spec `throw_id`); the firmware has none.

### B. Scopes and aggregates

**Aggregation rule.** A metric shown for more than one throw is **the average of its per-throw values** over the throws in that scope: the sum of the per-throw values divided by the number of throws. Counts are counted. The few operations that are not averages are defined below, each once.

**Scopes** the screens use: a session; a period chosen with the period control (day, week or month, D-20); a trailing window (7, 28, 30 or 90 days); the last N throws (the Live chart); the last 10 sessions (Durability). A chart is a sequence of values, one per sub-scope (per day, per week, per session or per throw).

| ID | Element | Definition | Also shown as |
|---|---|---|---|
| D-20 | Period | Scope chosen with the period control: day, week or month, plus paging back and forward | Home and Sessions header |
| D-30 | **Metric average** | Average of a throw metric (D-13…D-16, D-01…D-03, D-25) over its **eligible** throws in a scope (A24). CBR is not averaged (D-06). | Home dials, tiles and contributors (period) · Home chart points (per day, week or session) · session tile score and arm speed, session detail tiles (session) · AVG reference lines (throws on the chart) |
| D-31 | **Throw count** | Number of throws in a scope | "Pitch count" (01a tile), "Pitch volume" (01a contributor, 01n), "Throws" (02b, 03b), "42 throws" (03a, 02c) |
| D-32 | Throw count by session type | D-31 split by session type (D-41) | 01n "Where they came from" (period), 07a Volume by session type (30 days, as %) |
| D-06 | **Daily CBR** | 0–100. The athlete's estimated readiness at the start of a day, from the Charge side: time since recent sessions, rest intervals, workload history, recent changes in execution, and optional soreness, sleep and recovery data (D-140, D-120). **No formula yet** (A16). Historical CBR is the series of D-06 and D-17 values over time. | Home CBR dial, CBR card (start-of-day value), Recovery report |
| D-05 | Session demand | `B_live = Σ SU` (D-04) over the day's throws so far | Feeds Live CBR (D-17) |
| D-33 | 30-day range | Minimum and maximum of a metric over the trailing 30 days. The granularity is still to define (A7). | 01b, 01l, 01m "Your range, 30 days" |
| D-34 | Change against a reference | Difference between the same value in two scopes, as a direction, a percentage or points | Contributor bar direction (against the previous period) · 01n "+12% against your 28-day average" · Progress (90 days) · Workload trend "rising" · insight and WYNG Man "gained four points" |
| D-35 | Maximum | Highest value over the athlete's history | Personal bests: peak arm speed (max D-13), best session (max D-30 Performance at session scope), most throws per week (max D-31 per week) |
| D-36 | Reference values | Constants: workload ceiling 1.30, and the metric ranges and chart axes in board §4.2. **CBR is now 0–100**, so the board's 0.80 floor and its Burned / Ready / Charged states no longer apply. New thresholds are not defined yet (A16). | Dashed target lines, the 01k state list, the 07c ceiling |
| D-39 | In-session flags | Warnings raised during a session. **Which flags apply depends on the session type (D-41).** For example, a drop-off in acceleration is a flag in a Bullpen but not in Catch or Warm-Up. A deviation in consistency or mechanics is a pre-injury indicator in a Game but not in Flat Ground. The rules per type and where flags are shown will be revisited later (A5). | Not shown on any screen yet |
| D-37 | Contributor bar fill | Position of a D-30 value within its metric range (D-36), 0–1 | Home contributor bars |
| D-28 | Weight in score | Performance weights: Execution 45%, Output 30%, Consistency 25%. Placeholders, not validated. | 01b, 01l, 01m "Weight in score" (the wireframes still show the old weights, A18) |

### C. Session

A `Session` is a *"label over a range of throws"*. Its figures (throw count, average arm speed, score) are aggregates at session scope (D-30, D-31) and are not stored as separate session fields.

| ID | Element | Definition | Origin |
|---|---|---|---|
| D-40 | Session name | **Auto-generated** as `Type - N - Date`: the session type (D-41), then N, the number of sessions of that type on that date (starting at 1), then the date as Month Day, Year. Example: "Bullpen - 2 - Aug 27, 2026". The athlete can edit it afterwards. | App, editable by the user |
| D-41 | Session type | One of, **in this order**: Catch, Warm-Up, Long Toss, Bullpen, Flat Ground, Weighted Balls / Plyos, Live BP, Game, Rehab, Other. Chosen by the athlete **when they start recording**, not at the end. For a session created by gap detection, chosen when the athlete confirms it. The athlete can change it afterwards (A5). | User |
| D-42 | Start / end | `startedAt`, `endedAt`. Each marked session is one recording: the app stamps wall time at REC START and REC STOP (1.8). | User (marked) or detected (gap detection, A25) |
| D-45 | Duration | `endedAt − startedAt`. Shown as a running timer while a session is active. | Derived |
| D-43 | Origin, confirmed | `origin` (marked, selected or detected) and `confirmed`. A detected session stays unconfirmed, with no type or generated name, until the athlete confirms it and picks the type (A5). | App |
| D-46 | Session effort | The intended effort of the session. The spec scores "like with like" by effort, but doesn't say how effort is captured (A20). | User or derived |
| D-47 | Recording puck(s) | The puck serials (1.3), sleeve ID and per-puck recorder session number (1.8). The REC START `tag` can link the three recordings. | App |
| D-48 | Session location | `location {lat, lng, label}` from the phone's GPS. Not shown on any screen. | App |
| D-49 | Session sparkline | A series across the session's throws. The metric it plots is still to define (A6). | Derived |

### D. Reports

| ID | Element | Definition |
|---|---|---|
| D-50 | Workload | **Acute** = D-31 over the trailing 7 days (412). **Chronic** = D-31 over the trailing 28 days ÷ 4, a weekly average (349). **Ratio** = acute ÷ chronic (1.18). **Throws to ceiling** = ceiling (D-36) × chronic − acute (1.30 × 349 − 412 ≈ 40). Status word (Watch) and trend (rising, D-34). |
| D-51 | Durability | D-30 arm speed per pitch-number bucket (D-11: 1–10, 11–20, …) over the last 10 sessions. Gives the throw where arm speed starts to drop (28) and the drop per 10 throws (−2.1%). |
| D-53 | Recovery | CBR against hours since a heavy day (60+ throws), giving the hours back to charged (CBR over 0.85, D-36). Shown on 07a and on the 01k sheet ("about 38 hrs"). Needs CBR at times when no throws happen (A16). |
| D-29 | Generated text | The Home insight (title, message and the question it seeds in WYNG Man), each report's finding sentence, and 07c "What this means" |
| D-58 | Report selection | Which of the 6 reports are active, pinned, and in what order |

### E. Pucks and connection

Each puck is shown separately. The values the pucks send are **not redefined here**: battery and firmware (1.3), signal strength (1.2). Only app-side data is listed.

| ID | Element | Definition |
|---|---|---|
| D-110 | Puck name | Defaults to the advertised name built from the index and serial tail (1.2), e.g. "WYNG-2-8144", rather than the wireframes' "WYNG 4821". The athlete can rename it. Stored app-side. |
| D-111 | Connected | The phone's Bluetooth link state for that puck |
| D-112 | Puck placement and arm side | Biceps, triceps or flexor, worked out by the app from a recording (1.7, A14). The firmware stores no position and no arm side. Shown as "Right sleeve" and "Arm: Right" (06a, 06c). |
| D-115 | Last sync | When the app last completed an offload. Kept by the app, because the puck has no sync marker (1.8). |
| D-118 | Firmware up to date | The puck's firmware version (1.3) compared with the latest release |
| D-61 | Connection status | "Connected", "Offline, looking for your WYNG" on the Live strip and in the out-of-range state. Built from D-111 for all pucks (A9). |
| D-62 | Sync status | "Synced" or "Couldn't sync": result of the last sync |
| D-63 | Bluetooth permission | iOS Bluetooth authorization |
| D-72 | Ever paired | Whether the app has written a sleeve ID to the pucks with PAIR (ICD §12.1). There is no Bluetooth bonding in this firmware. |

### F. Athlete, account and preferences

| ID | Element | Contract | Origin |
|---|---|---|---|
| D-90 | Name, email, picture | `Athlete.name`, `email` | Account |
| D-91 | Position (Pitcher) | **not in the contract** | User |
| D-92 | Throwing arm | `throwingArm`. Shown as "Right-handed" (05a) and "Throwing arm: Right" (05b). This is the athlete's arm. The arm the sleeve is on is a separate value (D-112) and should agree with it (A14). | User |
| D-93 | Date of birth | `dateOfBirth` | User |
| D-94 | Height, weight | `heightCm`, `weightKg` | User |
| D-95 | Gender | **not in the contract** | User |
| D-96 | Measurements (arm, forearm) | **not in the contract** (A13) | User |
| D-98 | Arm radius | `arm_radius_m`: distance from the centre of rotation to the forearm puck, in m. Used for arm speed (D-13). Taken from the measurements (D-96, A13). | User / derived |
| D-99 | Pitch repertoire | `selected_repertoire`: the pitch types (D-18) the athlete throws, usually 3–4 | User |
| D-97 | Home location | Town, e.g. "Northgate, CA". **Not in the contract.** Unrelated to session location (D-48) and puck placement (D-112). | User |
| D-100 | Units | `units` (imperial or metric). Everything is stored in SI and converted for display. | User |
| D-101 | Appearance | local | App |
| D-102 | Language | local | App |
| D-103 | Notifications | OS permission plus preferences | App/OS |
| D-104 | Health app link | HealthKit authorization | App/OS |
| D-105 | Two factor | Auth | Account |
| D-106 | Contact preferences, data export, delete account | Auth / backend | Account |
| D-107 | App version and build | Bundle | App |
| — | Timezone | `timezone`, not shown | App |

### G. Athlete-entered wellness

| ID | Element | Contract |
|---|---|---|
| D-120 | Feel state | `state`: clear, tight, sore or pain |
| D-121 | Muscle areas | `muscle[]` |
| D-122 | Joint areas | `joint[]` |
| D-123 | Occurred at | `occurredAt` |
| D-124 | Note | `note` |
| D-125 | Recent entries | A list of the above |

### J. Baselines and models

Per-athlete reference data the scores are measured against. Kept separate from the raw data and versioned, so history can be recalculated with a newer model.

| ID | Element | Definition |
|---|---|---|
| D-130 | Pitch baseline | Per pitch type: cluster centre μ and covariance Σ of the feature vector (D-08), the calibration throws it was built from, algorithm version, date refreshed |
| D-131 | Calibration status | Per pitch type: how many labelled calibration throws exist (e.g. 4S: 15, SL: 12) and whether classification is automatic yet. Drives the "calibrating" / "not enough data" states. |
| D-132 | Output baseline | The athlete's expected arm speed for the context (pitch type, effort, session type), used by D-03 |
| D-133 | Execution target | The athlete's target EMG activation timing per muscle, and the weights wᵢ, used by D-02 |
| D-134 | Calibrated maxima | The athlete's maximum angular velocity and acceleration, used by D-04 |
| D-135 | Versions | Classifier/model version (`active_model_ver`), score version and calibration version, stored with every score |
| D-137 | Raw data ledger | The raw recordings, kept unchanged so scores can be recalculated (engine spec §1). **About 237 MB per 28-minute session across 3 pucks** (1.9.5), or about 12 MB if only 2 s around each throw is kept (A27). |

### K. Optional external data

Never required. Missing data must not lower a score.

| ID | Element | Definition |
|---|---|---|
| D-140 | Recovery data | From WHOOP, Oura or Apple Health / Watch: HRV, resting heart rate, sleep duration and quality, activity/strain, the platform's own readiness score. Feeds Daily CBR (D-06). |
| D-141 | Pitch-tracking data | From Rapsodo or TrackMan: ball velocity, spin, movement. Optional input to Performance. |
| D-142 | Missing-input flags | Which optional inputs were unavailable for a score, so the app can say so |

### H. Assistant (WYNG Man)

| ID | Element | Notes |
|---|---|---|
| D-80 | Conversation | User and assistant messages, grouped by day |
| D-81 | Suggestions | Opening and follow-up chips |
| D-82 | Structured answer | Claim, numbers, and a link to the source screen. The board says to render this *"from a typed response, not as markdown"*. |
| D-83 | Context payload | The throws, sessions, aggregates and reports given to the model |

### I. Empty and state data

| ID | Element | Built from |
|---|---|---|
| D-70 | Has any throws, has any in this period | D-31 |
| D-71 | Last throw date ("You last threw on Aug 12") | Latest D-12 |

## 2.3 Lineage: raw → UI metric

> **Preliminary.** This is groundwork for the next phase, which checks whether each UI metric can be derived from the raw data and what is missing. That work has started in [METRIC_CALCULATIONS.md](METRIC_CALCULATIONS.md).

The ICD defines no metric algorithm. The table uses the raw streams in ICD rev 0.4 (1.9), and METRIC_CALCULATIONS.md has the methods.

| UI metric | Board definition | Likely raw inputs | Also needs |
|---|---|---|---|
| Arm speed (D-13) | *"Peak angular speed of your arm through the throw. Not ball velocity."* | `g`, flexor puck. Clips at ±4000 dps. | A segment length to convert angular velocity into the stored m/s, taken from the athlete-entered arm and forearm lengths (D-96, A4). |
| Acceleration (D-15) | *"How hard the arm gets up to speed"* | `g` through v(t), per the spec formula. `h` (±320 g) marks release and locates the throw. | Calibration IMU bias |
| Arm slot (D-14) | Release angle in degrees | Orientation the app fuses from `a` and `g`, flexor puck, at release. No magnetometer until firmware v0.9. | Sensor-to-segment alignment |
| Arm load (D-16) | Nm, the stress on the arm | Elbow angle from app-fused orientation and ToF `le` / `range` (1.9.4), `g` and `h` | Lever arms, **height and weight** (inverse dynamics) |
| CBR (D-06, D-17) | Charge Burn Ratio: readiness from workload and recovery | Live: ω and acceleration per throw (via D-04). Daily: throw history, rest intervals, optional recovery data | Live CBR defined (METRIC_CALCULATIONS 4.14). Daily CBR has no formula yet (A16). |
| Pitch type and confidence (D-18/19) | One of 16 types, or General Volumetric | Feature vector (D-08): forearm pronation velocity, EMG timing (flexor-pronator mass vs triceps), slot ratio, acceleration ratio | Per-athlete calibration (D-130). METRIC_CALCULATIONS 4.11. |
| Consistency (D-25) | Pitch-specific repeatability, 0–100 | Mahalanobis distance of the feature vector from the pitch's cluster | METRIC_CALCULATIONS 4.12 |
| Performance (D-01) | 0.45 Execution + 0.30 Output + 0.25 Consistency | EMG timing (D-02), arm speed vs baseline (D-03), Consistency (D-25) | Baselines (J). METRIC_CALCULATIONS 4.13. |
| Workload, Durability, Recovery, PBs, Progress | See D-50, D-51, D-53, D-35, D-34 | Derived from throws and sessions | History: workload needs ≥ 28 days, Progress 90 days |
| Temperature, EMG envelope, elbow angle, contact | **Not shown on any screen** | Raw (1.9, 1.10). The app calculates the elbow angle from its own fused orientation and the ToF ranges. | — |

## 2.4 Screens the raw data needs that the wireframes do not have

Several firmware functions produce data or need user interaction, but no wireframe covers them. **The UI must cover each of these gaps.**

| Firmware function | Raw data involved | Wireframe status |
|---|---|---|
| Calibration | Recordings during a calibration routine. The pucks store no calibration (1.7). | **Missing.** An app-only flow if needed, e.g. a static pose for sensor alignment (METRIC_CALCULATIONS OP-3). Placement is not confirmed here: it has no UI (A14). |
| Live view during a session | PUCK at 1 Hz, EMG stream, ToF MEAS (1.10) | **Missing.** Live shows per-throw results, which firmware v0.7.0 can't deliver during a recording (A26). |
| Pairing the 3 pucks as one sleeve (ICD §12.1) | Advertising (1.2), PUCK `heard`, `sync` | 06a–06c pair **one "WYNG"**. The app has to write PAIR (`sleeve_id`, `n`, `idx`) to each of 3 pucks. Puck placement has no screen of its own (A14). |
| Firmware update with progress and per-puck confirm (1.12) | SMP upload progress, `PUCK.sflags`, version | Listed as a *future feature* in board Part 5. **The app must confirm each puck**, or the update reverts. |
| Offload progress, recording room left, full-NAND stop | 1.5, 1.8 | Only "SYNCED" and "Couldn't sync" exist. Offload can take hours (A27), so progress is essential. |
| Low battery, and a recording stopped by it | `soc_q8`, `charge`, END reason 2 (1.3, 1.8) | Missing. The 30% / 15% levels aren't in the firmware; the app can apply its own to `soc_q8`. |
| Arm and forearm lengths | Not puck data: used by the app's calculations (A13) | Collected in onboarding. The 05b row should open the same fields. |
| Unpair and erase | PAIR with `sleeve_id` 0, REC ERASE, REBOOT (ICD §6.2, §10.2) | Missing (06a only has Rename). The factory-reset gesture is planned for firmware v0.8. |

---

# Open issues

Open questions about the data the UI needs. The rest of the document refers to them by number. Numbers are kept stable, so there are gaps where issues were closed or moved out of scope. Questions about the firmware and the phone↔puck communication are tracked in [APP_FIRMWARE_OPEN_QUESTIONS.md](APP_FIRMWARE_OPEN_QUESTIONS.md). A25–A27 here cover what firmware v0.7.0 means for the UI.

| # | Issue |
|---|---|
| A3 | **Resolved, revised by the specs.** Arm load is calculated per throw; its definition is still assumed (METRIC_CALCULATIONS AS-4). CBR is **not** a per-throw metric. Live CBR (D-17) is a state updated after every throw. |
| A4 | **Decided.** The athlete enters arm and forearm lengths (A13). They feed the app's own calculations: arm speed, lever arms and arm load. **Nothing is written to the pucks**, because firmware v0.7.0 has no calibration service. |
| A5 | **Session type and name: decided, UI needs changing.** (1) **Type list, in this order:** Catch, Warm-Up, Long Toss, Bullpen, Flat Ground, Weighted Balls / Plyos, Live BP, Game, Rehab, Other. (2) **Type is chosen when the athlete starts recording** (Start a session, 02a), not at the end. Knowing the type during the session lets the app give accurate feedback and type-specific flags (D-39). (3) **The name is generated** from the type, the count of that type on the day, and the date, e.g. "Bullpen - 2 - Aug 27, 2026" (D-40). **The current wireframes conflict with this:** they set the name at the end of the session (02c "Edit session", 03d "Rename") and have no type selection at the start. Type also drives 01n "Where they came from", the 07a/07d "Volume by session type" report and the 01g Day chart axis, so every type needs a colour in the session-type palette. **Decided:** the athlete can edit the generated name and change the type after the session (03d). **Decided:** sessions created by gap detection have no type until the athlete confirms them. The confirmation prompt (the auto-log notification for unconfirmed sessions) asks for the type, and the name is generated once the type is set. Until then the session is shown as untyped. **Noted, to revisit later:** where in-session flags are shown (the board lists "Live alerts mid session" as a future feature because "the phone is in a bag"), and which flags apply to which type. |
| A6 | Define which metric the session tile **sparkline** (D-49) plots across the session's throws. |
| A7 | Consistency now has a per-throw value (D-25), so it averages like the other metrics. **Still open:** the granularity of the 30-day range (D-33), per throw or per day. |
| A8 | **Decided.** The pitch number is per session and assigned by the iOS app, which matches 03b ("All 42 throws", numbered 42 → 34). Change the board's `Throw.sequence` definition from "index within the day" to "index within the session". **Still open:** what Live shows for throws not in any session. 02a lists pitches 35–42 with no session marked, and 02d shows throw 42 as "Session: Unassigned". Also, if a throw is later moved to another session or a session is split (a future feature), the numbers in the affected sessions have to be reassigned. |
| A9 | **Decided:** each puck is shown separately, with its own battery, firmware, last sync, placement and name. **Still open:** the Live strip (02a–02f), the "Synced" line (02c) and the out-of-range state show a single connection status (D-61). Define it from the pucks' link states, e.g. "connected" only when every puck is connected, or show one status per puck. The app also has to check `rec_state` on each puck, because a puck that missed REC START isn't recording. |
| A13 | **Measurements.** Upper arm and forearm length are already collected in onboarding, and can be pre-filled from height. **Still needed:** the "Measurements" row on My profile (05b) should open the same fields. |
| A14 | **Decided: puck placement has no UI, and the firmware agrees.** Pucks carry no position, and capture doesn't depend on it (ICD INV-ROLE-1). The app works it out from a recording using ICD §12.4: which pucks range with each other, plus their IMU motion. **Still open:** arm side, which ICD §12.4 doesn't cover. Work it out from the rotation direction in a throw, or take it from the profile's throwing arm (D-92). |
| A15 | **Resolved.** Performance is defined per throw (D-01), with Consistency per throw from the Mahalanobis distance (D-25). Session Performance is the average over eligible throws (A24). |
| A16 | **CBR: partly defined.** Live CBR (D-17) has a formula. **Still open:** (1) the Daily CBR formula (D-06), meaning the Charge side and how recovery raises CBR between days; (2) the scale factor γ; (3) thresholds and states on the new 0–100 scale; (4) the Recovery report's "hours back to charged" (D-53), which depends on (1); (5) which CBR value Home shows for a week or month, since CBR is a state and is not averaged. |
| A17 | **Units.** The engine spec stores arm speed in mph and acceleration in mph/s. The board's data contract stores SI (m/s, m/s²), and the UI shows acceleration in **g**: "39.6 g" is about 869 mph/s. Decide the stored unit and the display unit for acceleration, and update the chart axis (0–60 g). The engine spec's conversion constant C is also wrong (D-13). |
| A18 | **The wireframes need updating for the new definitions.** (1) **CBR** moves to 0–100: the Home dial (0.82), the CBR chart and its 0.80 target, the 01k Burned/Ready/Charged sheet and the Recovery report all change. The spec's CBR card also needs the start-of-day value, the live value, a workload trend and indicators for optional inputs. (2) **Home contributors:** Performance is now Execution, Output and Consistency, not the five contributors on 01a. The weight sheets (28% / 22% / 18%) are wrong. (3) **Consistency card**, with an overall score, a score per pitch type, notable deviations and a session trend: no screen has this. (4) **Pitch types:** 16 types instead of 5. Curveball becomes **CU** (the board uses CB), the pitch palette has colours for only 5 types, and General Volumetric throws need a display. (5) **"Calibrating" / "not enough data" states** for every score while baselines are being built. |
| A19 | **Keep the Workload (ACWR) report?** CBR is designed as an alternative to ACWR, and must not be calculated from it. The Workload report (D-50, 07a/07c) is built on ACWR. Keep it as a separate report, or drop it? |
| A20 | **Session effort.** Performance and Consistency compare "like with like" by intended effort, and the spec stores effort with the session, but nothing defines how effort is captured: entered by the athlete, implied by the session type, or derived from the stress units. |
| A21 | **Missing UI for pitch classification and integrations:** repertoire selection (D-99), guided labelled calibration throws (about 10–30 per pitch, spread over normal sessions), calibration progress (D-131), optional post-session correction of pitch labels (D-136), re-calibration when a pitch is added, and connecting WHOOP, Oura or a pitch tracker (D-140, D-141). Settings has only a Health app toggle. |
| A22 | **Session types differ.** The engine spec's `session_category` has 5 values ('Catch Play', 'Weighted Ball', 'Warmup', 'Bullpen', 'Game'). The agreed list has 10 (D-41). The engine spec also stores the category on each throw. Use the 10-type list, stored on the session. |
| A23 | **Arm radius vs measurements.** The engine uses one value, `arm_radius_m`, from the centre of rotation to the forearm puck. A13 collects arm and forearm length. Define which measurement the athlete enters, and whether the centre of rotation is the elbow or the shoulder. |
| A24 | **Eligible throws.** The specs aggregate "eligible pitches" into the session Performance and Consistency scores. Define eligibility: classified throws only, which session types (e.g. are Warm-Up and Catch included?), and the measurement-quality threshold. |
| A25 | **The sleeve doesn't always record.** The board says *"the sleeve never stops"*, and that sessions are labels over a continuous feed. In firmware v0.7.0 the pucks record **only between REC START and REC STOP**, with about 47 minutes of room per puck. A throw outside a recording isn't captured and can't count towards workload or CBR. This affects: "Start a session" (it now starts capture, which fits choosing the type at Record, A5), gap-detected sessions and unassigned throws (A5, A8), the 02f copy *"anything thrown while it was off syncs"*, and a Game longer than 47 minutes. Decide: keep sessions as explicit recordings and change the UI, or ask the firmware team for always-on capture. |
| A26 | **Live can't show per-throw metrics during a recording.** The recorder refuses an offload while recording, and there's no throw event. Available live: PUCK at 1 Hz (peak angular velocity, peak acceleration, clip counts, EMG RMS), EMG stream and ToF ranges (1.10). Live could show an **approximate** throw count, arm speed and stress from the one-second peaks. Arm slot, arm load, pitch type, Consistency and Performance have to wait for the offload. Firmware v0.10 plans a realtime service. Decide what Live shows and whether Live CBR is an estimate until the offload. |
| A27 | **Offload time.** At the measured 26 kB/s, a 28-minute bullpen takes about 2.5 hours to download from 3 pucks one at a time (1.9.5), against 7.6 minutes for 2 s around each throw. The 02c "Session saved · Synced" screen and any same-day score depend on this. Phone throughput isn't measured yet. Needs: a measurement on iPhone, then an on-puck throw index or L2CAP if it stays this slow. |
