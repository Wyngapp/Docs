# WYNG Metric Calculations

How the app gets from the raw puck data ([DATA_CONSOLIDATION.md](DATA_CONSOLIDATION.md) Section 1) to the metrics the UI shows (Section 2). This is a **planning document**. It records what each calculation needs, the method we intend to use, and what is still unknown. It is not an implementation spec.

**Status:** Draft 0.3, 6 Nov 2026. Inputs updated to ICD rev 0.4 (firmware v0.7.0 as built). Updated for the Core Metrics & Pitch Classification Developer Spec v3 and the Data Engine Specification v3.0. The engine spec's footer says it may contain mistakes. Section 5 lists the errors and gaps found in it. Proposals marked **(research)** come from [Pitching sensor metric definitions](../reports/Pitching%20sensor%20metric%20definitions.md), a literature review (3 Nov 2026) with sources. They are proposals to confirm, not validated methods.

**Data IDs** (`D-nn`) refer to the catalogue in DATA_CONSOLIDATION.md 2.2. **Record fields** (`a`, `g`, `h`, `le`, …) refer to ICD rev 0.4, as summarised in DATA_CONSOLIDATION.md 1.9–1.10. **The app does all of this processing:** the pucks only record raw streams. Whether it runs on the phone or the server is still TBD.

---

## 1. Scope

| Covered here | Not covered yet |
|---|---|
| Session decoding and **throw detection** · puck placement · elbow angle · release point · **arm speed** · **acceleration** · **arm slot** · **arm load** · **pitch classification and confidence** · **Consistency** · **Performance** · **Live CBR** · aggregates · session gap detection | **Validated methods:** nothing here has been tested on real recordings. **Daily CBR** has a proposed structure only (4.14), because the specs give no formula (DATA_CONSOLIDATION A16). |

Arm speed, acceleration, the scores and Live CBR follow the two specs. Arm slot and arm load are still based on assumed definitions (AS-3, AS-4), because neither spec defines them.

## 2. Assumptions to confirm

The board's wording is not precise enough to calculate from, so each metric below rests on an assumed definition. **Every method in this document depends on these.** Change an assumption and the method changes with it.

| # | Assumption | Basis |
|---|---|---|
| AS-1 | ~~Arm speed = peak forearm angular velocity × forearm-plus-hand length.~~ **Superseded** by the engine spec: `v = ω · r · C` with r = `arm_radius_m` (4.5). | Engine spec §3 |
| AS-2 | ~~Acceleration = peak linear acceleration magnitude of the forearm puck, in g.~~ **Superseded** by the engine spec: the change in arm velocity from foot plant to release, in mph/s (4.6). | Engine spec §3 |
| AS-3 | **Arm slot** is the elevation of the forearm's long axis (elbow → wrist) above horizontal at ball release, in degrees. 0° is sidearm, about 45° three-quarter, negative submarine. **(research)** This is the Motus / Driveline PULSE convention. Lab studies use a different angle (shoulder-to-hand line from vertical), which doesn't convert with a simple 90° − θ. Category cutoffs for this convention are unpublished, so labels are provisional: over-the-top > 60°, three-quarter 30–60°, sidearm −10° to 30°, submarine < −10°. | Board: release angle. Research report §Arm slot |
| AS-4 | **Arm load** is the peak net internal elbow varus torque between foot plant and release, in Nm, from two-segment Newton–Euler inverse dynamics (4.8). **(research)** It is not the load on the UCL ligament. **Scale is a product decision:** this method gives lab-scale values (high school ≈ 50–70 Nm, pros ≈ 80–95 Nm). The board's 36–42 Nm placeholder is on the Motus/PULSE scale, which reads about 40% low. Proposed: report WYNG's own value, labelled "estimated", mainly against the athlete's baseline. | Board: Nm. Research report §Arm load |
| AS-5 | **Ball release** is the peak of the forearm's resultant acceleration, refined by the sharp drop in acceleration when the ball leaves the hand, and cross-checked against the angular-velocity peak. **(research)** Release follows maximum external rotation by 20–50 ms. No published study reports the timing accuracy of this from an IMU; expect ±5–10 ms, unmeasured. | Research report §Events |
| AS-6 | The data from the forearm puck drives arm speed, acceleration and arm slot. The upper-arm pucks are needed for the elbow angle and arm load. | Follows from AS-1 to AS-4 |

**Parameters** (starting values, to tune against data):

| # | Parameter | Starting value | Used in |
|---|---|---|---|
| P-1 | Session gap: a gap between throws longer than this starts a new detected session | 20 min | 4.10 |
| P-2 | Throws used to decide placement | first 5 throws of a recording | 4.2 |
| P-3 | Ball mass | 0.142–0.145 kg | 4.8 |
| P-4 | λ₁, how steeply Output falls when arm speed is below the baseline | not given | 4.13 |
| P-5 | λ₂ for Consistency, if the exponential form is kept | **(research)** ln2 / median distance for the cluster's size: 0.278, 0.315, 0.332, 0.348 for n = 10, 15, 20, 30. The research prefers a percentile form instead (4.12). | 4.12 |
| P-6 | γ, how much session demand lowers Live CBR | not given | 4.14 |
| P-7 | wᵢ, weight of each muscle's timing error in Execution | not given | 4.13 |
| P-8 | Classification probability threshold | **0.85** (engine spec) | 4.11 |
| P-9 | Maximum Mahalanobis distance to belong to a cluster | **(research)** Per cluster: the 99th percentile of held-out (leave-one-session-out) D², floored at 13.28 (χ²₄ at 99%) and capped at the small-sample F value for that n (e.g. D ≤ 7.77 at n = 10, 4.37 at n = 30) | 4.11 |
| P-10 | Labelled calibration throws per pitch type | **10–30** (both specs, research targets) | 4.11 |
| P-11 | Foot plant estimate | **(research)** release − 150 ms (or max external rotation − 120 ms), ±40 ms, labelled "estimated" | 4.6, 4.8 |
| P-12 | Daily CBR fatigue time constant τF | **(research)** 1–1.5 days, optional slow component τ ≈ 4 days at weight ≈ 0.3. Calibrated per age band to the Pitch Smart rest tables. | 4.14 |
| P-13 | Daily CBR tolerance time constant | **(research)** 28–42 days, uncoupled from the fatigue window | 4.14 |
| P-15 | Throw detection threshold and window | Not given. The window is the app's choice: start with 1 s either side of the peak. The threshold has to catch light catch-play throws, because every throw counts for CBR. | 4.1 |
| P-14 | Low-pass cut-off before differentiating ω | **(research)** one fixed value in 20–50 Hz, version-locked; it moves peak torque | 4.8 |

## 3. Pipeline

```
 One continuous recording per puck (offloaded after REC STOP)
   │
   ├─4.1  Decode & find throws ─ decode records, sleeve time, detect throws
   ├─4.2  Placement ─────────── which puck is biceps, triceps, flexor; arm side
   ├─4.3  Segment orientation & elbow angle
   ├─4.4  Release point ─────── the moment the metrics are read at
   │
   ├─4.5  Arm speed      (D-13)
   ├─4.6  Acceleration   (D-15)
   ├─4.7  Arm slot       (D-14)
   ├─4.8  Arm load       (D-16)
   │
   ├─4.11 Pitch classification ─ feature vector, cluster match, confidence
   ├─4.12 Consistency    (D-25)
   ├─4.13 Performance    (D-01, with Execution D-02 and Output D-03)
   ├─4.14 Live CBR       (D-17, from stress units D-04)
   │
   ├─4.9  Aggregates ────────── counts, averages, workload, durability, ranges, bests
   └─4.10 Gap detection ─────── groups throws into detected sessions
```

The order matters. Placement (4.2) has to come first, because every metric depends on knowing which puck is on the forearm.

## 4. Calculations

Each calculation lists its **inputs**, the **method**, what it **depends on**, its **output**, how to **validate** it, and its **open points**.

### 4.1 Session decoding and throw detection

**The firmware has no throws, so this step is now the app's.** ICD §11 and its reference decoder (`ble-client/wyng_rec_decode.py`) cover steps 1–3.

**Inputs:** the offloaded pages of one session from each puck (DATA_CONSOLIDATION 1.8–1.9).

**Method:**
1. **Reassemble and check.** Rebuild pages, check each page's CRC, and resume decoding at the next page's `first` after a bad page. Record the END reason and any DROP records.
2. **Decode:**
   - `a`, `g`, `h`: multiply by the scales in the SESSION record (488 µg, 140 mdps and 10,417 µg per LSB).
   - EMG: µV = code × `lsb_nv` / 1000. The app computes its own envelope.
   - ToF: mm = value × 0.02. A value of −32768 means none.
3. **Time.** IMU samples: a line through neighbouring anchors, split at resync blocks. EMG: from `t_us` and `blk_last`. Then map all three pucks to sleeve time with the latest SYNC record.
4. **Link the recordings.** Match the three pucks' sessions by `sleeve` and `tag`. If a puck's session is missing or ended early, mark the affected throws partial.
5. **Find throws.** On the flexor puck, find peaks in |h| above a threshold (P-15), and cut a window around each in sleeve time. Each one becomes a throw with an app-generated ID (D-10).
6. **Captured at** (D-12): the wall time the app stamped at REC START + (release − session start).

**Output:** one throw record holding each puck's time-aligned, scaled samples and ToF events.

**Validate:** throws counted by hand from video, including low-effort throws and non-throwing arm movements (e.g. catching, shaking out the arm). False counts feed straight into CBR.

**Open points:** R-15 (throw detector), R-16 (data volume).

### 4.2 Puck placement from IMU profiles

There is no placement UI (A14), and the pucks store no position (ICD INV-ROLE-1). **ICD §12.4 now gives the method**, and capture doesn't depend on it.

**Inputs:** a recording from all pucks: ToF records and IMU data.

**Method (ICD §12.4):**
1. **Ranging pair.** The two pucks that detect each other's ultrasound (RX edges by transmitter in the TOF records) are the upper-arm and forearm pair, biceps and flexor. The puck left out is the **triceps**.
2. **Biceps vs flexor.** The two upper-arm pucks move together in the IMU data, so the ranging puck that moves with the triceps is the **biceps**, and the other is the **flexor**.
3. **Arm side (not in the ICD).** Proposed: the rotation direction during a throw is mirrored between right- and left-handed throws. Check it against the profile's throwing arm (D-92).
4. **Cross-check:** the flexor puck should have the larger peak angular velocity and acceleration, and the triceps and biceps EMG timing should differ.
5. Recompute for each recording. Pucks can be moved between recordings, and the session record doesn't say otherwise.

**Output:** the role and arm side of each puck, used by every step below.

**Validate:** recordings with known placement, including deliberately swapped pucks.

**Open points:** how a sleeve put on the wrong way round shows up in the data. Whether a recording with no movement yet can be placed.

### 4.3 Segment orientation and elbow angle

The elbow angle feeds arm load (4.8).

**Inputs:** orientation the app fuses from `a` and `g`; there are no on-puck quaternions. ToF `le` / `range` between the biceps and flexor pucks at 200 Hz (1.9.4). The athlete's arm and forearm measurements (D-96).

**Method:** two independent estimates, to use together.
1. **From orientation.** Express each puck's orientation as its segment's orientation (sensor-to-segment alignment), then take the relative rotation between forearm and upper arm. The flexion angle is the rotation about the elbow's hinge axis.
   - **Difficulty:** with no magnetometer (planned for firmware v0.9), each puck's fused orientation has its own unrelated heading (yaw) reference, so orientations can't be compared directly. Either align the headings with a known pose, or estimate the hinge axis from the gyro data, using the fact that the elbow mostly bends about one axis. A published method for this is joint-axis estimation for hinge joints from two IMUs (Seel et al., 2014).
2. **From the ultrasonic range.** With `a` the upper-arm puck's distance to the elbow, `b` the forearm puck's, and `d` the measured distance between the two pucks: `d² = a² + b² − 2ab·cos θ`, where θ is the angle at the elbow. Flexion = 180° − θ. This needs `a` and `b`, the lever arms, which the app has to supply. The firmware doesn't use them. **ToF is a 200 Hz series** (5 ms spacing): about 30 samples across the 150 ms delivery, but only about 6 between max external rotation and release. That's enough to correct the drift of estimate 1, and coarse for the fastest part of the throw. The transmitter can also change at each discovery.

**Output:** the elbow flexion angle over the throw. It is not shown on any screen.

**Validate:** a goniometer at static poses. Optical motion capture for throws.

**Open points:** OP-3 (heading alignment), the source of the lever arms (Q14), speed of sound (R-17).

### 4.4 Release point and event timing

**Inputs:** forearm acceleration and angular velocity; upper-arm angular velocity.

**Method (research):** with typical phase timing from published biomechanics (foot plant → release ≈ 145–160 ms, max external rotation → release ≈ 20–30 ms, stable from youth to pro):

| Event | Detection | Prior (t = 0 at release) |
|---|---|---|
| Release | Peak of forearm |h| (the ±320 g channel; low-g clips at 16 g), refined by the post-release drop; cross-check with the angular-velocity peak and elbow extension between pucks | 0 |
| Max external rotation (MER) | Sign change of the upper arm's rotation about its long axis, external → internal | −30 ms (−20 to −50) |
| Foot plant | **Not detectable from arm sensors** in any published method. Estimate as release − 150 ms (P-11). Use only to bound search windows, never as a displayed metric, until validated. | −150 ms (−130 to −180) |

**Output:** `t_release`, `t_MER`, `t_foot_plant_est`, in sleeve time.

**Validate:** ≥ 1000 fps video, synchronised. Motion capture lags and flattens the peaks, so it is a biased reference for timing.

### 4.5 Arm speed (D-13)

**Definition (engine spec §3):** `v_mph = ω · r · C`
- ω: angular velocity in deg/s. The engine stores its peak as `raw_max_omega`.
- r: `arm_radius_m`, "arm segment length measured from centre of rotation to the forearm sensor" (D-98).
- C: converts deg/s × m to mph. **The spec's C ≈ 0.0568 is wrong.** The correct value is π/180 × 2.23694 ≈ **0.0390**. With 0.0568, a throw at 5,000 °/s and r = 0.40 m reads 114 mph instead of 78 mph. 0.0568 is close to the factor for ft/s (0.0573), which may be where it came from.

**Inputs:** flexor puck `g`. Arm radius (D-98).

**Method:**
1. |ω| per sample, in deg/s.
2. ω_peak = max |ω| over the window, stored as `raw_max_omega`.
3. v = ω_peak × r × 0.0390, in mph. The spec also uses v(t) as a time series for acceleration (4.6).

**Ambiguities to resolve:**
- **Which ω?** The spec says "taken from the differential rotation across the arm segment". That could mean the forearm's absolute angular velocity, or the forearm's velocity relative to the upper arm (the elbow). These give very different numbers.
- **Which centre of rotation?** This decides r. For the board's typical 74–82 mph at ~5,000 °/s, r must be about 0.4 m, which fits the **shoulder** better than the elbow (DATA_CONSOLIDATION A23).

**Output:** `calc_velocity_mph` (DATA_CONSOLIDATION A17 on units).

**Validate:** optical motion capture.

**Risk:** gyro saturation (R-1). This is now confirmed as a real limit: the gyro range is ±4000 dps, so a clipped throw can't read above about 4,000 °/s × r × 0.0390 per axis.

### 4.6 Acceleration (D-15)

**Definition (engine spec §3):** `a = (v(t_release) − v(t_foot_plant)) / Δt`, in mph/s, where Δt = t_release − t_foot_plant and v(t) comes from 4.5.

**Inputs:** v(t) over the window, `t_release` (4.4), `t_foot_plant`.

**Method:**
1. Compute v(t) for every sample (4.5).
2. Find `t_foot_plant` and `t_release` in the window.
3. a = (v(t_release) − v(t_foot_plant)) / (t_release − t_foot_plant).

**Open points:**
- **Foot plant can't be observed directly.** The pucks are on the arm, with no sensor on the legs or trunk. A proxy is needed, for example the start of the rapid arm acceleration phase or an EMG onset. It has to be validated against video (R-6).
- This is an average acceleration over an interval, not a peak, so it is **not** the "39.6 g" the UI shows. 1 g ≈ 21.9 mph/s (DATA_CONSOLIDATION A17).
- The formula uses v(t) from the gyro, so **gyro clipping (R-1) directly affects this metric** when the arm is near release. The ±320 g channel `h` is available if the definition returns to a measured peak.

### 4.7 Arm slot (D-14)

**Assumed definition:** AS-3.

**Inputs:** forearm orientation, at `t_release` (4.4).

**Method:**
1. Calibrate sensor-to-segment alignment with a static pose: arm hanging, then forearm horizontal **(research)**.
2. Fuse orientation on the forearm puck. Re-anchor gravity in a quiet moment **before** the throw, which the continuous recording provides, then gate out the accelerometer correction during the delivery (|a| ≫ 1 g) and integrate the gyro through the ~150 ms delivery.
3. At `t_release`: arm slot = `asin(forearm_axis · up)`, in degrees. Yaw doesn't enter this formula, so the missing magnetometer costs nothing here.
4. Optional secondary "lab-style" value: the shoulder-to-hand angle from both segments' orientations and lengths.
5. Withhold the value if the gyro clipped during the integration span (R-1).

**Expected accuracy (research):** ±5–10° absolute, a few degrees within one pitcher. Published sleeve comparisons: 7–10° low with R = 0.975 (Driveline co-authored), and a 5° mean difference (independent). Strap rotation is the main error source, and trunk lean changes a gravity-referenced slot.

**Output:** `armSlot`, degrees.

**Validate:** high-speed video from behind the pitcher, with the forearm angle measured at release.

**Open points:**
- **Gravity reference: largely resolved.** Rev 0.4 records continuously, so the quiet moment before a delivery is available, rather than only the 1 s window of rev 0.3. In practice the time between that moment and release has to stay short enough for gyro drift. The app has to do the fusion itself, because there are no on-puck quaternions.
- Sensor-to-segment alignment (OP-3).

### 4.8 Arm load (D-16)

**Assumed definition:** AS-4. This is the least certain of the four. The definition should be confirmed before any work starts.

**Inputs:**
- Forearm and upper-arm orientation and angular velocity (4.3)
- Forearm acceleration
- Elbow angle (4.3)
- Athlete height, weight and segment lengths (D-94, D-96)
- Ball mass (P-3)

**Method (research):** two-segment Newton–Euler inverse dynamics on the forearm, hand and ball, about the elbow. Segment parameters from de Leva (1996), scaled to the athlete: forearm 1.62% of body mass with its centre of mass at 45.74% of forearm length from the elbow; hand 0.61% of body mass, centre of mass at 79% of hand length from the wrist (male values); ball 0.142 kg at the hand until release. Search for the peak between the estimated foot plant (P-11) and release. Each 1 cm error in the elbow-to-puck offset shifts torque by about 3%, so a per-donning calibration of that offset (using the ultrasonic distances) matters more than refining the segment table. Keep EMG out of the torque number: no published method uses it, and net torque needs only kinematics. Never show the cadaver UCL failure load (30–35 Nm) as a risk line: pitchers exceed it on almost every throw.

**Earlier outline:** an inverse-dynamics estimate on the forearm + hand + ball as one rigid body, pivoting at the elbow.
1. Segment mass, centre-of-mass position and moment of inertia come from published anthropometric tables, scaled by body weight and the measured segment lengths. Examples are Dempster's or de Leva's (1996) segment parameters.
2. The elbow position comes from the upper-arm segment and its length. The forearm's angular velocity and angular acceleration come from its puck (angular acceleration by differentiating ω, with smoothing).
3. Newton–Euler: the torque at the elbow that produces the segment's motion = I·α + ω × (I·ω) + r_com × m·a_com, plus the ball's contribution until release.
4. Take the component about the varus axis. Arm load = its peak over the window, which normally occurs near maximum shoulder external rotation, before release.

**Output:** `armLoad`, Nm.

**Validate:** this needs a motion-capture lab running the same inverse dynamics. No simpler reference exists.

**Open points:**
- Confirm AS-4.
- Segment tables from adult populations may misfit youth athletes (the profile's DOB example is 2009).
- The EMG signal could refine the estimate, since muscle activity shares the load with the ligament. That is out of scope for a first version.
- Differentiating ω amplifies noise, so it needs filtering.

### 4.9 Aggregates

These follow the aggregation rule in DATA_CONSOLIDATION.md 2.2 B. For a scope S containing throws t₁…tₙ:

| Element | Formula | Needs |
|---|---|---|
| Throw count (D-31) | n | — |
| Count by session type (D-32) | n per D-41 value, or as a share of n | Session type |
| Metric average (D-30) | (1/n) Σ m(tᵢ) over the eligible throws, for arm speed, acceleration, arm slot, arm load, Performance and its components, Consistency (overall and per pitch type). CBR is a state and isn't averaged. | 4.5–4.8, 4.12, 4.13 |
| Workload (D-50), if kept (DATA_CONSOLIDATION A19) | acute = n over the last 7 days · chronic = (n over the last 28 days) / 4 · ratio = acute / chronic · throws to ceiling = 1.30 × chronic − acute | Counts only |
| Durability (D-51) | D-30 arm speed per pitch-number bucket (1–10, 11–20, …) over the last 10 sessions · drop-off = last bucket before a sustained fall · decay = slope after it, in % per 10 throws | 4.5, pitch number (D-11) |
| 30-day range (D-33) | min and max of the metric over the last 30 days | Granularity still open (DATA_CONSOLIDATION A7) |
| Change against a reference (D-34) | value(S) − value(S_ref), or as % of value(S_ref) | — |
| Maximum (D-35) | peak arm speed = max m(tᵢ) over all throws · most throws per week = max weekly n | 4.5 |

Rest days have no throws. They are left out of averages and drawn as gaps in charts, not as zeros.

### 4.10 Gap detection (detected sessions)

**Inputs:** captured-at times (D-12) of throws not in any marked session.

**Method:** sort by time, then start a new session wherever two throws are more than P-1 apart. Each group becomes a detected session (`origin = detected`, `confirmed = false`). It has no type or name until the athlete confirms it and picks a type (DATA_CONSOLIDATION A5).

**Open points:** tune P-1 against real usage, e.g. a long-toss session with breaks.

### 4.11 Pitch classification and confidence (D-18, D-19)

**Definition (developer spec §05, engine spec §4):** an individual model per athlete. Each pitch type in the athlete's repertoire (D-99) gets a cluster, built from labelled calibration throws (P-10). A new throw is assigned to a type only when it is confidently close to that type's cluster. Otherwise it is **General Volumetric**: it still counts for volume and CBR, but not for pitch-specific scores.

**Feature vector x (D-08), per throw:**

| Feature | Spec definition | Proposed calculation | Inputs |
|---|---|---|---|
| x_pronation | Peak forearm pronation velocity just before release | Peak of the forearm angular velocity about the forearm's long axis, in a short window before `t_release` | Flexor `g`, sensor-to-segment alignment (OP-3) |
| x_emg_delta | Gap in ms between the peaks of the flexor-pronator mass and the triceps | t(peak envelope, flexor) − t(peak envelope, triceps) | An envelope the app computes from the 2 kHz EMG codes. Placement (4.2). |
| x_slot_ratio | "Ratio of elbow joint adjustment velocity to shoulder rotational displacement via ToF" | **Unclear.** The ToF measures the distance between upper-arm and forearm pucks, which gives the elbow, not the shoulder. Needs clarifying (R-11). | ToF events, upper-arm orientation |
| x_accel_ratio | a_linear / ω_max | D-15 / `raw_max_omega` | 4.5, 4.6 |

The engine spec calls these "four scale-invariant ratios", so that the classifier still works at 75% effort. Only the last one is a ratio. Pronation velocity in particular grows with effort, so classification may be less reliable at low effort than claimed (R-14).

**Calibration, per pitch type p:**
1. Collect P-10 labelled throws, spread over normal sessions.
2. **z-score** the four features with the athlete's means and SDs. Cluster centre μ_p = mean of their x. Covariance **(research):** OAS shrinkage, keeping the paper's 2/p terms (scikit-learn drops them, which is wrong at p = 4), with a floor of 0.1 below n = 15. Then partial pooling across the athlete's pitch types: Σ_p = (1 − η)·Σ_p,shrunk + η·Σ_pooled, with η ≈ 0.5 below n = 15 tapering to ≈ 0.2 by n = 30. Unregularised per-type covariance at n = 10 was overconfident in simulation (R-10).
3. Store as the pitch baseline (D-130), with its version.

**Classifying a throw:**
1. D_M(x, p) = √((x − μ_p)ᵀ Σ_p⁻¹ (x − μ_p)) for every pitch type in the repertoire (D-09).
2. Turn the distances into probabilities with a softmax over the **Gaussian log-likelihood** ℓ_p = −½D_M² − ½ log|Σ_p| + log π_p (uniform priors at first). **(research)** A softmax of −D_M (one reading of the spec) was badly underconfident in simulation: only 27–49% of throws reached 0.85, though 94–99% of those were right. With log-likelihoods about 70% pass at 94–97% precision. Later, fit one temperature on held-out sessions.
3. If max P ≥ 0.85 (P-8) **and** D² ≤ the cluster's cutoff (P-9), assign that type. **Don't use χ² cutoffs at these sample sizes (research):** with μ and Σ estimated from n throws, a new throw's D² follows [p(n+1)(n−1)/(n(n−p))]·F(p, n−p). At n = 10 a χ² 95% cutoff wrongly rejects about 33% of genuine throws. Log "ambiguous" (probability gate) and "novel" (distance gate) rejections separately, with confidence = max P (D-19). Otherwise assign General Volumetric, with no confidence. The cutoff matters because a softmax only compares the clusters with each other, so a throw far from all of them can still get a high probability.
4. Store the label, confidence, model version and any athlete correction (D-136). Never retrain on unconfirmed predictions.

**Validate:** leave-one-session-out cross-validation, never random throw-wise splits (they inflate accuracy). Per-type precision and recall on **new sessions**, not the calibration ones; precision vs coverage across the probability threshold; a leave-one-pitch-type-out novelty test. Also: the share of throws classified automatically, the confusion between types, and stability after the sleeve is taken off and put back on. Start with fastball vs curveball vs changeup.

### 4.12 Consistency (D-25)

**Definition (engine spec §5):** per throw, `S_Consistency = 100 · exp(−λ₂ · D_M(x, p))`, using the distance to the throw's assigned pitch type from 4.11.

**Method:**
1. Per classified throw: compute S_Consistency with P-5. **Alternative (research, preferred):** a percentile score, `100 · (1 − F_ref(D²))`, where F_ref is the small-sample F distribution for the cluster's n (or the athlete's held-out distances). A median throw scores 50 and the best approach 100, and athletes with few calibration throws aren't penalised. The spec's exponential form with a fixed λ makes small-n athletes look less consistent and makes 100 unreachable. **Product decision.**
2. Per pitch type in a session: average over that type's throws. Session Consistency: average over all eligible classified throws (DATA_CONSOLIDATION A24).
3. Flag meaningful deviations and declining consistency, with the features that contributed. Which deviations count as meaningful is not defined (developer spec §06).

**Open points:**
- **General Volumetric throws hide the biggest drifts (R-8).** A throw that drifts beyond the cutoff becomes General Volumetric and is excluded, so Consistency can never be scored on it. To keep drift visible, track the share of throws that fall out of their cluster per session alongside the score.
- **EMG timing is counted twice (R-9).** It is both a Consistency feature (x_emg_delta) and the whole of Execution (4.13). The developer spec warns against counting the same deviation in both.
- The developer spec also lists arm slot, acceleration and angular velocity variability, upper-arm and forearm patterns, and phase timing as Consistency inputs. The engine spec uses only the 4-value feature vector.

### 4.13 Performance (D-01, D-02, D-03)

**Definition (engine spec §5):** per throw, `S_Performance = 0.45 · S_Execution + 0.30 · S_Output + 0.25 · S_Consistency`. The weights are placeholders. The session score is the average over eligible throws (A24), shown with its three components.

**Execution (D-02):** `S_Execution = 100 · (1 − Σᵢ wᵢ · |tᵢ − tᵢ,target| / ΔT_delivery)`
- tᵢ: activation time of muscle i (biceps, flexor-pronator mass, triceps), from the peak of each puck's EMG envelope, relative to the first. Stored as `emg_peaks_ms`, e.g. {biceps: 0, fpm: 8, triceps: 14}.
- tᵢ,target: the athlete's target timing for this pitch type (D-133). The spec doesn't say how it is set or updated.
- ΔT_delivery: the delivery duration. **Not defined.** Foot plant to release would make sense, with the same foot-plant problem as 4.6.
- **Clamp to 0–100.** If the timing errors add up to more than ΔT_delivery, the formula goes negative (R-12).

**Output (D-03):** `S_Output = 100 · exp(−λ₁ · max(0, v_baseline − v)²)`
- v: arm speed (4.5). v_baseline: the athlete's baseline for the same context (pitch type, effort, session type) (D-132).
- Only a drop below the baseline lowers the score. Throwing above the baseline scores 100, not more.
- The developer spec says Output covers "arm speed, acceleration and other output metrics". The formula uses arm speed only.

**Consistency:** from 4.12. A General Volumetric throw has no Consistency, so its Performance can't be computed by the formula. **Proposed:** leave such throws out of Performance (A24).

**Baselines:** the spec requires a "building baseline" state until enough throws exist, and baselines that evolve with training. Neither the minimum count nor the update rule is defined (developer spec §06).

### 4.14 Live CBR (D-04, D-05, D-17)

**Definition (engine spec §6):**
- Stress unit per throw: `SU_k = (ω_k / ω_cal_max)² · (a_k / a_cal_max)²` (D-04). **Every** throw counts: catch play, warm-up and General Volumetric too.
- Session demand: `B_live(n) = Σ_{k≤n} SU_k` over the day's throws (D-05).
- Live CBR: `S_CBR_Live = max(0, S_CBR_Daily − γ · B_live(n))` (D-17).

**Inputs:** `raw_max_omega` (4.5) and a (4.6) per throw. The athlete's calibrated maxima ω_cal_max and a_cal_max (D-134). Daily CBR (D-06). γ (P-6).

**Method:** at the start of the day, record S_CBR_Daily as the starting value. After each throw, add SU_k and recompute Live CBR. Show both values (developer spec: "preserves the starting-of-day score").

**Daily CBR, proposed structure (research, not validated):** a Banister-style decaying fatigue state, with any long-term term used only to scale capacity, never as a ratio (so it stays clear of ACWR).
1. Daily dose D(s) = Σ SU on day s, every throw counted.
2. Fatigue F(d) = Σ_{s<d} D(s)·e^{−(d−s)/τF}, τF per P-12. Arm-recovery studies (shoulder range of motion, strength, HRV) point to 24–72 h.
3. Tolerance T(d) = an uncoupled decayed sum of D with τ per P-13, lagged to exclude the fatigue window.
4. Capacity C(d) = C_min + (C_max − C_min)·(1 − e^{−T/T_ref}). New users get population defaults by age.
5. Charge = clamp(100·(1 − F/C), 0, 100).
6. Optional inputs as multipliers: Daily CBR = clamp(Charge · Π mᵢ, 0, 100), mᵢ = 1 + βᵢ·clip(zᵢ, −2, 2) against the athlete's own 14–28 day baseline, active after 7 baseline days, **mᵢ = 1 when missing** so missing data never lowers the score. Arm soreness (Arm check, D-120) β ≈ 0.10; HRV and sleep β ≈ 0.05 each (no evidence links them to arm recovery).
7. Calibrate τF, the dose exponent and the slow component per age band so that an outing at the top of each MLB Pitch Smart band returns Charge above the "ready" line after exactly that band's rest days (e.g. ages 19–22: 1–30 pitches → 0 days … 106+ → 5 days).

**Open points:**
- The structure above is a proposal. No peer-reviewed non-ratio readiness model for throwers exists to validate it against. The constants, the "ready" line and the modifier weights are product decisions until WYNG has its own data (DATA_CONSOLIDATION A16).
- A throw harder than the calibrated maximum gives SU > 1. Decide whether a new maximum updates ω_cal_max and a_cal_max.
- The ACWR formula must not be used for CBR (developer spec §02).

## 5. Risks

| # | Risk | Effect | Check |
|---|---|---|---|
| R-1 | **Gyro saturation: range confirmed at ±4000 dps** (ICD §2). **(research)** Reported forearm and upper-arm rates in pitching are 6,000–12,000 °/s from IMU studies (motion capture reports lower, ≈ 2,400–4,700 °/s, partly from filtering), so clipping is likely for competitive adults. | Arm speed, acceleration and stress units capped; orientation behind arm slot and angular acceleration behind arm load corrupted, all near release. | Measure the clip rate on real throws (`imu_clip_g`, or samples at full scale). Withhold metrics computed across a clipped span. Untested idea: estimate ω above the limit from the centripetal term in `h` (|a| ≈ ω²r). |
| R-2 | **Accelerometer range: resolved.** A ±320 g high-g channel `h` is recorded alongside the ±16 g channel `a` (ICD §2), above the 126–138 g reported in pitching. | Use `h` for peaks and `a` in quiet periods. | Confirm with real throws. |
| R-3 | **No absolute heading.** No magnetometer until firmware v0.9 (ICD §16). | Comparing orientations across pucks needs heading alignment (4.3). | Validate heading alignment early. |
| R-4 | **Puck position on the segment varies** between sleeves and athletes. | Arm radius (4.5) and the lever arms (4.3, 4.8) shift. | Measure the variation across sleeve sizes. |
| R-5 | **Wrong conversion constant in the engine spec.** C ≈ 0.0568 instead of ≈ 0.0390. | Arm speed is 46% too high. Every score built on it (Output, CBR stress units, accel ratio) shifts with it. | Use 0.0390 and correct the spec. |
| R-6 | **Foot plant is not observable** from arm-worn sensors. **(research)** No published method exists; foot-contact methods use foot or pelvis sensors. | Acceleration (4.6) and ΔT_delivery (4.13) depend on an estimate (P-11, ±40 ms). | Validate against synchronised high-speed video. |
| R-7 | **The engine spec differentiates a ToF time series twice.** Rev 0.4 records ToF at **200 Hz** on every trigger, so a series now exists, but about 6 samples between max external rotation and release makes a second derivative very noisy. | Elbow angular velocity and acceleration from ToF alone will be coarse. | Use orientation for the fast part, corrected by ToF. Test on real recordings. |
| R-8 | **General Volumetric hides drift.** | Consistency can't see the largest deviations (4.12). | Track the share of throws falling out of their cluster. |
| R-9 | **EMG timing is counted in both Execution and Consistency.** | The same deviation is penalised twice. | Decide which score owns it. |
| R-10 | **Covariance from 10 throws** in 4 dimensions is unstable. | Unreliable distances, probabilities and Consistency early on. | Regularise (shrinkage), and require more throws before scoring Consistency. |
| R-11 | **x_slot_ratio needs shoulder rotation**, which no puck measures directly. | The feature may not be computable as defined. | Clarify the definition. |
| R-12 | **S_Execution can go negative.** | Out-of-range scores. | Clamp to 0–100. |
| R-13 | **The engine spec's data model is incomplete.** It names four core entities but defines two (athlete profile, throw event). | Sessions, CBR state and pitch baselines have to be taken from the developer spec's field list. | Ask for the missing tables. |
| R-15 | **Throw detection is unspecified and now the app's job.** The firmware has no throw concept. | Missed or false throws distort counts, workload and CBR. Catch-play throws are the hardest. | Validate against video counts (4.1). |
| R-16 | **Data volume.** About 47 kB/s per puck, or 237 MB for a 28-minute session, and about 2.5 h to offload at the 26 kB/s measured to a PC (DATA_CONSOLIDATION 1.9.5). Recording is capped at about 47 min. | Scores may not be available the same day, and phone storage and processing are affected. | Measure on iPhone. Consider an on-puck throw index (firmware question Q24). |
| R-17 | **Speed of sound.** The firmware's range uses a fixed value (0.343 mm/µs in ICD §6.6). About 0.17% per °C, so a 15 °C difference shifts a 200 mm range by about 5 mm. The only temperature reading is the chip (`die_temp_q2`). | Elbow angle from ToF shifts with temperature. | Test how well the chip temperature tracks air temperature. |
| R-14 | **Features aren't all scale-invariant**, despite the spec's claim. | Classification may degrade at low effort (warm-up, catch play). | Test accuracy at 50–75% effort. |

## 6. Open points

| # | Open point |
|---|---|
| OP-1 | **Resolved.** Scales are fixed and carried in `imu_info` and the SESSION record (ICD §8.1, §10.7). |
| OP-2 | **Resolved.** Raw `a`, `g`, `h` only. Orientation is fused in the app. On-puck quaternions are "planned" with no version given (ICD §15). |
| OP-3 | Sensor-to-segment alignment and heading alignment: by a calibration pose, or estimated from throw data. Entirely the app's responsibility now, since the pucks store no calibration. The calibration routine has no screen yet (DATA_CONSOLIDATION 2.4). |
| OP-4 | Confirm AS-3 to AS-5. AS-4 (arm load) matters most. AS-1 and AS-2 are superseded by the specs. |
| OP-5 | Reference data for validation. **(research)** One combined session with optical motion capture and ≥ 1000 fps video can settle release timing, the foot-plant offset, arm-slot bias and torque agreement together. Plus labelled pitch sessions for classification. |
| OP-6 | Values for the undefined parameters P-4 to P-7 and P-9. |
| OP-7 | Baseline rules: the minimum throws before each score is shown, and how baselines update with training (developer spec §06). |
| OP-8 | How effort is captured, for "like with like" comparison (DATA_CONSOLIDATION A20). |
