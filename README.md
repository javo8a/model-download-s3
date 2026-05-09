# model-storage

Small OpenShift helpers to **copy model weights from S3 (or S3-compatible storage) into an existing PVC**, using the same pattern as the Open Data Hub KServe PVC init Job: a one-shot `batch/v1` `Job` that runs the **KServe storage initializer** image with a source URI and a destination directory on the volume.

Upstream reference: [ODH KServe `pvc-init` sample `job.yaml`](https://github.com/opendatahub-io/kserve/blob/odh-v3.3/docs/samples/storage/pvc-init/job.yaml) (that example uses `hf://…`; this repo uses `s3://…`).

## Contents

| Path | Purpose |
|------|---------|
| [`openshift/model-sync-from-s3-template.yaml`](openshift/model-sync-from-s3-template.yaml) | OpenShift `Template`: renders a `Job` that syncs `S3_URI` → `MOUNT_PATH` on the PVC |
| [`openshift/sample.env`](openshift/sample.env) | Example parameter file for `oc process --param-file` |

## Prerequisites

- OpenShift CLI (`oc`) and a namespace where you can create `Job`s and use your PVC.
- A **PersistentVolumeClaim** that already exists (`PVC_CLAIM_NAME`); the Job only writes into it.
- A **ServiceAccount** that is allowed to run the pod and mount that PVC (`SERVICE_ACCOUNT_NAME`). Adjust RBAC and SCCs to match your cluster policies.
- A **Secret** in the **same namespace** as the Job, referenced by `S3_CREDENTIALS_SECRET`, with credentials the initializer expects for S3:

  - Required: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
  - Recommended: `AWS_DEFAULT_REGION`
  - Optional (MinIO / on-cluster / custom endpoint): `AWS_ENDPOINT_URL`

## Quick start

1. Copy the sample parameters and edit for your model and cluster:

   ```bash
   cp openshift/sample.env openshift/my-model.env
   ```

   Set at least `JOB_NAME`, `PVC_CLAIM_NAME`, `SERVICE_ACCOUNT_NAME`, `S3_CREDENTIALS_SECRET`, and `S3_URI`.

2. Create the credentials secret (example):

   ```bash
   oc create secret generic my-model-s3-credentials \
     --from-literal=AWS_ACCESS_KEY_ID='<key>' \
     --from-literal=AWS_SECRET_ACCESS_KEY='<secret>' \
     --from-literal=AWS_DEFAULT_REGION='us-east-1'
   ```

   Add `--from-literal=AWS_ENDPOINT_URL='https://…'` if you use an S3-compatible endpoint.

3. Render the template and apply it (replace `-n` as needed):

   ```bash
   oc process --local -f openshift/model-sync-from-s3-template.yaml \
     --param-file=openshift/my-model.env \
     -o yaml | oc apply -n my-project -f -
   ```

   Use **`--local`** so `oc process` does not need a live API connection to read the `Template`; omit `--local` if you prefer the server to expand the template.

4. Watch the Job and logs:

   ```bash
   oc get job,pods -l app.kubernetes.io/name=model-storage-sync
   oc logs -f job/<JOB_NAME>
   ```

## Parameters

All parameters can be set via `-p NAME=value` or a **`--param-file`** where each line is `NAME=value` (no spaces around `=`).

| Parameter | Description |
|-----------|-------------|
| `JOB_NAME` | Name of the `Job` (DNS-1123 subdomain) |
| `PVC_CLAIM_NAME` | Existing PVC to populate |
| `S3_URI` | Source URI, e.g. `s3://my-bucket/path/to/model/` |
| `MOUNT_PATH` | Directory on the PVC inside the container (default `/mnt/models`) |
| `SERVICE_ACCOUNT_NAME` | Pod `serviceAccountName` |
| `S3_CREDENTIALS_SECRET` | Secret name for `envFrom` (AWS-style keys) |
| `STORAGE_INITIALIZER_IMAGE` | Container image (default ODH KServe storage initializer) |
| `HF_HUB_DISABLE_TELEMETRY` | Extra env; default `1` |
| `CPU_REQUEST`, `MEMORY_REQUEST`, `CPU_LIMIT`, `MEMORY_LIMIT` | Pod resources |

Defaults match the sample Job’s shape (single completion, initializer args `[source, dest]`). Tune CPU and memory for your model size and network.

## Shell substitution

If you prefer exporting variables instead of a param file:

```bash
set -a && source openshift/my-model.env && set +a
oc process --local -f openshift/model-sync-from-s3-template.yaml \
  -p JOB_NAME="$JOB_NAME" \
  -p PVC_CLAIM_NAME="$PVC_CLAIM_NAME" \
  -p SERVICE_ACCOUNT_NAME="$SERVICE_ACCOUNT_NAME" \
  -p S3_CREDENTIALS_SECRET="$S3_CREDENTIALS_SECRET" \
  -p S3_URI="$S3_URI" \
  -p MOUNT_PATH="$MOUNT_PATH" \
  -o yaml | oc apply -f -
```

## Notes

- **Idempotency:** Re-running the same Job name after success may require deleting the old `Job` or using a new `JOB_NAME`; Kubernetes treats `Job` names as unique.
- **Partial downloads:** If a run fails, inspect logs and PVC contents; you may need to clean stale files or bump `JOB_NAME` for a fresh run depending on how the initializer handles existing paths.
- **Image version:** Pin `STORAGE_INITIALIZER_IMAGE` to a digest or a tested tag in production rather than a rolling `latest` tag.
