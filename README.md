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
  - Optional S3 endpoint / TLS tuning: either set template parameters `S3_USE_HTTPS`, `S3_ENDPOINT`, `AWS_ENDPOINT_URL`, and `S3_VERIFY_SSL` (see [`openshift/model-sync-from-s3-template.yaml`](openshift/model-sync-from-s3-template.yaml)), **or** put `AWS_ENDPOINT_URL` (and any other keys your client reads) in the secret. Keys listed in both places use the **template / explicit env** value because it overrides `envFrom`.

For **public AWS S3**, leave those four parameters empty (omit them from your param file so the template defaults apply).

Explicit container `env` overrides `envFrom`, so an empty template value for `AWS_ENDPOINT_URL` still **replaces** `AWS_ENDPOINT_URL` from the secret. If you store the endpoint in the secret, set template parameter `AWS_ENDPOINT_URL` to the **same** URL (or edit the rendered manifest to drop that env entry).

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

   **`S3_URI` and YAML quotes:** The template lists args as `"${S3_URI}"`, but YAML only uses those quotes while parsing the template. After substitution, the value is the plain string `s3://…`; `oc process -o yaml` then prints scalars in the minimal style (often **without** quotes). That output is still valid YAML and is what the API stores—same as a quoted string. If you prefer JSON (where strings are always quoted), run `oc process … -o json | oc apply -f -`.

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
| `S3_USE_HTTPS` | Optional; empty default for public AWS S3 |
| `S3_ENDPOINT` | Optional S3-compatible API host (no `https://`); empty default for AWS |
| `AWS_ENDPOINT_URL` | Optional full URL with scheme (e.g. MinIO); empty default for AWS |
| `S3_VERIFY_SSL` | Optional TLS verify flag; empty default |
| `MEMORY_REQUEST`, `MEMORY_LIMIT` | Pod memory request / limit |

Defaults match the sample Job’s shape (single completion, initializer args `[source, dest]`). Tune memory for your model size and network.

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
