# SPEC: Team AI Datastore

Status: v4 (final for v1 build) · Owner: Accelerating AI team · Target: team OpenShift cluster, lab network (VPN)

## 1. What this is

The simplest trustworthy data store for the Accelerating AI boulder: profiling/trajectory run data in, curated training and post-training datasets out. **One MinIO pod + a server-side extractor CronJob + a simple CLI (datalib) + strict conventions.** No database, no API service, no operators (app components are plain k8s primitives).

Decisions made and locked:
- Kubernetes primitives only; installable into a single namespace.
- Use cases in scope: **training data** and **post-training data**. Out of scope for v1: RAG/retrieval, SQL analysis service. Analysis via DuckDB reading JSONL on S3 directly.
- Producers run **anywhere**; access via the MinIO S3 Route over the lab network.
- **Producer integration is zero-friction (team feedback, 2026-07-16):** producers do NOT embed a library. They push opaque tarballs/dirs with a simple CLI (`datastore push`). Integrity enforcement moved **server-side** into an extractor that canonicalizes uploads. (Ben + Luigi thread: "push up a tar to an incoming dir... tool on the backend extract and catalog.")
- Metadata is **inferred, not demanded**: extractor parses producer naming conventions; optional `metadata.yaml` enriches. Hard requirement survives ONLY for external corpus ingest: `--source` and `--license` (train-ability is not answerable retroactively).
- Trajectory data default: **one record per trajectory**, steps as ordered artifacts (matches Luigi's `model/kernel/trajectoryXXXX/step0..stepX` shape). Revisitable cheaply — original tarballs are preserved, extractor can be re-run with per-step logic.
- Secrets: hand-created Kubernetes Secrets; `secrets.template.yaml` documents required keys. Vault later.
- Backups: **written plan, not built** (§8). Trigger: first real data we'd be sad to lose.
- System may be throwaway; **the data and conventions are not.**

## 2. Deployment (OpenShift, namespace `octo-datastore`)

Kustomize manifests in `deploy/base` + `deploy/overlays/lab`:
- **MinIO**: StatefulSet (1 replica), PVC on cluster StorageClass (LVMS on the added disk if none exists — check `oc get storageclass`). Bucket versioning ON. ClusterIP Service + **Route** (TLS) for the S3 API. No console Route in v1.
- **Extractor**: CronJob (every 5 min), ~200-line Python container. Lists `incoming/`, processes uploads (§4), moves results to `runs/` or `quarantine/`. Idempotent; concurrency-safe via processed-marker objects.
- NetworkPolicy default-deny ingress; allow openshift-ingress → MinIO. SCC restricted-v2 everywhere.
- Secrets: `minio-root` (admin) + per-producer access keys (`kernelsmith`, `perfeval`, `human-<name>`), hand-created from template.
- IAM policies as code (setup Job running `mc admin`): producers write only `incoming/*` and `scratch/*`; everyone reads `datasets/*`, `raw/*`, `runs/*`; only the extractor's key writes `runs/*` and `quarantine/*`; only admin finalizes `datasets/*`; deny `s3:DeleteObjectVersion` to non-admin (immutability).
- Setup Job creates bucket, lifecycle rules, IAM policies idempotently.

## 3. Bucket layout (`octo-ai`, versioning ON)

```
incoming/{producer}/...            # opaque tarballs/dirs, pushed by anyone with a key
quarantine/{upload_id}/           # failed extraction + error.txt (never silently dropped)
raw/{source}/{yyyy-mm-dd}/         # external corpus (docs, examples); immutable; license recorded
runs/{pipeline}/{run_id}/          # canonical: record.json + artifacts; written by extractor only
datasets/{name}/v{semver}/         # published JSONL shards + manifest.yaml; immutable
scratch/{user}/                    # 30-day lifecycle expiry
```

- `record.json` written **last** in a run prefix → presence = complete (crash-safe).
- Original tarballs are archived under `runs/{pipeline}/{run_id}/original.tar.gz` so extraction logic can be re-run later (e.g., switching trajectory→per-step records).
- Lifecycle: expire `scratch/` 30d; `incoming/` processed markers 7d; noncurrent versions 90d except `raw/`.

## 4. Extractor behavior (server-side integrity)

For each new object in `incoming/`:
1. If tarball: extract; if dir: use as-is.
2. Infer metadata: producer from key prefix; timestamp/model/kernel from naming convention (`<timestamp>/<model>/<kernel-name>.tar.gz`; internal `trajectoryXXXX/step0..stepX` → ordered steps). Merge optional `metadata.yaml` if present (wins over inference).
3. Build and validate `record.json` (schema §5). Trajectory uploads → one record per trajectory, steps as ordered artifact list.
4. Success → write artifacts + record.json + original tarball to `runs/{pipeline}/{run_id}/`; drop processed marker.
5. Failure → move to `quarantine/{upload_id}/` with human-readable `error.txt`. Quarantine is for genuinely unparseable uploads only; missing optional metadata is not a failure.
6. New producer naming conventions = small parser additions to the extractor (documented pattern list in repo).

## 5. The run record (most important artifact in the repo)

Pydantic model in datalib, `schema_version` in every record. v1 fields:

```
schema_version: "1"
run_id: uuid            pipeline: str          created_at: iso8601
producer: str           model_id: str|null     kernel: str|null
kind: run | trajectory
spec: {op, shapes, dtypes, target_arch, ...} | null      # null when not inferable
status: compile_fail | incorrect | correct | error | unknown   # failures kept — they are signal
steps: [ {index, artifacts:{...}} ] | null    # trajectories: ordered steps
correctness / benchmark / profile_summary: {...} | null
artifacts: {name: s3key, ...}
license: str            source: str|null       pipeline_sha: str|null
inferred_fields: [str]  # honesty marker: which fields came from inference vs explicit metadata
```

Evolution: additive → minor bump; breaking → bump `schema_version`, datalib reads all prior versions.

## 6. datalib (simple CLI first, Python package underneath)

CLI (`datastore`) is the product — per team request. Library exists for programmatic use but nothing requires embedding it. Config: `DATASTORE_ENDPOINT`, `DATASTORE_ACCESS_KEY`, `DATASTORE_SECRET_KEY`.

Producer verbs (zero-friction):
- `datastore push <tarball-or-dir>` — multipart upload to `incoming/{producer}/`; that's it. Optional `--meta metadata.yaml`.
- `datastore get <run_id | dataset@version> <dest>` — download run artifacts or dataset; datasets checksum-verified against manifest.

Curation verbs:
- `datastore ingest <path> --source X --license Y` — external corpus → `raw/`; refuses without source+license; sha256s + `ingest.json` receipt.
- `datastore publish <name> <version> --filter ...` — scan run records, materialize matches as JSONL shards under `datasets/{name}/v{x.y.z}/` + `manifest.yaml` (filter text, sources, license, row_count, checksums, created_by, pipeline_sha); refuses to overwrite an existing version. Manifests also committed to git under `manifests/`.
- `datastore fetch <name>@<version> <dest>` — alias of `get` for datasets; verify checksums; print manifest summary; idempotent.

## 7. Consumption patterns (README, no components)

- **Training / post-training:** `datastore fetch <name>@<ver> ./data/` on the training box → point the dataloader at the directory. JSONL until something outgrows local disk.
- **Analysis / blog numbers:** DuckDB over S3: `read_json('s3://octo-ai/runs/*/**/record.json')`. No server.
- **Future RAG:** derived index later over `raw/` + `runs/`; guaranteed possible because sources are immutable and traceable.

## 8. Backup & DR plan (written now, built later)

Trigger: first real data we'd regret losing. Design: nightly CronJob `mc mirror --remove=false octo-ai <BACKUP_TARGET>` to a target OFF this cluster/disk; tested restore runbook. Until then the added disk is the only copy — acknowledged.

## 9. Implementation phases (Claude Code)

Each phase = reviewable PR with tests + README section.
1. **Repo + deploy skeleton:** kustomize (MinIO, Route, NetworkPolicy, secrets template, setup Job), `make deploy/undeploy`, StorageClass check.
2. **CLI push/get:** multipart, retries; unit tests (minio container), integration test against cluster.
3. **Extractor:** naming-convention parsers (Luigi's trajectory format first), record building/validation, quarantine path, idempotency; golden-file tests from a sample tarball Luigi provides.
4. **ingest + publish + fetch:** manifest generation/validation, checksum verify, refuse-overwrite.
5. **Adoption:** team pushes real data; seed corpus ingests (CUDA docs, CuTe DSL, Helion — licenses recorded); DuckDB examples in README.

## 10. Graduation triggers

- Backup trigger (§8) → build CronJob + tested restore.
- Trajectory training wants per-step units → re-run extractor over archived originals with per-step logic.
- Need RAG → derived index (pgvector or similar) fed from existing data.
- Query needs outgrow DuckDB → Postgres runs table backfilled from record.json files.
- Second team adopts → per-team buckets/policies; consider central API relay.
- Multi-node cluster with disks → ODF/Ceph replaces single-node MinIO.

## 11. Trade-offs accepted

- No database → no referential integrity; schema + extractor validation are the contract.
- Inferred metadata is weaker than declared metadata; `inferred_fields` keeps us honest, and hard requirements survive only where retroactive answers are impossible (licenses on corpus data).
- Extractor is a new standing component (a CronJob, not an operator) — the price of zero-friction producers.
- Anyone with an S3 key can technically write junk to `incoming/`; extractor quarantines rather than trusts.
- Single disk, no backups yet → loss window until §8 trigger; explicitly accepted.
- No search in v1 → keyword-grep over fetched files in the meantime.
