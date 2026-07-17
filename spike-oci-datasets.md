# Spike: Publish datasets as OCI artifacts (Quay + ORAS)

**Timebox:** half a day · **Owner:** Ben · **Status:** not started
**Origin:** Maryam's suggestion (Slack, 2026-07-16): k8s image volumes + ORAS + podman artifact.

## Question to answer

Should `datastore publish` / `fetch` target Quay (OCI artifacts) instead of the
`datasets/` prefix in MinIO?

**Hypothesis:** yes for published datasets only. A registry natively provides what
the current spec hand-rolls — immutability (content digests), versioning (tags),
integrity (digest verification) — and Quay is already operated in-house. MinIO
remains the working store (`incoming/`, `runs/`, `raw/`); this spike does NOT
reconsider that.

## Success criteria

The spike passes if 1 and 2 succeed. 3 is a bonus (in-cluster convenience), not a gate.

1. **Push:** `oras push` a sample dataset (2–3 JSONL shards + `manifest.yaml`) to a
   repo in our Quay org using a robot account, with the manifest attached as OCI
   annotations and a custom artifact media type
   (e.g. `application/vnd.octo.dataset.v1+json`).
   - Verify Quay UI renders the artifact repo sanely (or at least doesn't choke).
   - Note any org policy / artifact-type toggles needed.
2. **Pull:** on a bare-metal lab box, `oras pull` the artifact **by digest** and
   confirm contents match (sha256 of shards). This is the future `datastore fetch`.
3. **Mount (bonus):** on the team OpenShift cluster, mount the artifact into a test
   pod via an image volume and read the JSONL from the mounted path.
   - First check feasibility: `oc version` → k8s level; image volumes are stable
     upstream in v1.36, earlier levels may be feature-gated or absent. If the
     cluster can't do it, record that and move on — do not burn the timebox here.

## Non-goals

- No changes to datalib or the spec during the spike.
- No evaluation of storing `runs/` or `raw/` in the registry (known bad fit:
  high-churn small objects, prefix listing, DuckDB-in-place queries).
- No decision on models/kernels distribution (GKM overlap) — note thoughts if they
  arise, park them.

## Artifacts to produce

- `spike-notes.md` in the repo: commands used, Quay settings touched, timings for
  push/pull of the sample, and a PASS/FAIL per criterion.
- If PASS → spec v5 delta (pre-agreed shape):
  - `publish` = materialize shards → `oras push` to Quay, manifest as annotations;
    git manifest copy unchanged.
  - `fetch` = `oras pull` + digest verification.
  - Delete `datasets/` prefix from bucket layout; update IAM section.
  - Document image-volume mount as the in-cluster consumption path (if criterion 3
    passed); ORAS pull as the universal path.
  - Dataset access control = Quay RBAC (replaces MinIO dataset policies).
- If FAIL → one-paragraph note in spec §Graduation triggers ("revisit OCI datasets
  when …") and current design stands.

## Open questions to note during the spike

- Robot account permission model for artifact repos (push vs admin)?
- Quay retention/GC: are untagged-but-referenced digests safe from pruning?
- Max layer/blob size limits vs our largest plausible dataset shard?
- Does `podman artifact pull` offer anything over ORAS for lab boxes?
