# Corrective: disputed performance / MTP claims on the 7 cheap-GPU campaign recipes

Branch `campaign-corrective-disputed-perf` (off `main`@e8530c75). **Prepared, NOT merged,
NOT accepted.** Scope: the 7 validated recipes landed by PRs #56, #59, #60, their
speed-sweeps, and a public evidence mirror for them. **No `trust.py` / policy changes.**
Window and concurrency rulings are ExperimentAudit's; status semantics per Main/Opus.

## What `status: validated` means here

`validated` is the registry's **existing launch/acceptance contract only** — the recipe is
runnable (executable launch, digest-pinned image, full model revision, ports), and the
functional acceptance probes (chat/reasoning/tools/vision) passed. It does **not** mean the
protocol-v2 benchmark (throughput / MTP / concurrency) is verified. Those numbers were never
part of the contract `trust.py` derives status from; they are now explicitly marked disputed
in-record. Under the current contract every one of the 7 still qualifies, so **nothing
demotes**; the PRO 4000 / 4090 case is fixed by lowering the advertised window, not by
demotion.

## Why the numbers are disputed

- Per-stream decode throughput is **TabbyAPI server-reported per-request rate**, not
  client-timed aggregate (`decode_tok_s_aggregate: null`,
  `per_stream_decode_method: "median of TabbyAPI server-reported per-request rates"` in the
  raw rows).
- `serving.max_concurrency` was a **queued request count, not a certified capability**.
- MTP acceptance rates were **engine-log-derived**, never independently verified.
- Flash-Next MTP is **0/0** (configured, model ships an MTP head, but 0 accepted / 0
  rejected across 28 streams).
- 262144 on the two 24 GB cards (PRO 4000, 4090) was served **once, non-reproducibly**
  (multi-sample 262144 rows `infra-invalid`; 4090 serve log has 13 `torch.OutOfMemoryError`).

## Data corrections (registry records only; raw is never edited)

Per recipe (`registry/recipe/*.json`; `metadata.measurements` and `serving` are free-form):

- **MTP**: `mtp_acceptance` → `null`; raw counts moved verbatim to
  `metadata.measurements.mtp.engine_reported`; `mtp` carries `status`,
  `measurement_basis: engine-log-derived`, reason.
  - Flash-Next: `mtp.status = "no-speculative-tokens-observed"` (0/0).
  - Ornith: `mtp.status = "not-enabled"`, `upstream_support: "unverified"` — kept distinct
    from upstream-unsupported; the old "ships no MTP head" claim was unevidenced.
- **Concurrency** (ExperimentAudit + Main): `serving.max_concurrency` claim **dropped
  (`null`)**. Only client-side timestamps exist, so observations live under
  `serving.concurrency_observed`: `dropped_claim`, `observed_client_stream_overlap`
  (`{"32768": 2, "<advertised>": 1}`, or `null` for Ornith which has no per-row latency),
  `engine_batch: "unknown - not certifiable without engine-side instrumentation"`,
  `admission_behaviour`, `basis`. Client intervals show queue/overlap behaviour but do
  **not** certify the engine's internal GPU batch. No invented numbers.
- **Advertised window** (ExperimentAudit): PRO 4000 and RTX 4090
  `serving.max_context_tokens` → **131072** (reproducibly served, 12 / 13 completed rows);
  the single 262144 observation stays in the raw sweep as a disputed observation, per the
  campaign rule that a recipe never advertises a window the run did not serve. Other five
  keep their reproducibly-served windows.
- **Dispute block** `metadata.measurements.dispute`: shared findings + recipe-specific note.
  The cache-state finding is a **derived observation only** (`cached_tokens=0` in every raw
  record ⇒ cold); the immutable raw is not edited.
- **Descriptions**: "measured C1/C2/C4 acceptance" clause removed; MTP wording matched to
  evidence; window language corrected for PRO 4000 / 4090.

Per speed-sweep (`registry/speed-sweep/*.json`): `metrics.peak_generation_tps` → `null`
(engine-reported decode headline). **All raw `rows[]` kept verbatim.**

## Public evidence mirror (`evidence/omp-cheap-gpu-2026-09-18/`)

Origin `0xSero/inference-index` is archived + private; the 7 deliverables' artifacts are
mirrored into this public repo, verbatim, so the evidence is resolvable. Each
`<recipe-id>/manifest.json` carries the sha256 + byte size of every file, origin
repository/commit/paths, model revision, image digest, a **LEGACY / PROVISIONAL** label,
and `test_status` (`functional_acceptance: accepted`, `benchmark_metrics: disputed`,
`protocol_v2_verified: false`). The original raw stays local and intact.

Secret audit (auditable): scanned private IPv4, IP sockets, RunPod proxy hosts, SSH host
keys, PEM private keys, bearer/authorization, api/secret/access/`hf_`/RunPod keys across all
195 files (3.8 MB). Only matches: bind address `0.0.0.0:5000` and localhost `127.0.0.1`
(not sensitive). **Zero redactions**; byte-for-byte mirror.

Speed-sweep `source` still points at the origin commit: repinning to
`local-ai-registry` + tag SHA is a **post-merge** step (the resolvable SHA does not exist
until merge; a fabricated SHA is not written). Command in the mirror README.

## Claims still unverifiable from current evidence

1. Client-timed aggregate throughput (Flash-Next 262144/C4 end-to-end = 512 tok / 452.8 s =
   **1.13 tok/s** vs 22.95 headline).
2. Any engine/GPU batch or concurrency ceiling — client overlap is a lower-bound
   observation, not engine instrumentation.
3. MTP acceptance as a rate; Flash-Next MTP undefined (0/0).
4. Reproducible 262144 on 24 GB (PRO 4000, 4090) — needs re-measurement.
5. Warm-cache behavior (all TabbyAPI cold; Ornith has no per-row latency).
6. Whether Ornith-1.5-9B ships an SGLang MTP head.

Out of scope, flagged to Main: `tabbyapi:cu13@1c13dd17` is CUDA 13.2.1 (needs `cuda>=13.2`)
— portable only to 13.2-capable cards; 3-way image-provenance divergence (hardware/ recipes
on `ffa8388f`, campaign artifacts on `1c13dd17`, `0xsero/tabbyapi-exl3` only in campaign
docs) for ImageDelivery.

## Pre/post checks (run by Main after edits settle — not run here mid-flight)

```bash
cd <local-ai-registry worktree>   # branch: campaign-corrective-disputed-perf
# pre: prove records + manifests are self-consistent
python3 scripts/trust.py                                # expect: 0 recipe(s) disagree
python3 - <<'PY'                                        # verify every manifest sha256 matches the mirrored bytes
import json,hashlib,glob,os
for m in glob.glob("evidence/omp-cheap-gpu-2026-09-18/*/manifest.json"):
    d=json.load(open(m)); root=os.path.dirname(m)
    for f in d["files"]:
        h=hashlib.sha256(open(os.path.join(root,f["path"]),"rb").read()).hexdigest()
        assert h==f["sha256"], (m,f["path"])
print("manifest hashes OK")
PY
# regenerate derived files the edits invalidate
python3 scripts/curate_registry.py --index-only
python3 scripts/export_plugin_recipes.py --out plugin/recipes.json
python3 scripts/format_registry.py
# post: full CI
make check
```

`registry/index/*`, `plugin/recipes.json`, `registry/schema/types.ts` are generated —
regenerate them (+ `make types`), do not hand-edit. Do not merge without repo-owner review.
