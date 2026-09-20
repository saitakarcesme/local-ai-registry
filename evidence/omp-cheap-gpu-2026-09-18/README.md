# Evidence mirror — `omp-cheap-gpu-2026-09-18` (LEGACY / PROVISIONAL)

Public, resolvable mirror of the campaign artifacts that back the 7 recipes landed by PRs
#56, #59, #60. Origin (`0xSero/inference-index`) is archived and private; this copy exists
so the evidence is publicly replayable. **The original raw stays local and intact — it is
not modified by this mirror.**

## Status label

**LEGACY / PROVISIONAL — pre-protocol-v2.** These artifacts predate the protocol-v2
benchmark (`campaigns/omp-cheap-gpu-2026-09-18/DELIVERY-24.md`). What they establish and
what they do not:

- **Verified:** functional acceptance only — chat, reasoning, tools, vision passed
  (`omp_accept.py`), the model loaded and served, the image digest and model revision are
  pinned. This is exactly what the registry's `status: validated` means: the existing
  launch/acceptance contract, **not** benchmark verification.
- **Disputed (not verified):** all throughput (`decode_tok_s*`, `prefill_tok_s`,
  `peak_generation_tps`), MTP acceptance, and request concurrency. These are
  engine-reported / server-log-derived / client-timestamp-derived, not protocol-v2
  measurements. See each recipe's `metadata.measurements.dispute`.

## Layout

```
<recipe-id>/
  manifest.json      # sha256 of every mirrored file + provenance + label + test_status
  artifacts/         # verbatim copy of the origin artifact tree
```

## Integrity & redactions

Each `manifest.json` carries the sha256 and byte size of every file, the origin repository
/ commit / path, the model revision, and the image digest. An auditable secret scan (private
IPv4, IP sockets, RunPod proxy hosts, SSH host keys, PEM private keys, bearer/authorization
headers, api/secret/access tokens, `hf_`/RunPod keys) found only the server bind address
`0.0.0.0:5000` and localhost `127.0.0.1`, which are not sensitive. **Zero redactions were
needed; content is mirrored byte-for-byte.** Any future redaction MUST be a documented,
auditable transform recorded in `manifest.json.redactions` — never a silent hand edit.

## Repinning (post-merge)

After this branch merges and an immutable tag `evidence/omp-cheap-gpu-2026-09-18@<sha>` is
cut, repin each recipe's speed-sweep `source` (`repository` →
`https://github.com/0xSero/local-ai-registry`, `paths` → `evidence/omp-cheap-gpu-2026-09-18/<recipe-id>/artifacts`,
`commit` → the tag SHA). It is left pointing at the origin here because the resolvable SHA
does not exist until merge; repinning to a fabricated SHA is not done.
