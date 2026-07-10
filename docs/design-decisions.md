# Design Decisions & Lessons Learned

## 1. The Model Registry is the source of truth for "what is deployed where"

**Decision:** Every deploy and undeploy writes an `InferenceService` record (with `desiredState`) to the
Kubeflow Model Registry, linked to the exact model version and serving environment.

**Why:** The backend and UI need lineage: which dataset trained which version, and where that version
is running now. The registry already models `RegisteredModel → ModelVersion → InferenceService`, so the
gateway did not need its own database.

**Trade-off:** The registry stores the *desired* state, not the *actual* state. See lesson A below.

## 2. One business key (`task_type`) instead of platform IDs

**Decision:** Callers use only a human-readable `task_type`. The gateway resolves pipeline,
version, experiment and registry IDs on each request.

**Why:** Platform IDs change when a pipeline is re-uploaded or a cluster is rebuilt. The backend
contract should not change when that happens.

**Trade-off:** It costs a few extra lookups per request. That's fine for a control-plane API with low
traffic, and the naming convention has to be followed strictly.

## 3. Configuration over code for serving

**Decision:** Runtime, model format, service account and GPU memory per `task_type` live in a JSON ConfigMap.

**Why:** Most new model types only differ in runtime and resources. Keeping this in config made
"add a model" a config change and "move to a new cluster" a different ConfigMap.

## 4. Validate pipeline parameters at the edge

**Decision:** Before submitting a run, read the compiled pipeline spec's `inputDefinitions` and reject
requests that are missing required parameters.

**Why:** Without this, a bad request shows up minutes later as a failed pod inside the cluster. A `400`
straight away, naming the missing parameter, saved a lot of back-and-forth with the backend team.

## 5. The backend owns version names

**Before:** The training pipeline picked the next version itself by scanning object-store prefixes.

**Problem:** Versioning logic was split across systems. Reruns or partial failures left the meaning
of a version unclear, and the backend's view of versions could drift from the pipeline's.

**Decision:** The backend passes an explicit `model_version` (`v<number>`). The pipeline validates the
format, **fails if the version already exists**, and uses that exact name for artifact paths, manifests and
registry registration. It never auto-increments and never overwrites.

**Result:** Artifact paths and registry entries are predictable, version names never change once used,
and audits are simpler. Serving did not change, because it already deployed by version label.

## 6. GPU sharing through a queue scheduler

**Decision:** Predictors are created with a GPU-aware queue scheduler, a queue label and a per-model
GPU memory annotation.

**Why:** Many small vision models do not need a whole GPU. Fractional allocation let several models share
one card, while large models (for example Whisper on vLLM) got a bigger budget. Training and
inference used separate queues, so a training job could not starve serving.

## 7. Robust run status across KFP versions

**Decision:** The detailed-status endpoint tries three data sources in turn (KFP runtime summary, then KFP v2
task details, then the raw Argo Workflow) and normalizes them into one response.

**Why:** No single source covered every case, especially runs that failed before any task was recorded.
The UI needed a consistent "what is running / what failed" view no matter which source answered.

---

## Lessons learned / what I'd change

**A. Desired state ≠ actual state.**
If a model crash-loops and KServe gives up, the registry still says `DEPLOYED`. The fix I planned was
a reconcile step that compares the KServe `Ready` condition and restart counts with the registry
record, either on read or as a periodic job, and marks failed deployments.

**B. Deployment should be asynchronous.**
Deploy currently creates the resource and returns, and readiness is a separate polling call. A small
job queue with deployment status events would be a better fit for slow-starting large models.

**C. Move pipeline config and secrets into the gateway.**
Pipelines read some configuration from cluster objects themselves. Having the gateway inject environment-specific
config and secret references at run creation would make the same pipeline work in dev and prod
without changes.

**D. Tests against a real cluster.**
Most bugs were integration bugs: differences between SDK versions and changes in response shapes. A kind or
minikube-based integration test stage with KFP and KServe installed would have caught them earlier.

**E. AuthN/Z at the gateway.**
The gateway relied on network placement and cluster RBAC. A production version should add service-to-service
auth and per-environment authorization, for example so that only the release pipeline can deploy to production.
