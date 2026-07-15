# SPEC: Team AI Datastore

Status: v3 (final for v1 build) · Owner: Accelerating AI team · Target: team OpenShift cluster, lab network (VPN)

## 1. What this is

The simplest trustworthy data store for the Accelerating AI boulder: KernelSmith / perf-eval run data in, curated training and post-training datasets out. **One MinIO pod + a Python client library (datalib) + strict conventions.** No database, no API service, no operators (app components run as plain k8s primitives).

Decisions made and locked (2026-07-15 planning session):
- Kubernetes primitives only; installable into a single namespace with no cluster-admin beyond StorageClass availability.
- Use cases in scope: **training data** and **post-training data** (verified run records as reward signal). Out of scope for v1: RAG/retrieval, SQL analysis service. Analysis is served by DuckDB reading JSONL on S3 directly.
- Producers run **anywhere** (bare-metal benchmark machines, laptops, in-cluster); access is via the MinIO S3 Route over the lab network.
- One front door: **datalib**. No NFS mounts, no ad-hoc S3 writes (social contract at team size).
- Secrets: hand-created Kubernetes Secrets; `secrets.template.yaml` in repo documents required keys. Vault later.
- Backups: **written plan, not built** (§7). Trigger to build: the first real run data we'd be sad to lose.
- Embedding worker: removed. Search: none in v1.
- System may be throwaway; **the data and conventions are not** — records must outlive the system.

## 2. Deployment (OpenShift, namespace `octo-datastore`)

Kustomize manifests in `deploy/base` + `deploy/overlays/lab`:
- **MinIO**: StatefulSet (1 replica), PVC on the cluster StorageClass (LVMS on the added disk if no default exists — check `oc get storageclass` first; LVMS is one-time infra plumbing, not part of the app). Bucket versioning ON.
- Service (ClusterIP) + **Route** (edge or reencrypt TLS) for the S3 API. No console Route in v1.
- NetworkPolicy: default-deny ingress; allow openshift-ingress → MinIO.
- SCC: default restricted-v2; no exceptions.
- Secrets: `minio-root` (admin) + one access key per producer (`kernelsmith`, `perfeval`, `human-<name>`), created by hand from the template. MinIO IAM policies as code (Job running `mc admin`): producers can write only their `runs/{pipeline}/*`; everyone reads `datasets/*` and `raw/*`; only admin writes `datasets/*` finalization and can delete object versions (immutability enforcement: deny `s3:DeleteObjectVersion` to non-admin).
- Setup Job: creates bucket, lifecycle rules, IAM policies idempotently.

## 3. Bucket layout (`octo-ai`, versioning ON)

```
raw/{source}/{yyyy-mm-dd}/...      # manual corpus uploads; immutable
runs/{pipeline}/{run_id}/          # record.json + artifacts; written once
datasets/{name}/v{semver}/         # published JSONL shards + manifest.yaml; immutable
scratch/{user}/                    # 30-day lifecycle expiry
```

- `record.json` is written **last** in a run prefix → its presence means the run is complete (crash-safe).
- Lifecycle: expire `scratch/` after 30d; expire noncurrent versions after 90d except under `raw/`.

## 4. The run record (most important artifact in the repo)

Pydantic model in datalib, versioned via `schema_version` field in every record. v1 fields:

```
schema_version: "1"
run_id: uuid            pipeline: str          created_at: iso8601
spec: {op, shapes, dtypes, target_arch, constraints...}
status: compile_fail | incorrect | correct | error     # failures are kept — they are signal
model_id: str           parent_run: uuid|null  # refinement chains
correctness: {ref_impl, tolerance, max_abs_err, max_rel_err} | null
benchmark: {latency_us, throughput, vs_baseline, machine, env_sha} | null
profile_summary: {occupancy, mem_bw_pct, top_stalls} | null
artifacts: {kernel: s3key, compile_log: s3key, profile: s3key, ...}
license: str            created_by: str        pipeline_sha: str
```

Schema evolution: additive changes bump minor; breaking changes bump `schema_version` and datalib must keep reading all prior versions.

## 5. datalib (Python package + `datastore` CLI)

The only sanctioned access path. ~small, boto3 + pydantic + typer. Config: `DATASTORE_ENDPOINT`, `DATASTORE_ACCESS_KEY`, `DATASTORE_SECRET_KEY`.

- `record(pipeline, spec, status, results, artifacts)` — validate against schema, upload artifacts to `runs/{pipeline}/{run_id}/`, write `record.json` last.
- `ingest(path, source, license)` — refuses without source+license; sha256s; uploads to `raw/{source}/{date}/`; writes `ingest.json` receipt (who/when/what).
- `publish(name, version, filter)` — scan run records, apply filter, materialize matching records as JSONL shards under `datasets/{name}/v{x.y.z}/`, write `manifest.yaml` (name, version, filter text, sources, license, row_count, checksums, pipeline_sha, created_by); refuse to overwrite an existing version.
- `fetch(name@version, dest)` — download dataset prefix, verify all checksums against manifest, print manifest summary; idempotent.
- Multipart uploads, retry/backoff, works identically on-cluster and off-cluster.
- Manifests are ALSO committed to git under `manifests/{name}/v{x.y.z}.yaml`.

## 6. Consumption patterns (documented in README, no components)

- **Training / post-training:** `datastore fetch <name>@<ver> ./data/` on the training box → point the dataloader at the directory. JSONL until a dataset outgrows local disk (likely never at our scale).
- **Analysis / blog numbers:** DuckDB directly over S3: `read_json('s3://octo-ai/runs/kernelsmith/*/record.json')`. No server.
- **Future RAG:** derived index built later over `raw/` + `runs/`; guaranteed possible because sources are immutable and traceable. Not in v1.

## 7. Backup & DR plan (written now, built later)

Build trigger: first real KernelSmith/perf-eval data we'd regret losing. Design when triggered: CronJob running `mc mirror --remove=false octo-ai <BACKUP_TARGET>` nightly to a target OFF this cluster/disk; restore runbook with a tested restore. Until then: the added disk is the only copy — acknowledged.

## 8. Implementation phases (Claude Code)

Each phase = reviewable PR with tests + README section.
1. **Repo + deploy skeleton:** kustomize manifests (MinIO, Route, NetworkPolicy, secrets template, setup Job), `make deploy/undeploy`, StorageClass check documented.
2. **datalib core:** schema models, `record` + `ingest`, unit tests (moto/minio-container), integration test against the cluster.
3. **publish + fetch:** manifest generation/validation, checksum verification, refuse-overwrite semantics.
4. **Adoption:** wire KernelSmith + perf-eval to `record`; seed corpus ingests (CUDA docs, CuTe DSL, Helion — licenses recorded); DuckDB analysis examples in README.

## 9. Graduation triggers

- Backup trigger hit (§7) → build the CronJob + tested restore.
- Need RAG → add derived index (pgvector or similar) fed from existing data.
- Need rich queries/joins beyond DuckDB comfort → introduce Postgres runs table, backfilled from record.json files.
- Second team adopts → per-team buckets/policies; consider the API-relay pattern for central enforcement.
- Multi-node cluster with disks → ODF/Ceph replaces single-node MinIO (rclone migration).

## 10. Trade-offs accepted

- No database → no referential integrity; the pydantic schema + code review are the contract.
- datalib-only access is a social contract; S3 keys can technically bypass it (3-person team, acceptable).
- Single disk, no backups yet → data loss window until §7 trigger; explicitly accepted.
- No search in v1 → fuzzy retrieval waits; keyword-grep over fetched files in the meantime.
