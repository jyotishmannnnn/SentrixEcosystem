# Sentrix Ecosystem

> The canonical entry point for new contributors. Read this, then
> [`WORKFLOWS.md`](WORKFLOWS.md), then the relevant repo `README.md`. Per-repo
> rules live in each repo's `CLAUDE.md`.

---

## Overview

**SentrixEcosystem** is a physical-data pipeline that turns motion
demonstrations into searchable, premium, ML-ready datasets for robotics and
physical-AI teams (VLA, dexterous manipulation, world/reward models). It grounds
the physical signal — contact, force, timing, confidence — losslessly and
provably, then projects it into the formats labs train on (LeRobot, RLDS, HDF5,
MCAP, Parquet).

The stack is **descriptor-driven and topology-aware**: one shared topology
descriptor defines the sensor layout; both a synthetic producer (SentrixSim) and a
real-hardware producer (SentrixCapture) emit the *same* sensor-id-keyed raw
artifact; synchronization and materialization are count-agnostic; and every
dataset is provenance-closed back to the exact hardware revision it came from.

**Maturity:** end-to-end on synthetic data today. The real host capture path is
implemented and verified; firmware is real with on-silicon bring-up pending.
Autolabeling and the transactional catalog are not built.

## Repository Map

| Repo | Version | Role | One-line |
|---|---|---|---|
| **SentrixContracts** | 0.1.0 | Shared contracts | Wire codec, raw-frame schema, topology descriptor, column convention. Everyone imports it; nothing else owns these definitions. |
| **SentrixSim** | 0.1.0 | Synthetic producer | Descriptor-driven forward simulator (L0–L7); 9 gestures; emits raw per-device episodes. |
| **SentrixCapture** | 0.1.0 | Real-hardware producer | RP2350 firmware + host capture engine; emits a raw artifact byte-compatible with Sim. |
| **SentrixSync** | 0.4.0 (contract 1.1.0) | Synchronizer | Modality-neutral multi-device clock sync → in-memory `SyncResult` + `Session` manifest. |
| **SentrixDataEngine** | 0.1.0 | Materializer | `SyncResult` → canonical Silver table → Gold exports → validate → package with provenance. |
| **SentrixViz** | 0.1.0 | Visualization SDK | Topology-driven rendering (PNG / sequence / video / live). Sits beside the pipeline. |

Root tooling:

- **`tools/orchestrator/`** — **ORCH-1**: one-command Sim → Sync → DataEngine run.
- **`tests/integration/`** — **INT-1**: cross-repo end-to-end pipeline tests.

## Data Flow

Strictly one-way. No repo imports a downstream one; Sync/DataEngine never import a
producer.

```
Descriptor (sentrix_contracts: Mark2_v1, …)
   │
   ├─ SentrixSim ─────┐
   │ (synthetic)      │
   │                  ├─→ SentrixSync ──→ SentrixDataEngine ──→ Gold datasets + provenance
   └─ SentrixCapture ─┘   synchronize       materialize
     (real hardware)      N devices          export · validate · package

   SentrixViz consumes the raw artifact (or Sync silver) at any point for rendering.
```

- **Producers** (Sim, Capture) emit the *same* self-describing `raw.parquet`
  (a device-local timestamp column + `sensor_id`-keyed payload columns + topology
  metadata). They are interchangeable.
- **SentrixSync** depends on producer *output artifacts*, not producer code. It
  reconciles N device clocks onto one reference timeline and returns a `SyncResult`
  (grid + as-of join indices + three-component confidence) plus a `Session`
  manifest. It writes **no datasets**.
- **SentrixDataEngine** depends on Sync *types* (read-only). It resolves the
  payload-ref URIs Sync recorded, applies the join indices to build one canonical
  Silver table, projects to Gold formats, validates, and packages with signed
  provenance. It never mutates `SyncResult` (the only write-back is appending an
  `ExportRecord` to the `Session`).
- All producers + DataEngine + Viz depend on **`sentrix_contracts`** for topology,
  columns, and frames.

## Contracts

`SentrixContracts` (import name `sentrix_contracts`) is the single source of truth
for the cross-stack domain contracts. It is standalone, typed, and has zero
runtime third-party dependencies (core).

| Module | Contract |
|---|---|
| `descriptor` | Topology runtime model (`Descriptor`, `Sensor`, `Cluster`, `load_descriptor`, `bundled_descriptor_path`). A new hardware revision is a new JSON file, not code. |
| `frame` | Decoded raw-frame dataclasses (`FieldFrame`, `DynFrame`, `SyncEdge`, `HealthFrame`). Count-agnostic, `sensor_id`-keyed. `SCHEMA_VERSION = 1`. |
| `wire` | On-wire codec — Python mirror of `c/sentrix_frame.h`. `MAGIC\|VER\|LEN\|payload\|CRC32`, frozen byte layout (`WireMap`, `encode_*`, `decode_payload`, `iter_frames`). |
| `columns` | At-rest naming (`mag.<sensor_id>.bx_uT`, `dyn.<sensor_id>.ax_g`); `column_for`, `parse_column`, `mag_columns`, `dyn_columns`. |
| `derived` (extra) | Cluster math (`normal_proxy`, `shear_*`, `centroid`); needs `numpy`. |

Bundled inside the package: `descriptors/Mark2_v1.json` (canonical Layout-B,
`descriptor_hash: sha256:392c8627ae88…83a5`) and the four JSON-Schemas
(`raw_frame`, `topology_descriptor`, `session_manifest`, `calibration_bundle`).
The wire format, schema version, descriptor hash, and column convention are
**frozen**. **CON-2** ships `tools/validate_ecosystem.py`, which checks that every
sibling repo imports the package and (optionally) runs their suites.

## Dataset Pipeline

How a dataset is produced, stage by stage:

1. **Produce raw episodes.** SentrixSim simulates a gesture from a descriptor, or
   SentrixCapture records/replays one over the real wire → `raw.parquet`
   (sensor-id-keyed, self-describing).
2. **Synchronize.** SentrixSync ingests one or more producer artifacts, detects
   shared events, reconciles clocks over a graph, and returns a `SyncResult`
   (reference grid + per-stream join indices + confidence) and a `Session` manifest.
3. **Materialize.** SentrixDataEngine resolves payload refs, applies the join
   indices to build **one canonical Silver table**, and writes `silver/aligned/part-000.parquet`.
4. **Export.** The Silver table is projected to Gold formats — every format is a
   pure projection of the same source of truth (LeRobot v2/v3, MCAP, RLDS, HDF5,
   Parquet, and an opt-in topology-dependent `derived` export).
5. **Validate.** Schema / timeline / metadata / confidence checks plus a release
   gate verdict: `CERTIFIED | RELEASE | NEEDS_REVIEW | BLOCKED`.
6. **Package.** Gold directory layout + dataset manifest + Merkle-rooted,
   Ed25519-signed provenance sidecar + data card + reproducible content hash.
   Per-device topology provenance (`device_id → topology_ref → topology_hash`) is
   carried into Silver KV, the manifest, the signed sidecar, and the data card —
   closing the lineage from a delivered dataset back to the hardware revision.

The single-command form of steps 1–6 is **ORCH-1** (see WORKFLOWS.md).

## Current Dataset Release

**SentrixTactileMotionDataset v0.3** — `SentrixSim/SentrixDataset_v0.3_FINAL/`.
https://drive.google.com/drive/folders/1jah21e-l8auIvuoOENOcJzRzc4gdTCRG?usp=sharing

A *hardware-aligned synthetic* dataset of synchronized tactile + motion sensing
for robotic / XR manipulation. Reproducibility id `sha256:392c8627ae88…83a5`
(pins the entire release; equals the `Mark2_v1` descriptor hash).

- **24 sensing points → 75 channels** on one microsecond timeline: tactile
  (21 points / 63 ch, 400 Hz, µT), motion (3 / 9 ch, 1.6 kHz, g), thermal
  (3 / 3 ch, ≤50 Hz, °C).
- **900 episodes**, 9 manipulation categories, class-balanced; realistic
  conditions (dropouts ~0.95 %, jitter, drift, 50 hard-negative episodes).
- Formats: **Parquet** (analysis) + **LeRobot v3** (robot learning).
- **Validated:** 900/900 quality-gate pass; 9/9 full-pipeline runs **CERTIFIED**
  with signed provenance.

**Honest framing:** real today = schema, formats, signal character, dynamic range,
cadence, pipeline, provenance/QA. Simulated = absolute magnitudes, signal shapes,
augmentations. Unverified until first physical capture = absolute calibration,
multi-stream alignment at scale, real-world robustness. **No vision/camera data.**
See the release's `README.md`, `EXECUTIVE_SUMMARY.md`, `BENCHMARKS.md`, `LIMITATIONS.md`.

## Hardware Status

- **Host capture path:** real and tested. Shared wire codec
  (`sentrix_contracts.wire`) + device simulator + `transport`/`capture` →
  `raw.parquet` that ingests unchanged into Sync and DataEngine (verified
  end-to-end, values bit-faithful). 12 Capture tests pass; `test_firmware_parity.py`
  pins firmware↔host byte parity.
- **Firmware:** real C source (RP2350 + Pico SDK) mirroring the byte-parity-anchored
  wire contract. **Not yet validated on silicon** (no ARM toolchain build here);
  on-silicon bring-up (PIO I3C buses, USB enumeration) is the next milestone.
- **Live USB backend** (pyusb): wired-but-pending seam. Today the host runs via the
  device simulator replaying Sim episodes over the real wire.
- **No vision stream** → no video export; camera genlock is an architectural seam.
- **Not built:** autolabeling (manual Phase 3), transactional catalog (Phase 4).

## Developer Setup

Python **3.11+** (developed/tested on CPython 3.12). The repos are installed
editable from source into a shared virtual environment; there are no published
wheels. Install in dependency order (contracts first).

```bash
cd D:\New_Direction\Glove_Architecture
python -m venv .venv
. .venv/Scripts/activate          # PowerShell: .venv\Scripts\Activate.ps1

pip install -e SentrixContracts                 # shared contracts (no deps)
pip install -e SentrixSim                        # producer (numpy/scipy/pyarrow/mcap)
pip install -e "SentrixSync[dev]"                # synchronizer (+pyarrow)
pip install -e "SentrixDataEngine[dev]"          # materializer (+h5py/cryptography/mcap)
pip install -e "SentrixCapture[dev]"             # capture host engine
pip install -e "SentrixViz[dev]"                 # visualization (+matplotlib)
```

Run each repo's tests (expected pass counts):

```bash
pytest SentrixSim/tests -q          # 23
pytest SentrixCapture/tests -q      # 12
pytest SentrixSync -q               # 215
pytest SentrixDataEngine/tests -q   # 41
pytest SentrixViz/tests -q          # 11 modules, descriptor-parametrized
pytest SentrixContracts -q          # package unit tests
pytest tests/integration -q         # INT-1 cross-repo end-to-end
python SentrixContracts/tools/validate_ecosystem.py   # CON-2 ecosystem validator
```

## Common Workflows

Copy-pasteable commands live in [`WORKFLOWS.md`](WORKFLOWS.md). In brief:

- **Generate simulation data** — `sentrixsim simulate` / `build-dataset`.
- **Materialize a dataset** — `Pipeline().run(MaterializationRequest(...))` from a
  `SyncResult` (Python; the timeline must be in memory).
- **Run visualization** — `sentrixviz render` / `render-sequence` / `live`.
- **Run integration validation** — `python tools/orchestrator/run_pipeline.py`
  (ORCH-1) and `pytest tests/integration` (INT-1).

## Repository Ownership Rules

What belongs in each repo — and, just as importantly, what does not.

| Repo | Owns | Never |
|---|---|---|
| **SentrixContracts** | Wire codec, raw-frame schema, topology descriptor, column convention, bundled descriptors/schemas. | Any pipeline logic; any per-repo behavior. New contracts go through architecture review. |
| **SentrixSim** | Glove physics, episode generation, parameter registry, Hard Mode, exporters. | Synchronization, datasets, downstream imports. |
| **SentrixCapture** | Firmware, USB transport, host capture, raw recording, session manifests, capture-time QA/viz. | Clock reconciliation, dataset materialization, edge calibration/fusion, downstream imports. |
| **SentrixSync** | Clocks, timeline, event association, graph reconciliation, confidence, QA. | Payload resolution, dataset/export, modality/topology branching, mutating its own output. |
| **SentrixDataEngine** | Resolve, materialize (canonical Silver), export, validate, package, provenance. | Clock/grid/join logic, producer imports, `SyncResult` mutation, training, catalog/commerce. |
| **SentrixViz** | Topology-driven rendering (geometry / frame / sequence / video / live). | Producer/consumer imports, geometry constants, sensor-count branching, persisting display signals. |

**Non-negotiable principles:** one-way dependency flow; one canonical Silver
representation (every Gold format is a projection); synchronization separated from
materialization; three confidence components carried verbatim; never fabricate
gaps (invalid frames stay NaN + confidence 0); immutable `SyncResult`; payloads by
reference; descriptor-driven and count-agnostic (a new hardware revision is a new
descriptor, not a code change).

---

## Documentation recommended for removal

*List only — do not delete automatically. Reviewed against the current
post-CON-2 / ORCH-1 / SYNC-1 / INT-1 / VIZ-P5 / SIM-HW-FIDELITY state.*

- **`ecosystem_review_outputs/`** (whole directory) — point-in-time audit/review
  artifacts (`sim1_report.md`, `sim3_report.md`, `int1_report.md`,
  `sync1_report.md`, `viz_p4_report.md`, `performance_report.md`,
  `orchestrator_*.md`, `failure_report.md`, `release_checklist.md`,
  `sentrix_ecosystem_critical_path.md`, `sentrix_ecosystem_tasklist.csv`,
  `final_release_recommendation.md`, `INTERNAL_TECHNICAL_APPENDIX.md`). Superseded
  by this `ECOSYSTEM.md` + `WORKFLOWS.md` + the per-repo READMEs. Archive outside
  the repos if a historical record is wanted.
- **Stale claims to correct (not remove)** in living status docs:
  - Root `PROJECT_STATUS.md` and `CLAUDE.md` describe `sentrix_contracts` as
    "vendored in SentrixCapture" — it is now the standalone **SentrixContracts**
    package (CON-1 extraction, CON-2 validator). Update those two lines.
  - Root `PROJECT_STATUS.md` references `contracts/schemas/` under SentrixCapture;
    schemas now ship inside the `sentrix-contracts` package.
- **Duplication to watch:** the root `SentrixDataEngine_DESIGN.md` overlaps with
  `SentrixDataEngine/docs/`. Keep one as canonical (the in-repo `docs/` is the
  natural home) and treat the root copy as a pointer if retained.
