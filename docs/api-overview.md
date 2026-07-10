# API Overview

All payloads below are illustrative. Model names, IDs and namespaces are placeholders.

## Pipelines

### `GET /pipelines/by-name/parameters?task_type=object-detection`

Returns the input schema of the latest version of the named pipeline, read from its compiled spec.

```json
{
  "status": "success",
  "task_type": "object-detection",
  "pipeline_version": "2026-05-02-a1",
  "parameters": {
    "dataset_path":       { "type": "STRING",       "required": true  },
    "base_model_version": { "type": "STRING",       "required": false },
    "model_version":      { "type": "STRING",       "required": true  },
    "epochs":             { "type": "NUMBER_INTEGER","required": false }
  }
}
```

## Runs

### `POST /runs/by-name/create`

```json
{
  "task_type": "object-detection",
  "run_name": "od-train-v4",
  "params": {
    "dataset_path": "s3://datasets/object-detection/batch-17.json",
    "base_model_version": "v3",
    "model_version": "v4"
  }
}
```

```json
{
  "status": "success",
  "run_id": "4c1e…",
  "pipeline_id": "9b0a…",
  "version_id": "e72f…",
  "experiment_name": "object-detection",
  "params_used": { "...": "..." }
}
```

If a required parameter is missing, the gateway answers `400` before anything reaches the cluster:

```json
{ "detail": "Missing required parameter: 'model_version' (type: STRING)" }
```

### `GET /runs/{run_id}/detailed-status`

```json
{
  "status": "success",
  "run_id": "4c1e…",
  "pipeline_state": "RUNNING",
  "pipeline_error": null,
  "current_components": ["train-model", "export-onnx"],
  "failed_components": [],
  "components": [
    { "component_name": "load-dataset", "state": "Succeeded", "start_time": "…", "end_time": "…" },
    { "component_name": "train-model",  "state": "Running",   "start_time": "…", "end_time": null }
  ]
}
```

- `current_components` can hold more than one entry when steps run in parallel.
- `failed_components` lists every failed task, so alerts can name the step that broke.
- `pipeline_error` carries run-level errors such as scheduling or RPC failures.

### `POST /runs/{run_id}/terminate`

Stops the run and cleans it up.

## Model Registry

### `GET /model-registry/{task_type}/versions`

```json
{
  "task_type": "object-detection",
  "versions": [
    { "name": "v3", "id": "12", "state": "LIVE", "created": "…", "metrics": { "mAP50": 0.91 } },
    { "name": "v4", "id": "15", "state": "LIVE", "created": "…", "metrics": { "mAP50": 0.93 } }
  ]
}
```

### `GET /model-registry/{task_type}/{version}`

This returns the version's custom properties: hyperparameters, metrics, the parent/base version, the
artifact URIs (raw weights, exported format, serving-ready model store) and the dataset lineage.

## Serving

### `POST /serving-models/deploy`

```json
{ "task_type": "object-detection", "version": "v4", "environment": "staging" }
```

If you leave out `version`, the latest version is deployed.

```json
{
  "status": "success",
  "deployment": {
    "id": "53",
    "name": "object-detection-svc",
    "model": "object-detection",
    "version": "v4",
    "namespace": "ml-staging",
    "environment": "staging",
    "runtime": "triton-onnx",
    "model_store": "s3://model-store/object-detection/v4/triton"
  }
}
```

The InferenceService this creates is shown in [`examples/inference-service.generated.yaml`](../examples/inference-service.generated.yaml).

### `GET /serving-models/deployments/{name}/status?namespace=ml-staging&timeout=120`

This polls the InferenceService `Ready` condition. With `timeout=0` it checks once and returns.

```json
{ "ready": true, "url": "http://object-detection-svc.ml-staging.example.com", "message": "Service is ready" }
```

### `GET /serving-models/deployments?env=production&show_all=false`

This lists the deployments recorded in the Model Registry. With `show_all=true`, `UNDEPLOYED` history is included too.

### `DELETE /serving-models/deployments/{id}`

This deletes the KServe resource and marks the registry record `UNDEPLOYED`.
