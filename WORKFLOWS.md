# Sentrix Workflows

> A practical, copy-pasteable cookbook. For architecture and repo boundaries see
> [`ECOSYSTEM.md`](ECOSYSTEM.md). All commands assume the shared venv from
> ECOSYSTEM.md → *Developer Setup* is active and the working directory is the repo
> root `D:\New_Direction\Glove_Architecture` unless noted.

```bash
cd D:\New_Direction\Glove_Architecture
. .venv/Scripts/activate          # PowerShell: .venv\Scripts\Activate.ps1
```

---

## 1. Generate a synthetic dataset (SentrixSim)

```bash
# one episode, all formats
sentrixsim simulate --event grasp --out ./out/grasp \
    --formats parquet,mcap,lerobot --seed 0

# every gesture once
sentrixsim simulate-all --out ./out/all

# a balanced 900-episode dataset (9 gestures × 100); clean baseline
sentrixsim build-dataset --out ./dataset_v0.1

# the Hard Mode variant (dropouts, jitter, drift, hard negatives)
sentrixsim build-dataset --out ./dataset_v0.2 --version 0.2 --hard-mode

# inspect what's available
sentrixsim list-events
sentrixsim show-params --tier UNKNOWN
```

Output: per-episode `raw.parquet` (sensor-id-keyed: `mag.<sid>.*` / `dyn.<sid>.*`),
plus MCAP / LeRobot exports and `manifest.json` / `stats.json` / `validation.json`.
Use `--descriptor <version|path>` to render a different topology with zero code
change; `--legacy-columns` emits the back-compat Layout-B names.

## 2. Record / replay with hardware (SentrixCapture)

```bash
# hardware-free: replay a SentrixSim episode over the REAL wire → a capture session
sentrixcapture record --from-sim ./out/grasp/raw.parquet \
    --descriptor Mark2_v1 --session s001 --device-uid glove_L --out ./captures

# topology summary / descriptor validation
sentrixcapture inspect Mark2_v1
sentrixcapture validate-descriptor path/to/descriptor.json

# live USB (needs real hardware + the `usb` extra) drops in behind the same path:
# sentrixcapture record --from-usb --descriptor Mark2_v1 --session s001 --out ./captures
```

Output (under `./captures/s001/devices/glove_L/`): `raw.parquet` (byte-compatible
with SentrixSim), optional `raw.mcap`, `sync_evidence.parquet`, `health.parquet`,
`descriptor.json`, and a `session_manifest.json` SentrixSync ingests directly.

## 3. Run the ORCH-1 pipeline (Sim → Sync → DataEngine, one command)

```bash
# full default run (event=grasp, Parquet + LeRobot gold)
python tools/orchestrator/run_pipeline.py

# pick a gesture / seed / formats
python tools/orchestrator/run_pipeline.py --event press --seed 1 \
    --gold-formats parquet,lerobot,mcap

# other knobs: --descriptor --device-id --grid-rate-hz --rejection-tolerance-us
#              --allow-placeholders --runs-root runs --run-id <fixed-id>
```

Prints a per-stage OK/XX summary and, on success, the Silver path, each Gold
format, the manifest path, the QA verdict, and the content hash. Artifacts and a
`run_manifest.json` land under `runs/<run_id>/`. Exit code 0 = success.

## 4. Materialize a dataset from a Sync bundle (SentrixDataEngine)

SentrixSync (**SYNC-1**) can persist a `SyncResult` + `Session` bundle to disk;
the DataEngine `materialize` CLI (**DE-CLI-1**) runs the full
resolve → materialize → validate → export → package pipeline from it — no live
SentrixSync process:

```bash
sentrixdataengine materialize \
    --bundle <syncresult_bundle_dir> \
    --out gold \
    --formats parquet,lerobot,mcap,rlds,hdf5 \
    --version 0.1.0
# prints JSON: verdict, dataset path, silver, content_hash, manifest, provenance
```

Materializing directly from an **in-memory** `SyncResult` (e.g. when you already
ran `synchronize(...)`) uses the Python API:

```python
from pathlib import Path
from sentrixdataengine import Pipeline, MaterializationRequest

result = Pipeline().run(MaterializationRequest(
    sync_result=sync_result, session=session, out_root=Path("gold"),
    formats=("lerobot", "mcap", "rlds", "hdf5", "parquet"),
    format_options={"lerobot": {"layout": "v3"}}))
print(result.qa.gate_verdict, result.layout.base)
```

Opt-in topology-dependent features: add `"derived"` to `formats` (needs a topology
descriptor via `sentrix_contracts`).

## 5. Visualize a dataset (SentrixViz)

```bash
sentrixviz inspect raw.parquet                              # summarize an artifact
sentrixviz topology --descriptor Mark2_v1 -o topo.png       # static geometry
sentrixviz render raw.parquet --view heatmap -o frame.png   # one frame
#   --view {raw|heatmap|clusters|centroid|shear}
sentrixviz render-sequence raw.parquet --out seq/           # PNG sequence + manifest
sentrixviz render-video --seq seq/ -o out.mp4               # MP4 (ffmpeg) or GIF
sentrixviz live --synth                                     # live synthetic feed
sentrixviz render silver.parquet --silver -o frame.png      # render a Sync silver artifact
```

## 6. Validate provenance & QA

```bash
# QA verdict for a packaged dataset (CERTIFIED | RELEASE | NEEDS_REVIEW | BLOCKED)
sentrixdataengine validate --dataset "gold/dataset=<id>/version=0.1.0"

# summary stats (coverage, confidence, value ranges, content hash)
sentrixdataengine inspect --dataset "gold/dataset=<id>/version=0.1.0"

# compare two dataset versions (content-hash identity + per-stream deltas)
sentrixdataengine diff --a <ver_dir_A> --b <ver_dir_B>
```

Each packaged dataset carries a `manifest.json`, a Merkle-rooted, Ed25519-signed
`provenance.sidecar.json` (HMAC fallback is flagged unsigned and BLOCKs the gate),
a `DATACARD.md`, and a `qa_report.json`. Per-device topology provenance
(`topology_ref` / `topology_hash`) is closed through the Silver KV, manifest,
signed sidecar, and data card — so a delivered dataset traces back to the exact
descriptor (and hardware revision) it came from.

To verify the whole stack wires together, run the cross-repo integration suite
(**INT-1**) and the contracts ecosystem validator (**CON-2**):

```bash
pytest tests/integration -q
python SentrixContracts/tools/validate_ecosystem.py            # add --no-tests to skip suites
```

## 7. Create a buyer dataset release

The current release lives at `SentrixSim/SentrixDataset_v0.3_FINAL/`
(**SentrixTactileMotionDataset v0.3**). Reproduce / refresh the underlying data and
package a release:

```bash
# 1. regenerate the balanced source data (deterministic, seeded)
sentrixsim build-dataset --out ./dataset_v0.2 --version 0.2 --hard-mode

# 2. run the full pipeline per gesture to get CERTIFIED, signed datasets
python tools/orchestrator/run_pipeline.py --event grasp \
    --gold-formats parquet,lerobot

# 3. confirm the release gate is CERTIFIED with signed provenance
sentrixdataengine validate --dataset "runs/<run_id>/.../version=0.3.0"
```

The release directory is **self-contained** and buyer-facing — start from its
`README.md` / `EXECUTIVE_SUMMARY.md`, with `BENCHMARKS.md` and `LIMITATIONS.md`
for the honest synthetic-vs-real account. Keep the framing honest: real today =
schema/formats/cadence/pipeline/provenance; simulated = absolute magnitudes;
unverified until first physical capture = calibration and real-world robustness.
No vision/camera data ships in v0.3.
