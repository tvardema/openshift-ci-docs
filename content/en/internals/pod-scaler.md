---
title: "Pod Scaler"
description: How pod-scaler recommends and mutates CPU/memory on CI workloads, what repo owners need to know, and how test-platform operators run admission on build farms.
---

Pod-scaler is a mutating admission webhook on **build farm clusters** (`build01`–`build13`). It reads historical CPU and memory usage from Prometheus (cached in GCS), recommends resources for recurring workloads, and may change requests and limits when a pod is **created**.

- Source: [`openshift/ci-tools/cmd/pod-scaler`](https://github.com/openshift/ci-tools/tree/master/cmd/pod-scaler)
- Build-farm admission manifest: [`openshift/release/clusters/build-clusters/build-shared/pod-scaler/`](https://github.com/openshift/release/tree/main/clusters/build-clusters/build-shared/pod-scaler)
- Producer and UI on `app.ci`: [`clusters/app.ci/pod-scaler/`](https://github.com/openshift/release/tree/main/clusters/app.ci/pod-scaler)

## What changed recently

These landed on `main` in July 2026 and are worth knowing before you debug a job:

| Change | Repo | What it means for you |
| ------ | ---- | --------------------- |
| **Cache hardening** | ci-tools | Bad GCS cache entries are repaired on reload. Admission skips unusable recommendations (zero CPU, below 10m CPU, below 1Mi memory, or more than 10× your configured value). Skip flags let operators exempt workload types from authoritative **decreases**. |
| **Flag cleanup** | release | Removed `*-dry-run` CLI flags (they no longer exist in the binary). Dry-run is now `--authoritative-*=false`. Build farms run authoritative **decrease dry-run** today — logs what would change, does not lower resources yet. |
| **Memory floor follow-up** (merging soon) | ci-tools | Producer drops memory samples below 1Mi before they enter histograms. Admission raises sub-1Mi memory to 1Mi right before stamping historical values onto a pod. New flag `--recommendation-buffer-percent` (default **20**, same as the old fixed 1.2× multiplier). |

If admission pods crash-loop with `flag provided but not defined: -authoritative-cpu-request-dry-run`, the cluster still has the **old** release args. Sync to release `main` (or ArgoCD hard-sync `build-shared-buildNN-pod-scaler`) before chasing job failures.

---

## For repo and job owners

You set CPU and memory in `.ci-operator.yaml` (test steps, image builds, registry steps). On build clusters, pod-scaler may **raise** those values at pod creation when history shows higher usage. **Decreases** are in dry-run on build farms right now (see above), so you should not see limits drop from authoritative mode until operators flip the apply flags to `true`.

### What you will see

| Situation | What pod-scaler does |
| --------- | -------------------- |
| First run / no history | Leaves your configured resources unchanged. |
| History shows higher usage | May **increase** requests (and limits when limit mutation is on), up to cluster caps. |
| History shows lower usage | With authoritative apply **off** (current build-farm default): **logs** a would-be decrease; pod keeps your YAML values. |
| Pod marked measured (`pod-scaler.openshift.io/measured=true`) | Extra CPU headroom; authoritative decreases skipped on that pod. |
| Job has `disable_pod_scaler: true` | **No mutation** — your YAML values are kept. |

Pod-scaler does **not** resize running pods. Retrigger the job or start a new build to pick up changes.

### When to edit your config

1. **Set explicit resources** for memory-sensitive or bursty work (image builds, heavy compiles). Defaults plus history are not a substitute for knowing your job.
2. **Use `disable_pod_scaler: true`** when one test target must keep fixed resources regardless of history.
3. **Skip flags are not per-repo.** You cannot set them in your repository; ask test-platform if you think a workload class needs cluster-wide decrease exemptions.

### Per-job opt-out (`disable_pod_scaler`)

Set on the test `as:` name in `.ci-operator.yaml`:

**Image build (`*-images` jobs):**

{{< highlight yaml >}}
tests:
- as: images
  disable_pod_scaler: true
  steps:
    test:
    - as: src
      commands: # unchanged
      from: src
      resources:
        requests:
          cpu: 100m
          memory: 4Gi
        limits:
          memory: 8Gi
{{< / highlight >}}

**Multi-step test:**

{{< highlight yaml >}}
tests:
- as: e2e-aws
  disable_pod_scaler: true
  steps:
    workflow:
    - as: openshift-e2e-test
      ...
{{< / highlight >}}

ci-operator sets `ci-workload-autoscaler.openshift.io/scale: "false"` on generated pods and OpenShift `Build` objects for that target.

### Troubleshooting your jobs

#### `compile: signal: killed` or OOM after a pod-scaler image rollout

Image builds often **burst** above average history. Build farms already use **peak** usage basis for decrease math (when decreases are enabled), so limits track spikes better than p80 alone.

If a job still OOMs:

1. Raise `resources.limits.memory` (and requests if needed) in `.ci-operator.yaml`.
2. Set `disable_pod_scaler: true` on that test target if the job must stay fixed.
3. Check whether admission is healthy on the build cluster (see [Verify deployment](#verify-deployment)).

#### Resources look stuck high (e.g. 20Gi memory)

Admission caps recommendations at **`--memory-cap=20Gi`** and **`--cpu-cap=10`**. A pod stamped above the cap before a fix keeps that value until the **next** pod create; admission also clamps corrupt stamped values when it can derive a valid recommendation.

#### Resources did not change after you edited YAML

Pod-scaler keys on workload metadata (org, repo, branch, variant, target, container), not only the Prow job name. Confirm the pod is in a namespace with `ci.openshift.io/scale-pods=true`, is not opted out, and is a **new** pod.

#### Alert: `pod-scaler-admission-resource-warning`

Fires when a workload used roughly **10×** declared CPU or memory. That usually means undersized YAML, a leak, or bad history — not “add cluster capacity.” See the [pod-scaler admission SOP](https://github.com/openshift/release/blob/master/docs/dptp-triage-sop/pod-scaler-admission.md) on `openshift/release`.

---

## How pod-scaler works

### Components

| Component | Where | Role |
| --------- | ----- | ---- |
| **Producer** | `app.ci` | Queries Prometheus, builds histograms, writes GCS cache (`origin-ci-resource-usage-data-v2`). |
| **Admission consumer** | `build01`–`build13` | Loads cache, serves `/pods` webhook, mutates resources at pod **create**. |
| **UI** | `app.ci` | Inspect recommendations (when deployed). |

Admission runs as `pod-scaler-consumer-admission` in namespace `ci`. The webhook matches namespaces labeled `ci.openshift.io/scale-pods=true`.

### Recommendation pipeline

1. Producer records container CPU/memory samples into histograms per workload fingerprint.
2. Consumer merges histograms into recommendations keyed by workload metadata (org, repo, branch, variant, target, container, measured vs unmeasured, etc.).
3. At admission, pod-scaler loads the best matching recommendation (max of measured and unmeasured when both exist).
4. Recommendations use **p80** of merged history, then a buffer multiplier on top (hard-coded 1.2× today; upcoming releases expose this as `--recommendation-buffer-percent`, default 20).
5. Peak limits for digest use histogram max, capped before admission applies them.

### Default behavior (increases)

- Admission **increases** configured requests/limits when the recommendation is higher (unless you opted out).
- **Measured pods** (~`--percentage-measured`, **10%** on build farms) get extra CPU from `--measured-pod-cpu-increase` (50%). Authoritative **decreases** are skipped on measured pods only.
- **Memory limits** — with `--mutate-resource-limits=true`, limits are kept and forced to at least **2×** memory request after mutation.

### Safety rails

| Rail | Value | Effect |
| ---- | ----- | ------ |
| Cluster memory cap | `20Gi` | Requests and limits cannot exceed cap. |
| Cluster CPU cap | `10` cores | Requests and limits cannot exceed cap. |
| Increase cap | 10× configured baseline | Larger recommendations are capped before cluster cap. |
| Minimum usable memory | **1Mi** | Recommendations below this are skipped; corrupt stamps are raised on admit (a second floor at the stamp point lands in the next image). |
| Minimum usable CPU | **10m** | Recommendations below this are skipped. |
| Corrupt stamped values | — | Values above cluster cap or above 10× recommendation are clamped down. |

### Cache load and repair

Consumer reloads GCS cache periodically (hourly tick; restart admission to force an immediate reload after incidents). On load, pod-scaler **repairs** corrupt cache entries (`pkg/pod-scaler/cache_sanitize.go`):

- Drops orphan histogram references.
- Removes histograms with impossible spikes (above cluster cap or above 10× baseline).
- Failed reload **keeps** the previous in-memory cache when one exists.

**Already-stamped** resources on old pods are unchanged until the next create.

---

## Authoritative decrease mode

When `--authoritative-*-request` or `--authoritative-*-limit` is **`true`**, admission may **lower** requests or limits when scaled usage is below what is configured.

When the flag is **`false`**, admission **logs** what it would decrease (`authoritative_*_dry_run` events) but does **not** mutate. There are no separate `*-dry-run` CLI flags anymore — set the authoritative apply flag to `false` instead.

**Build farms today:** all four authoritative apply flags are **`false`** (decrease dry-run). Max-reduction and usage-basis flags are already wired for when operators enable apply.

Each decrease (when enabled) is bounded by:

| Guard | Behavior |
| ----- | -------- |
| Max reduction per admission | `*-max-reduction-percent` (**0.50** on build farms = at most 50% drop per create). |
| Minimum change | Decreases smaller than ~5% are skipped. |
| Measured pods | Never decreased. |
| Escalation | After OOM or CPU throttle, failure escalation can raise resources; further decreases suppressed at higher levels. |
| Floors | CPU not below **10m**; memory not below **1Mi**. |

Four independent toggles: CPU/memory × request/limit.

### Usage basis for decreases

Flag: `--pod-scaler-authoritative-decrease-usage-basis`

| Value | Meaning |
| ----- | ------- |
| `p80` | Binary default. Steady-state request from histograms. |
| `peak` | Histogram **max** (burst) before the buffer multiplier. |
| `max` | Alias for `peak`. |

Build farms run **`peak`**. Peak basis uses **max(peak, request)** so a stale low peak cannot shrink below the p80 request recommendation.

### Workload skip for decreases

Four optional flags; comma-separated lists; **empty = no skip**.

| Flag | Skips |
| ---- | ----- |
| `--pod-scaler-skip-workload-type-limit-decrease` | Limit decreases for listed **workload types** |
| `--pod-scaler-skip-workload-type-request-decrease` | Request decreases for listed types |
| `--pod-scaler-skip-workload-class-limit-decrease` | Limit decreases for listed **`ci-workload`** classes |
| `--pod-scaler-skip-workload-class-request-decrease` | Request decreases for listed classes |

Skip affects **decreases only**; increases still apply. None of these are set on build farms in release `main` today — they are available when operators need them (for example `build` image builds).

### Workload types (skip matching)

Detection order (first match wins):

| Type | When |
| ---- | ---- |
| `build` | Pod has `openshift.io/build.name` (image builds, `*-images` jobs). |
| `prowjob` | Pod has `prow.k8s.io/job`. |
| `step` | Pod has `ci.openshift.io/metadata.step`. |
| `undefined` | No match. |

**Class** comes from `ci-workload` (scheduling pool): `builds`, `tests`, `longtests`, `prowjobs`, etc.

---

## Operator reference

### Release manifest (build farms)

Args live in [`pod-scaler-admission.yaml`](https://github.com/openshift/release/blob/main/clusters/build-clusters/build-shared/pod-scaler/pod-scaler-admission.yaml). Deployed by ArgoCD ApplicationSet `appset-cluster-build-shared` on `core-ci` (`build-shared-buildNN-pod-scaler`).

Current `main` excerpt:

{{< highlight yaml >}}
- --cpu-cap=10
- --memory-cap=20Gi
- --mutate-resource-limits=true
- --percentage-measured=10
- --measured-pod-cpu-increase=50
- --authoritative-cpu-request=false
- --authoritative-cpu-limit=false
- --authoritative-memory-request=false
- --authoritative-memory-limit=false
- --authoritative-cpu-request-max-reduction-percent=0.50
- --authoritative-cpu-limit-max-reduction-percent=0.50
- --authoritative-memory-request-max-reduction-percent=0.50
- --authoritative-memory-limit-max-reduction-percent=0.50
- --pod-scaler-authoritative-decrease-usage-basis=peak
{{< / highlight >}}

To **apply** decreases on one field, set that `--authoritative-*` flag to **`true`**. Roll one field at a time if you are unsure.

Image tag: `quay-proxy.ci.openshift.org/openshift/ci:ci_pod-scaler_latest` (keel `@hourly` on build farms).

### Flag reference

#### Runtime and caps

| Flag | Build farms |
| ---- | ----------- |
| `--mutate-resource-limits` | `true` |
| `--cpu-cap` / `--memory-cap` | `10` / `20Gi` |
| `--percentage-measured` | `10` |
| `--measured-pod-cpu-increase` | `50` |
| `--recommendation-buffer-percent` | `20` (binary default; may appear explicitly in release manifest after next image) |

#### Authoritative apply (four fields)

| Flag | Binary default | Build farms (`main`) |
| ---- | -------------- | -------------------- |
| `--authoritative-cpu-request` | `false` | `false` (dry-run) |
| `--authoritative-cpu-limit` | `false` | `false` (dry-run) |
| `--authoritative-memory-request` | `false` | `false` (dry-run) |
| `--authoritative-memory-limit` | `false` | `false` (dry-run) |

#### Max reduction (four fields)

All set to **`0.50`** on build farms.

#### Usage basis

`--pod-scaler-authoritative-decrease-usage-basis=peak` on build farms.

#### Deprecated aliases (still accepted)

| Deprecated | Replacement |
| ---------- | ----------- |
| `--authoritative-cpu` | `--authoritative-cpu-request` + `--authoritative-cpu-limit` |
| `--authoritative-memory` | `--authoritative-memory-request` + `--authoritative-memory-limit` |
| `--authoritative-cpu-max-reduction-percent` | `--authoritative-cpu-limit-max-reduction-percent` |
| `--authoritative-memory-max-reduction-percent` | `--authoritative-memory-limit-max-reduction-percent` |

Removed: `*-dry-run` flags. Use `=false` on the apply flag.

### Verify deployment

On a build cluster:

```bash
CTX=build01   # or build02 … build13

oc --context "$CTX" -n ci get deploy pod-scaler-consumer-admission \
  -o jsonpath='ready={.status.readyReplicas}/{.spec.replicas}{"\n"}'

oc --context "$CTX" -n ci get deploy pod-scaler-consumer-admission \
  -o jsonpath='{.spec.template.spec.containers[0].args}' | jq .

oc --context "$CTX" -n ci get pods -l app=pod-scaler-consumer-admission \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.containerStatuses[0].ready}{" "}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

Expect **2/2** ready, image digest matching `ci_pod-scaler_latest`, and **no** `*-dry-run` in args.

Producer on `app.ci`:

```bash
oc --context app.ci -n ci get deploy pod-scaler-producer pod-scaler-ui
```

Restart admission after cache incidents:

```bash
oc --context "$CTX" -n ci rollout restart deploy/pod-scaler-consumer-admission
```

### Operator troubleshooting

#### Admission CrashLoopBackOff after image rollout

Check logs for unknown flags (old release args vs new binary). Fix: sync ArgoCD app to release `main`. If deploy args are correct but stale ReplicaSets remain, delete crashing pods or force-sync the Application.

#### ArgoCD shows Synced but args are wrong

`ServerSideApply` + `ApplyOutOfSyncOnly` can leave stale `args` field managers. Hard-sync the `build-shared-buildNN-pod-scaler` app on `core-ci`, or server-side apply the manifest with `--force-conflicts`.

#### Absurd CPU in logs (e.g. `-9223372036854775808m`)

Bad histogram quantile (NaN/Inf). Repair on cache load should drop it; roll admission and confirm producer is healthy.

#### Suspect corrupt cache in GCS

Consumer repair runs on every load. After producer fixes, restart admission. Operational cache prune scripts live in `openshift/ci-tools/hack/` for incident response.

#### Admission logs worth searching

| Log / event | Meaning |
| ----------- | ------- |
| `recommendation_increase_capped` | 10× or cluster cap applied on increase. |
| `configured_resource_clamped` | Corrupt stamped value clamped. |
| `memory_floor_applied` | Sub-1Mi memory raised before stamp. |
| `authoritative_*_decreased` | Decrease applied (apply flag `true`). |
| `authoritative_*_dry_run` | Would decrease; apply flag `false` (current build-farm default). |
| `skipping recommendation with no usable histogram data` | No safe recommendation for that container. |
