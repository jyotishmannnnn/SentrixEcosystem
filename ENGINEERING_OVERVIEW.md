# Sentrix — Engineering Overview

> An internal technical architecture review of the Sentrix ecosystem, written for
> software engineers, robotics engineers, and technical reviewers. This document
> explains *how the system is engineered* — the scientific, mathematical, and
> systems foundations behind the architecture — rather than what features it
> ships. For repository maps, workflows, and per-repo detail, see
> [`ECOSYSTEM.md`](ECOSYSTEM.md), [`WORKFLOWS.md`](WORKFLOWS.md), and the
> repository READMEs (Section 7).
>
> Every statement below is grounded in the repository contents as of this
> writing. Where a capability is a contract, a seam, or a roadmap item rather
> than running code, it is explicitly labeled **Planned** or **Future**.

---

# 1. Introduction

## 1.1 Engineering goals

Sentrix turns visuotactile glove demonstrations into ML-ready datasets for
robotics and physical-AI research. The engineering problem underneath that
sentence is narrower and harder than "record sensor data":

1. **Ground the physical signal losslessly.** Contact, force proxies, and
   timing must survive the trip from sensor silicon to training file without
   silent modification. Raw values are preserved as measured; nothing is
   filtered, zeroed, calibrated, or interpolated at the edge.
2. **Quantify uncertainty instead of hiding it.** Every aligned sample carries
   an explicit, decomposed confidence (source, clock, interpolation). Gaps are
   represented as gaps — `NaN` with confidence zero — never fabricated by
   interpolation across a dropout.
3. **Make time a first-class measured quantity.** Multiple devices with
   independent, drifting clocks must be reconciled onto one reference timeline
   with estimated (not assumed) clock models and a documented accuracy budget.
4. **Make every dataset provable.** A delivered dataset must trace
   cryptographically back to the exact source episodes and the exact hardware
   revision (topology descriptor) it came from.
5. **Decouple the pipeline from the hardware revision.** Sensor count and
   placement are data (a descriptor), not code. A new glove revision is a new
   JSON file.

## 1.2 System philosophy

Three commitments shape everything:

- **Contract-first.** A standalone, zero-dependency package
  (`sentrix_contracts`) owns the wire codec, the raw-frame schema, the topology
  descriptor, and the column naming convention. Every other repository imports
  it; none redefines it. The wire layout, schema version, descriptor hash, and
  column convention are frozen and pinned by byte-level tests.
- **Measurement before interpretation.** The capture layer records device-local
  timestamps and raw absolute field values, plus *evidence* for later
  reconciliation (strobe edges, health telemetry). Clock reconciliation,
  calibration application, and feature derivation all happen downstream, where
  they are reproducible and re-runnable. Nothing irreversible happens at the
  edge.
- **Honest fidelity accounting.** The simulator refuses to invent unknown
  physical constants: parameters are tiered `KNOWN | ESTIMATED | UNKNOWN`, and
  reading an `UNKNOWN` parameter raises unless placeholders are explicitly
  allowed — in which case the output is stamped with a degraded
  `physics_fidelity` label. The same honesty extends to datasets
  (release-gate verdicts) and documentation (simulated vs. real vs. unverified
  claims are separated in the dataset release notes).

## 1.3 Why the architecture is modular

The pipeline is factored along *invariant boundaries*, not convenience
boundaries:

- **Producer / synchronizer split.** Synthetic (SentrixSim) and real-hardware
  (SentrixCapture) producers emit the *same* self-describing raw artifact.
  Everything downstream is therefore testable end-to-end today, on synthetic
  data, with zero changes required when real hardware arrives. This is the
  standard sim-first bring-up strategy for robotics stacks, enforced here at
  the byte level rather than by convention.
- **Synchronization / materialization split.** SentrixSync computes *indices
  and confidence* (which source sample maps to which grid point, and how
  trustworthy that mapping is); SentrixDataEngine *applies* them. The
  synchronizer never touches payloads; the materializer never re-derives a
  join. This makes the alignment decision auditable and the dataset build a
  pure, deterministic function of `(raw artifacts, SyncResult)`.
- **One canonical representation.** All export formats (LeRobot, RLDS, HDF5,
  MCAP, Parquet) are pure projections of a single canonical Silver table, so
  format-level disagreements are impossible by construction.
- **Visualization beside, not inside, the pipeline.** SentrixViz consumes the
  same artifacts read-only and never persists display-space transforms.

## 1.4 How the repositories work together

Six repositories share one dependency direction:

```
SentrixContracts (wire codec · frame schema · topology descriptor · columns)
        │  imported by everyone; owns nothing else
        ▼
SentrixSim ──────┐                 (synthetic producer)
SentrixCapture ──┤ → raw.parquet   (real-hardware producer, byte-compatible)
                 ▼
SentrixSync      → SyncResult + Session   (clock models, grid, join indices, confidence)
                 ▼
SentrixDataEngine → Silver → Gold exports → validation → signed, provenance-closed package
                 
SentrixViz       ← reads raw or Silver artifacts at any point (rendering only)
```

Sync depends on producer *output artifacts*, not producer code; DataEngine
depends on Sync *types*, read-only. Root-level tooling (`tools/orchestrator/`,
`tests/integration/`) drives the whole chain in one process and asserts
cross-repo invariants no single repository can check.

---

# 2. System Architecture

## 2.1 Stage responsibilities and why each exists

**Contracts (SentrixContracts).** Multi-repository systems fail at their seams.
The contract layer removes the seams' degrees of freedom: one frozen on-wire
byte layout (`MAGIC | VER | LEN | payload | CRC32`), one raw-frame schema
(`SCHEMA_VERSION = 1`, mirrored in C and Python), one topology descriptor
format with a canonical SHA-256 identity, one at-rest column convention
(`mag.<sensor_id>.bx_uT`, `dyn.<sensor_id>.ax_g`). The package has zero runtime
dependencies precisely because everything imports it — the keystone must not
constrain anyone's environment. Cross-language drift is prevented by an
*executable specification*: a test reconstructs expected bytes independently
from the C struct layout and asserts the shared codec produces them exactly.

**Producers (SentrixSim / SentrixCapture).** A dataset pipeline needs input
long before hardware is reliable, and hardware needs a reference implementation
to be validated against. Sim is a descriptor-driven forward model of the glove
(Section 3.8) that emits physically-shaped, sensor-realistic episodes;
Capture is the firmware + host path that will produce the same artifact from
silicon. Because the two artifacts are byte-compatible — verified by replaying
Sim episodes through the *real* wire codec and host decode path — the producers
are interchangeable from the pipeline's point of view.

**Synchronization (SentrixSync).** N devices means N independent crystal
oscillators, each with offset and drift relative to any reference. Ignoring
this (assuming a shared clock, or trusting wall-clock stamps) corrupts
cross-modal timing at exactly the timescales dexterous-manipulation research
cares about (millisecond contact events). Sync estimates per-device affine
clock models from shared observable events, reconciles them over a device
graph, builds a uniform reference grid, and computes per-stream join indices
with per-sample confidence. It is deliberately *modality-neutral*: it consumes
only abstract timestamped events and never inspects payloads or topology.

**Processing (SentrixDataEngine).** Materialization is separated from
synchronization so that dataset builds are deterministic, offline, and
re-runnable: resolve payload references, apply the precomputed join, produce
one canonical Silver table, project to Gold formats, validate against explicit
thresholds, and package with signed provenance. The engine's release gate is
*ceilinged* by the synchronizer's own verdict — a dataset can never claim
better quality than the synchronization it rests on.

**Visualization (SentrixViz).** Human inspection of tactile fields requires
spatial rendering, which requires geometry — but geometry must come from the
descriptor, never from constants baked into rendering code. Viz derives every
spatial fact (2D projection, interpolation length scales, cluster shapes) from
sensor positions at runtime, and keeps a strict wall between *scientific space*
(descriptor coordinates, PCA projection) and *display space* (anatomical
warping, color mapping, normalization), with display-space signals never
persisted.

## 2.2 Dependency relationships

The rules are few and absolute (see `ECOSYSTEM.md` → Repository Ownership
Rules):

- Data flows strictly left→right; no repository imports a downstream one.
- Sync and DataEngine never import a producer; they see files and types.
- DataEngine never mutates a `SyncResult`; its only write-back is appending an
  `ExportRecord` to the `Session` manifest.
- Payloads move by reference (`payload_ref` URIs), never inline through the
  synchronizer.
- Topology flows as *opaque provenance* (`topology_ref`, `topology_hash`)
  through Sync — carried, never consumed — and is only ever interpreted by the
  producers, the opt-in derived exporter, and Viz.

The practical consequence: each stage can be re-implemented, scaled, or tested
in isolation against artifacts, and the integration surface is a small set of
frozen schemas rather than a web of code dependencies.

---

# 3. Mathematical Foundations

This section covers the mathematics actually implemented in the repositories,
plus the mathematics the architecture explicitly reserves space for. Status
labels: **[Implemented]**, **[Planned]** (contract or seam exists, code does
not), **[Future]** (roadmap only).

## 3.1 Discrete-time signals and sampling **[Implemented]**

The system is natively **multi-rate**: tactile field frames at 400 Hz, inertial
dynamics at 1.6 kHz, thermal at ≤50 Hz, health telemetry at 1–10 Hz. No stage
pretends otherwise:

- Producers emit each modality as a separately timestamped stream at its native
  cadence. The simulator generates on a 1600 Hz master grid and *decimates* to
  each modality's rate (field ratio 4, i.e. every 4th master sample), which is
  exact rather than approximate resampling.
- The synchronizer resamples onto a uniform **reference grid** whose rate is
  configurable and conventionally set to the highest joined native rate
  (default 1600 Hz), never anchored to an assumed camera frame rate.
- For future anchor streams at non-commensurate rates (e.g. 30 fps video
  against 1600 Hz tactile), a fixed-ratio **sub-frame bucketing** scheme is
  implemented and tested: `R = ceil(grid_rate / anchor_fps)` samples per
  anchor frame, with genuinely-empty buckets marked invalid rather than padded.

*Why it matters:* aligning multi-rate streams is the core data-engineering
problem of multimodal robotics datasets; doing it with explicit grids and
validity masks (rather than implicit library resampling) is what makes the
downstream confidence accounting possible.

*Trade-offs:* a uniform grid at the highest native rate maximizes information
retention at the cost of storage (slow streams are upsampled with
validity/confidence marking). The common alternative — downsampling everything
to the slowest stream — destroys the high-rate dynamics content and was
rejected.

## 3.2 Timestamp mathematics and numerical precision **[Implemented]**

Timestamps are **64-bit integer microseconds, everywhere** (an explicit
contract decision; float timestamps are rejected at the type level in Sync).
The rationale is documented in the Sync contract: float seconds introduce unit
ambiguity and precision loss (a float64 second-count near 10⁹ has ~µs-scale ULP
already; float32 is hopeless), while int64 µs gives exact arithmetic, exact
equality, and ~292,000 years of range. Firmware uses the same convention: a
free-running 64-bit µs counter is the device master clock, so wraparound
handling is unnecessary by construction.

The numerical-precision policy across the stack is deliberate and layered:

- **Time:** int64 µs at rest and on the wire; estimation math converts to
  float64 internally and rounds back to int64 (`round`, not truncation) at
  every boundary. Grid construction uses a float step (`1e6 / rate`) with
  per-point rounding so non-integer-period rates remain unbiased over long
  spans, and monotonicity is enforced (strictly increasing, ≥1 µs separation).
- **Payloads:** float32 end-to-end (matching sensor-realistic precision and
  halving storage), with promotion to float64 inside derived-feature math and
  demotion back to float32 on output.
- **Quantization:** the simulator models sensor LSB quantization explicitly
  (`round(value / lsb) · lsb` with datasheet step sizes: 0.1 µT for the
  magnetometer, 14-bit codes over ±16 g for the accelerometer, 0.0625 °C for
  temperature), so downstream consumers see realistically discretized values.

*Common alternative:* nanosecond integers (ROS 2 / MCAP convention). The
contract fixes µs for now and documents ns as a future contract revision; MCAP
export multiplies by 1000 at the boundary.

## 3.3 Clock models and synchronization mathematics **[Implemented]**

Each device clock is modeled as an **affine map to the reference timeline**:

```
t_ref = α · t_local + β        (α ≈ 1 + skew_ppm·10⁻⁶,  β = offset in µs)
```

with optional piecewise-affine segments for nonlinear drift. Estimators, in
increasing robustness (all in `SentrixSync/src/sentrixsync/clock/estimate.py`):

- **Offset-only least squares** — `β = mean(t_ref − t_local)`, `α = 1`; the
  degenerate fallback.
- **Ordinary least squares (OLS) affine** — closed-form slope
  `α = Σ(x−x̄)(y−ȳ) / Σ(x−x̄)²`, with a single-pass 3σ residual rejection and
  refit.
- **Total least squares (TLS, orthogonal regression)** — the production
  default. Closed form on centered second moments:
  `α = (s_yy − s_xx + √((s_yy − s_xx)² + 4·s_xy²)) / (2·s_xy)`.
  TLS is chosen over OLS because *both* clocks in a pairwise fit are noisy
  observations — OLS's assumption of an exact regressor is wrong here, and the
  errors-in-variables formulation removes the resulting skew bias.
- **RANSAC** (opt-in "robust" mode) — 200 seeded iterations of minimal
  two-point models, inlier threshold 1000 µs, maximal-consensus set refit with
  TLS. Used when event association may contain gross outliers
  (mis-associated events), which no σ-based rejection survives.

*Alternatives and why not (yet):* Kalman/state-space clock filters (as in
PTP/NTP servo loops) handle online, continuously-updating clocks; Sentrix
synchronization is currently *batch* over a recorded session, where closed-form
robust regression is simpler, deterministic, and easier to audit. Piecewise
affine covers slow nonlinear drift without introducing filter tuning.

**Event basis.** Clock fits need common observables. The implemented detectors
(tactile tap, visual flash) are deliberately simple deterministic peak finders
(threshold crossing → one peak per supra-threshold region at the argmax),
registered through a plugin interface. Cross-device association pre-aligns with
coarse wall-clock offsets, then greedily clusters detections within a tolerance
window, requiring a minimum number of observers per event. The hardware path
adds a stronger basis: **strobe edges** — the firmware timestamps every emitted
camera-genlock edge against its master clock and streams them as sync frames;
the host records them to `sync_evidence.parquet` for Sync to consume.

## 3.4 Graph-based multi-device reconciliation **[Implemented]**

With N devices, pairwise fits form a graph: devices are nodes; a pair
co-observing enough shared events contributes an edge carrying a fitted affine
transform and a confidence. Reconciliation onto one reference is a shortest-path
problem:

- Affine maps **compose** along a path:
  `(α₂, β₂) ∘ (α₁, β₁) = (α₂α₁, α₂β₁ + β₂)`, and invert as
  `(1/α, −β/α)` — so any connected device can be mapped to the reference
  through intermediate devices.
- Path selection runs **Dijkstra from the reference node minimizing cumulative
  `−log(confidence)`** — equivalently, maximizing the product of edge
  confidences along the path. Cost is *reliability-weighted*, not
  residual-weighted: a two-point edge with a perfect (overfit) residual cannot
  outrank a well-supported edge, because the confidence term folds in sample
  count as well as residual.
- **Error propagation across hops** is multiplicative in confidence and
  accumulated in the path cost; the synthetic accuracy budget holds transitive
  (≥2-hop) devices to explicitly looser bounds (≤1500 µs RMSE) than
  direct-edge devices (≤500 µs).
- Unreachable devices degrade gracefully: identity model, clock confidence 0,
  reported in diagnostics — never an error, never fabricated alignment.

*Alternative:* a global simultaneous least-squares over all edges (as in
sensor-network time sync literature) would use all constraints at once rather
than a spanning tree; the tree formulation was chosen for interpretability
(every device has one auditable path and one confidence product) and is
sufficient at current device counts.

## 3.5 Interpolation and the no-fabrication invariant **[Implemented]**

The as-of join maps each grid point to source samples via
`searchsorted` (last sample ≤ grid point), with two kernels:

- **Hold (zero-order hold):** value = previous sample; valid iff the gap to it
  is within tolerance. Used for streams where holding is physically meaningful.
- **Continuous (linear bracket):** stores previous/next indices plus a clipped
  interpolation weight `w = (t_grid − t_prev)/(t_next − t_prev)`; the
  materializer computes `v = a·(1−w) + b·w`. Validity requires the nearest
  bracket sample within tolerance.

The **rejection tolerance** (default ≈3 grid periods) is the single knob that
separates interpolation from fabrication: any grid point whose nearest source
sample is farther than the tolerance is marked invalid, its value stays `NaN`,
and *all* its confidence components are zeroed. This invariant is enforced
three times independently — as a property check inside Sync, as a critical
validation check in DataEngine (any violation → `BLOCKED`), and in Viz (invalid
sensors drawn hollow, never interpolated).

*Alternatives:* spline or sinc reconstruction would give smoother resampling
but would (a) smear dropouts across neighbors — precisely the failure mode the
system is designed to prevent — and (b) impose a signal model on data whose
value is that it is raw. Linear-within-tolerance is the most conservative
choice that still supports a uniform grid.

## 3.6 Statistics, probability, and estimation **[Implemented]**

Beyond the regression machinery above:

- **Robust statistics:** 3σ residual rejection (single-pass, refit),
  RANSAC consensus, minimum-support gating on graph edges (edges with too few
  shared events are never formed).
- **Confidence as a structured estimate.** Every aligned sample carries three
  components, kept separate end-to-end:
  - *source* — producer-asserted per-sample validity/confidence;
  - *clock* — the fitted model's confidence,
    `(1 − 1/(n+1)) · exp(−RMS/1000 µs)` (support × residual terms), optionally
    decayed as `exp(−d/τ)` with distance `d` to the nearest sync event —
    an explicit extrapolation-uncertainty model;
  - *interpolation* — gap-decay `1 − gap/tolerance`, clipped to [0, 1].
  The scalar `source · clock · interpolation` exists **only at export**; the
  components are the source of truth. The code is explicit that these are
  *estimated heuristics, not calibrated probabilities* — an honest engineering
  statement rather than a probabilistic claim.
- **Generative loss models for validation:** Bernoulli i.i.d. dropout and a
  two-state Markov (Gilbert) burst-loss model, Gaussian timestamp jitter, and
  µs quantization are implemented as forward corruption models used to stress
  the synchronizer against known ground truth, giving measured (not assumed)
  accuracy budgets.
- **Simulator degradation statistics (Hard Mode):** per-sensor dropout spans,
  clipped-Gaussian timestamp jitter (σ = 70 µs, max 250 µs), per-session
  calibration drift (gain `1 + N(0, 0.06)`, offset `N(0, 1.5 µT)` per channel),
  style variation (time-warping `u^k`, amplitude scaling, tremor sinusoids),
  and hard-negative episodes (non-contact motion plus common-mode field
  wander) — each parameterized and confidence-tagged in configuration rather
  than hard-coded.

**[Future research]** — calibrated confidence (mapping the heuristic components
to empirical probabilities) is a natural extension the three-component design
leaves room for, but no calibration of the confidence itself exists today.

## 3.7 Linear algebra, geometry, and coordinate transformations **[Implemented]**

- **Frames and rotations.** The descriptor defines a device frame (metres;
  for Mark 2: origin at the wrist hub, anatomical axes) and per-sensor
  positions `position_m` plus a 3×3 `local_frame` rotation (sensor→device).
  The simulator applies these rotations to field vectors via a batched
  `einsum`; for the flat Mark 2 board all mounts are identity *by hardware
  design*, but the machinery is exercised so curved future layouts are a data
  change.
- **Dipole field model (Sim).** Each fingertip magnet is a point dipole; the
  field shape is the classical
  `g(r, m̂) = (3(m̂·r̂)r̂ − m̂)/|r|³`, and the sensed perturbation is the
  *difference* between the dipole displaced by contact deformation and at rest.
  Absolute magnitude is intentionally a presentation-scaled quantity (the
  remanence `Br` is an UNKNOWN-tier parameter); the ~1/r³ spatial structure and
  cross-sensor coupling are the physically meaningful content.
- **Cluster feature math (shared contract).** The derived features are small,
  NaN-aware linear-algebra reductions defined *once* in
  `sentrix_contracts.derived` and imported by both the batch exporter and the
  visualizer so they cannot drift: common-mode response `nanmean(|ΔB|)`
  (normal-force proxy), mean lateral field vector and its L2 norm (shear proxy
  and magnitude), response sum (activation), and the **response-weighted
  centroid** `Σ rₖ·posₖ / Σ rₖ` with NaN members contributing zero weight —
  a convex combination of member positions, verified by test to lie within the
  cluster's extent.
- **Data-driven 2D projection (Viz).** Rendering never picks an axis pair:
  positions are centered and decomposed with SVD, and the two dominant
  right-singular vectors form the projection basis (PCA). Orientation is then
  canonicalized with one rigid rotation chosen via eigenvalue analysis of
  cluster covariance (the broadest cluster is palm-like; fingers rotated to
  +y). This works unchanged for a line of sensors, a plane, or a patch.
- **Scattered-data interpolation for display.** Heatmaps use inverse-distance
  weighting, `w = 1/d^p` (p = 2), with a support mask whose radius derives from
  the **median nearest-neighbor spacing** of the layout — the natural length
  scale of the topology, not a constant. IDW is chosen over Delaunay/RBF
  precisely because it survives every geometry (collinear, sparse, dense)
  without matrix solves or degeneracy branches.
- **Thin-plate-spline anatomical warp (Viz, display-only).** For human-readable
  hand rendering, sensor positions are warped onto a canonical hand template
  with the minimum-bending-energy interpolant (kernel `U(r) = r² log r²`,
  block-system solve with a scale-invariant regularizer λ = 10⁻⁶), with an
  inverse TPS solved for consistency checking (`inverse_rms` reported).
  Scientific coordinates are never modified; the warp exists only in display
  space.
- Supporting computational geometry: Euclidean neighbor derivation within a
  descriptor-declared radius, Andrew monotone-chain convex hulls and even-odd
  point-in-polygon tests for procedural hand silhouettes — all
  position-derived, no assets, no constants.

## 3.8 Calibration mathematics **[Planned]**

Calibration is currently a *contract*, not code. The bundled
`calibration_bundle` JSON-Schema defines per-serial, per-sensor parameters:
per-channel baseline (no-contact field), gain and offset arrays, an optional
**3×3 soft-iron correction matrix** for magnetometers, per-channel temperature
coefficients, refined (metrology-measured) sensor positions overriding nominal
descriptor positions, and noise floors. Two design decisions are already
binding:

- **Calibration is never applied at the edge.** Raw absolute field is recorded;
  the bundle travels alongside so any downstream consumer can apply (and
  re-apply, and audit) the correction reproducibly. `calib_version` is stamped
  into every raw frame header so data and calibration state are permanently
  associated.
- **Type-level geometry and unit-level calibration are separated** — the
  descriptor is shared across all units of a hardware revision; the calibration
  bundle is per-serial.

The implied correction model (baseline subtraction, gain/offset, soft-iron
multiplication, linear temperature compensation) is standard magnetometer
calibration practice; the arithmetic and the estimation procedure that produces
the parameters (e.g. ellipsoid fitting for soft-iron) are **not implemented
anywhere yet**. The simulator's Hard Mode applies *generative* miscalibration
(session gain/offset drift) to create data that will eventually exercise this
path.

## 3.9 Cryptographic and identity mathematics **[Implemented]**

Provenance rests on standard constructions used deliberately:

- **Canonical hashing for identity.** A descriptor's identity is
  `sha256` over its canonicalized JSON (sorted keys, compact separators, hash
  field excluded) — so semantically identical descriptors hash identically and
  any change is a new hardware revision by definition.
- **Merkle root over file hashes** (pairwise SHA-256 with odd-leaf
  duplication), fed sorted digests so the root is order-independent; the root
  is signed with **Ed25519**. A non-cryptographic HMAC fallback exists but is
  flagged `signed = false`, which the release gate treats as a hard failure.
- **Order-independent content hash** (hash of sorted per-file digests) as the
  dataset's reproducibility identity: two builds from the same inputs must —
  and, per the integration tests, do — produce equal content hashes.

---

# 4. Signal Processing & Data Engineering

## 4.1 Acquisition and timestamping at source

The firmware architecture (SentrixCapture) is built around one principle:
**timestamp at the measurement latch, not at read-out.** A single I3C broadcast
command latches a synchronized measurement across all magnetometers (target
intra-array skew < 50 µs); `t_capture_us` is sampled from the free-running
64-bit µs counter at that trigger edge, and the subsequent DMA read-back —
whose timing is load-dependent — cannot contaminate it. High-rate inertial
samples ride hardware FIFOs with per-sample `dt_us` offsets from the batch
timestamp. This is the standard discipline for low-jitter multi-sensor
acquisition, and it is what makes downstream clock estimation meaningful: the
estimator sees measurement times, not transport times.

Sensor transduction in the simulator mirrors datasheet reality — white noise
at datasheet densities (magnetometer 190/450 nT RMS per axis; accelerometer
90 µg/√Hz scaled by bandwidth), saturation clipping at ±2000 µT / ±16 g, LSB
quantization, and √N averaging — so pipeline and model code developed against
synthetic data meet realistic noise floors and discretization.

## 4.2 Synchronization, clock drift, and latency

Clock drift is modeled, measured, and budgeted rather than assumed away
(Sections 3.3–3.4). Skew appears as a non-unit α (ppm-scale), offset as β, and
the accuracy budget is verified against synthetic ground truth with forward
clock models plus corruption (jitter, dropout, burst loss). Latency in the
capture path is handled by buffering with *accounted* overflow: firmware
detects a full USB FIFO and counts an overrun; the host's frame bus is a
bounded ring whose overflows increment a counter; monotonic `frame_id`s let the
host detect drops as sequence gaps. All of it surfaces in **health telemetry**
(die temperature, supply voltage, USB over/underruns, buffer high-water,
per-bus error counts) that feeds capture-time QA gating and, ultimately, the
`source` confidence component. Backpressure is thus an observable, not a silent
data-quality hazard.

## 4.3 Resampling and dataset alignment

Alignment = clock correction (affine map, rounded to int64 µs) → uniform
reference grid → as-of join with tolerance → materialization applying the join
indices (Sections 3.1, 3.5). The materializer's output preserves the multi-rate
truth: upsampled slow streams carry validity masks and interpolation
confidence, so a consumer can always recover "which samples are real."

## 4.4 Filtering, smoothing, denoising — deliberately absent from the pipeline

No filtering, smoothing, or denoising is applied anywhere between sensor and
dataset. This is a design position, not an omission: the product is *raw,
provenance-closed measurement*; any filter is an irreversible modeling choice
that belongs to the consumer (or to a future, explicitly-versioned derived
layer). The only smoothing-like machinery in the stack is in visualization —
e.g. the rolling display normalizer's fast-attack/slow-release EMA auto-gain —
and it is display-only by hard rule, never persisted.

## 4.5 Normalization and calibration

Same posture: signal normalization exists only in display space (a
normalization-policy seam maps physical ranges to color/size/vector channels;
global scans or rolling estimators pick the ranges). Physical calibration is
carried as data and deferred downstream (Section 3.8). The simulator's
parameter registry keeps the normalization between "physically absolute" and
"shape-only" honest via fidelity stamps.

## 4.6 Serialization and transport robustness

One wire format serves firmware and host: packed little-endian structs
(IEEE-754 binary32 floats), framed as `MAGIC | VER | LEN | payload | CRC32`
with CRC32 (IEEE 802.3 polynomial) covering `VER..payload`. The host decoder
resynchronizes byte-wise on CRC failure — a corrupted packet costs exactly
itself, verified by tests that flip payload bytes and split packets across
arbitrary chunk boundaries. Sensor identity on the wire is a compact u16 index
into descriptor order, mapped back to stable string `sensor_id`s at decode via
a `WireMap`. At rest, everything is Parquet (zstd) with self-describing
key-value metadata (sensor-id lists, descriptor hash, timestamp column), plus
optional MCAP mirrors with schema-tagged channels. Payload columns are
sensor-id-keyed by the shared convention, making artifacts count-agnostic and
order-explicit.

## 4.7 Dataset consistency and validation

Consistency is enforced by explicit, threshold-based checks rather than
convention: schema checks (shape/length coherence), timeline checks (strict
grid monotonicity, bounded step `max ≤ 2·min`, no fabricated gaps), metadata
checks, and confidence checks (unit interval, zero-at-gaps). The release gate
combines missing-frame percentage, sync residual, and mean confidence against
three graded threshold bands (certified 0.2% / 500 µs / 0.95; release 1.0% /
2 ms / 0.90; hard-fail 3% / 5 ms / 0.85), requires signed lineage, treats the
no-fabrication and shape checks as unconditional blockers, and is clamped by
the synchronizer's own gate verdict. The four-level verdict
(`CERTIFIED | RELEASE | NEEDS_REVIEW | BLOCKED`) is a quality *measurement*, not
a boolean, which lets marginal data exist without being mislabeled.

## 4.8 Deterministic replay and reproducibility

Determinism is engineered end-to-end:

- Every episode's RNG streams derive from a single master seed by fixed
  arithmetic (independent sub-seeds for noise, drift, style, dropout, jitter),
  recorded in the dataset manifest; PCG64 generators throughout.
- RANSAC is seeded; the orchestrator's knobs all have deterministic defaults;
  content hashes and Merkle roots use canonical ordering.
- The device simulator replays recorded episodes **over the real wire codec**
  through the real host decode path — deterministic hardware-in-the-loop
  emulation, byte-faithful by test.
- The integration suite runs the full pipeline twice with identical inputs and
  asserts equal content hashes, and asserts value-faithfulness Sim→Viz to
  float32 precision (≤10⁻³ µT worst case) at coincident grid points.

For robotics datasets, this is the property that matters most: a reported
result can be traced to a bit-reproducible artifact.

---

# 5. Software Architecture

## 5.1 Contract-first modularity

The architecture inverts the usual monorepo temptation: instead of one codebase
with internal modules, six repositories share a frozen, zero-dependency
contract package and communicate through artifacts. What this buys, concretely:

- **Independent evolution.** Producers, synchronizer, materializer, and
  visualizer version independently (Sync even carries its own semantic
  contract version with an explicit compatibility rule: accept 1.x, reject
  ≥2.0).
- **Byte-level interchangeability.** Two producers, one artifact. The
  Sim-vs-Capture equivalence is not an interface promise but a tested byte
  identity, which is the strongest form of the "sim-to-real same-pipeline"
  pattern in robotics data systems.
- **A single point of schema truth.** Column names, frame layouts, and
  descriptor semantics have exactly one owner; everything else derives counts
  and columns at runtime from the descriptor ("nothing here knows Layout B").

## 5.2 Separation of concerns as enforced invariants

The boundaries described in Section 2 are not documentation — they are
mechanized: one-way imports (checked by the ecosystem validator), read-only
type dependencies, immutable `SyncResult` with a single append-only write-back,
payload-by-reference through the synchronizer, opaque topology provenance
through Sync, and the sync-verdict ceiling on the dataset gate. Each invariant
closes a specific failure class (hidden coupling, silent mutation, quality
laundering) common in long-lived robotics pipelines.

## 5.3 Pipeline design: canonical representation and pure projections

The Silver/Gold design is a deliberate application of the
single-source-of-truth pattern to dataset engineering: exactly one canonical
aligned table (int64 µs grid, float32 sensor-id-keyed streams, per-stream
validity and three confidence components), and every export format a pure
function of it. Exporters, payload resolvers, and event detectors are all
registered through small plugin registries, so adding a format, a payload
scheme, or a detection modality is additive. Derived features are the one
place topology-dependent math enters the export path — opt-in, versioned,
formula-registry-documented in the output metadata, and shared with the
visualizer through a single implementation.

## 5.4 Extensibility and scalability

Extensibility is descriptor-shaped: new hardware revision → new descriptor
JSON (new hash, new provenance identity), zero code change — demonstrated by
tests that run the full rendering and capture stacks across a dense glove, a
sparse tiny layout, and a collinear robot finger. Modality extensibility is
plugin-shaped (detectors, resolvers, exporters). Transport extensibility is
seam-shaped: the USB backend accepts any object satisfying a minimal `read`
contract, pyusb loads lazily behind an optional extra, and the framing is
documented as transport-agnostic (CAN-FD / 10BASE-T1S candidates behind the
same byte format). Scalability at current scope is process-level (batch
sessions, bounded-memory writers such as chunked HDF5 appends); nothing in the
artifact-passing design precludes distributing stages later, since every stage
boundary is already a file format.

## 5.5 Testing strategy

The test architecture is layered to match the invariants:

- **Executable byte specifications.** Firmware↔host parity is pinned by a test
  that reconstructs expected bytes independently from the C struct layout —
  the C compiler and the Python codec must independently agree with a third
  artifact.
- **Property checks over outputs**, not just unit assertions: grid
  monotonicity, bounded step, no-fabricated-gaps run inside the pipeline on
  every result.
- **Ground-truth round-trips.** Synthetic scenarios with known clock models and
  injected corruption (jitter, Bernoulli and Gilbert loss) verify estimation
  accuracy against an explicit budget document.
- **Descriptor-parametrized suites.** Viz and Capture tests run the same code
  across multiple topologies to prove count-agnosticism structurally.
- **Cross-repo integration (INT-1)** asserts what no single repo can: the
  descriptor hash threads unchanged through every stage; the persisted sync
  bundle round-trips; datasets are signed and hash-linked to their source
  episodes; content hashes reproduce across runs; and values survive
  Sim→Sync→Silver→Viz to float32 precision.
- **Ecosystem validation (CON-2)** checks that every sibling repo imports the
  contracts package and optionally runs all suites.

Current suite sizes are recorded in `ECOSYSTEM.md`; the notable point is not
the counts but the distribution — the synchronizer, where the subtle math
lives, carries the large majority of the tests.

## 5.6 Reproducibility as an architectural property

Reproducibility is not a script; it is the composition of: deterministic
seeding (Section 4.8), canonical serialization and hashing, provenance closure
(dataset → manifest → signed sidecar → descriptor hash → hardware revision),
append-only export records on the session, and an orchestrator that persists a
complete run manifest (resolved configuration, per-stage status and timing,
outputs, verdict, content hash) whether the run succeeds or fails.

---

# 6. Current Scope

## 6.1 Implemented (running code, tested)

- **Contracts:** frozen wire codec (C + Python, byte-parity tested), raw-frame
  schema v1, topology descriptor with canonical SHA-256 identity
  (`Mark2_v1`: 21 magnetometers + 3 accelerometers), column convention,
  shared NaN-aware derived-feature math, four bundled JSON-Schemas, ecosystem
  validator.
- **Synthetic producer:** full L0–L7 forward pipeline (piecewise-linear gesture
  profiles, lumped-compliance contact, dipole field, datasheet-true
  transduction/noise/quantization, decimated multi-rate timestamping), tiered
  parameter registry with placeholder gating, Hard Mode degradations,
  deterministic seeded dataset builder, Parquet/MCAP/LeRobot export.
- **Real-hardware host path:** shared-codec decode with CRC resync, device
  simulator replaying episodes over the real wire, session
  recording/manifesting, sync-evidence and health capture, MCAP mirroring,
  injectable USB seam; firmware C source mirroring the wire contract
  (bring-up single-core I²C loop; byte parity anchored by test).
- **Synchronization:** deterministic event detection and cross-device
  association; OLS/TLS/RANSAC/piecewise-affine clock estimation with robust
  rejection; confidence-weighted graph reconciliation with affine path
  composition; integer-µs reference grid; tolerance-gated hold/linear as-of
  join; three-component confidence with extrapolation decay; QA metrics and
  graded gate; persistable `SyncResult` bundles.
- **Materialization:** self-describing payload resolution (Parquet/MCAP),
  canonical Silver, five Gold exporters plus opt-in derived features,
  four-family validation with release gate and sync ceiling, Merkle/Ed25519
  provenance with topology closure, reproducible content hashes, inspect/diff,
  CLI.
- **Visualization:** descriptor-driven PCA projection, IDW heatmaps with
  data-derived support, cluster/centroid/shear views on shared math,
  display-only normalization policies and TPS anatomical warp, sequence/video
  rendering, live sessions — all topology-parametrized.
- **Cross-repo:** one-command orchestrator (ORCH-1), integration suite
  (INT-1), released synthetic dataset (v0.3) with signed, certified provenance.

## 6.2 Under development (seams and partial implementations in the tree)

- **On-silicon firmware bring-up:** production dual-core + PIO I3C architecture
  is documented and stubbed (PIO programs, I3C bus layer, LIS FIFO batching,
  hardware strobe marked TODO); shipped firmware is a single-core I²C bring-up
  loop, not yet validated on hardware.
- **Live USB capture:** the injectable-endpoint backend, GET_INFO parsing, and
  CLI path exist and are tested against fakes; real-device operation awaits
  hardware (and reconciliation of placeholder VID/PIDs).
- **Sub-frame bucketing:** implemented and tested; inactive in the default
  single-glove path pending an anchor (vision) stream.
- **RLDS:** portable step-dict export ships; the TFDS wrapper does not.
- **Phase-4 hook seams:** authorization and watermark hooks exist as explicit
  no-ops.

## 6.3 Planned / future (contracts or roadmap only — no implementation)

- **Calibration application:** the per-serial calibration bundle schema
  (baseline, gain/offset, soft-iron matrix, temperature coefficients, refined
  positions) is defined; the estimation and application mathematics are not
  implemented anywhere.
- **Vision stream and camera genlock:** strobe evidence and manifest fields
  exist; no camera capture, no video export, no tactile-visual alignment at
  scale.
- **Autolabeling engine** (Physical Data Engine, manual Phase 3) and
  **transactional catalog / commerce** (Phase 4), including per-licensee
  watermarking.
- **Absolute physical calibration of the simulator** (UNKNOWN-tier constants:
  magnet remanence, tissue elasticity, friction) — gated behind placeholder
  flags until measured.
- **Longer-horizon:** embodiment retargeting (manual Phase 10); nanosecond
  timestamp contract revision; calibrated (probabilistic) confidence.

These categories are kept deliberately un-blurred; when in doubt, the
per-repository documentation states which side of the line a capability is on.

---

# 7. Engineering References

| Topic | Where to read |
|---|---|
| Ecosystem architecture, repo map, ownership rules, invariants | [`ECOSYSTEM.md`](ECOSYSTEM.md) |
| Copy-pasteable operational workflows (simulate, capture, orchestrate, materialize, visualize, validate, release) | [`WORKFLOWS.md`](WORKFLOWS.md) |
| Shared contracts: wire codec, frame schema, descriptor, columns, derived math, bundled JSON-Schemas | `SentrixContracts/` (README; `src/sentrix_contracts/`; `c/sentrix_frame.h`; `schemas/`) |
| Simulator physics, parameter tiers, Hard Mode, assumptions | `SentrixSim/` (README; `docs/ASSUMPTIONS.md`; `docs/HARDMODE_failure_modes.md`; `configs/parameters.yaml`) |
| Firmware and capture architecture, USB protocol, storage layout | `SentrixCapture/docs/` (`ARCHITECTURE`, `FIRMWARE`, `USB_PROTOCOL`, `STORAGE_LAYOUT`) |
| Synchronization contract, estimation notes, accuracy budget, sub-frame design | `SentrixSync/` (README; `CONTRACT.md`; `IMPLEMENTATION_NOTES.md`; `SYNTHETIC_ACCURACY_BUDGET.md`; `SUBFRAME_BUCKETING.md`) |
| Canonical schema, materialization pipeline, multi-device validation | `SentrixDataEngine/docs/` (`CANONICAL_SCHEMA`, `USER_GUIDE`, `MULTI_DEVICE_VALIDATION`, `SENTRIX_ECOSYSTEM_GUIDE`) |
| Visualization SDK boundaries and dashboard architecture | `SentrixViz/` (README; `docs/DASHBOARD_ARCHITECTURE.md`) |
| End-to-end orchestration and integration invariants | `tools/orchestrator/`; `tests/integration/` |
| Released dataset (honest synthetic-vs-real framing) | `SentrixSim/SentrixDataset_v0.3_FINAL/` (`README`, `EXECUTIVE_SUMMARY`, `LIMITATIONS`, `BENCHMARKS`) |
| Long-term staging (marketplace → engine APIs → embodiment) | `Final/Sentrix Physical Data Engine Architecture Manual.docx` |

Key source directories for the mathematics discussed in Section 3:
`SentrixSync/src/sentrixsync/{clock,sync,detect}/`,
`SentrixSim/src/sentrixsim/layers/`,
`SentrixContracts/src/sentrix_contracts/derived.py`,
`SentrixDataEngine/src/sentrixdataengine/{materialize,validate,package}/`,
`SentrixViz/src/sentrixviz/{core,layers,render,normalize}/`.
