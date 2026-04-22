# nrp-clickhouse GitOps Deployment (Argo CD + Kustomize)

This repo is a **deployment repo** (separate from the application source repo) for deploying:
- <https://github.com/djw8605/nrp-clickhouse>

It tracks the upstream Kubernetes base and applies a `dev` overlay.

## Layout

- `apps/nrp-clickhouse/overlays/dev/kustomization.yaml`
- `apps/nrp-clickhouse/overlays/dev/configmap.yaml`
- `apps/nrp-clickhouse/overlays/dev/patches.yaml`
- `apps/nrp-clickhouse/overlays/dev/sealed-secret.yaml`
- `argocd/application-my-app-dev.yaml`

## What This Deploys

The upstream repo deploys the NRP accounting ETL pipeline and MCP server resources (not ClickHouse):
- `CronJob` (`nrp-accounting-etl`)
- `Deployment` (`nrp-accounting-mcp`)
- `Service` (`nrp-accounting-mcp`)
- `Ingress` (`nrp-accounting-mcp`) at `https://nrp-accounting-mcp.nrp-nautilus.io`
- config and service account resources

ClickHouse must exist separately and be reachable from the cluster.
The ETL also queries `portal.nrp.ai` each run to keep namespace metadata current in ClickHouse.

## Required Configuration

1. Argo CD Application repo URL:
   - File: `argocd/application-my-app-dev.yaml`
   - Set `spec.source.repoURL` to this deployment repo.

2. Image and tag for pipeline:
   - File: `apps/nrp-clickhouse/overlays/dev/kustomization.yaml`
   - Update `images[].newName` and `images[].newTag`.

3. Non-sensitive runtime settings:
   - File: `apps/nrp-clickhouse/overlays/dev/configmap.yaml`
   - Set values for:
     - `PROMETHEUS_URL`
     - `PORTAL_RPC_URL`
     - `CLICKHOUSE_DATABASE`
     - `CLICKHOUSE_PORT`
     - `CLICKHOUSE_SECURE`
     - `MAX_QUERY_WORKERS`
     - `QUERY_STEP`
     - `RETRY_LIMIT`
     - `PROMETHEUS_TIMEOUT_SECONDS`
     - `PORTAL_TIMEOUT_SECONDS`
     - `CLICKHOUSE_WRITE_BATCH_SIZE`

4. Sensitive ClickHouse credentials:
   - File: `apps/nrp-clickhouse/overlays/dev/sealed-secret.yaml`
   - Must contain encrypted values for:
     - `CLICKHOUSE_HOST`
     - `CLICKHOUSE_USER`
     - `CLICKHOUSE_PASSWORD`

`CLICKHOUSE_HOST` supports either `host` or `host:port`. If a port is embedded in `CLICKHOUSE_HOST`, it overrides `CLICKHOUSE_PORT`.

## Why There Is a Secret Delete Patch

The upstream base includes a plaintext `Secret` (`nrp-accounting-secrets`) with placeholder values.

This overlay deletes that plaintext Secret (`apps/nrp-clickhouse/overlays/dev/patches.yaml`) and replaces it with a SealedSecret-managed Secret of the same name.

## Namespace Behavior

This overlay is configured to **not** create a Namespace object.

You must create the target namespace before applying:

```bash
kubectl create namespace access-accounting
```

## Create / Update the Sealed Secret

Prerequisites:
- `kubeseal` installed locally
- Sealed Secrets controller installed in your cluster

1. Fetch the Sealed Secrets public certificate:

```bash
mkdir -p certs
kubeseal --fetch-cert \
  --controller-name sealed-secrets \
  --controller-namespace kube-system \
  > certs/sealed-secrets.pem
```

2. Create a temporary plaintext secret manifest:

```bash
cat > /tmp/nrp-accounting-secrets.yaml <<'YAML'
apiVersion: v1
kind: Secret
metadata:
  name: nrp-accounting-secrets
  namespace: access-accounting
type: Opaque
stringData:
  CLICKHOUSE_HOST: "clickhouse.example.org"
  CLICKHOUSE_USER: "default"
  CLICKHOUSE_PASSWORD: "replace-me"
YAML
```

3. Seal and write it to the repo file:

```bash
kubeseal \
  --cert certs/sealed-secrets.pem \
  --format yaml \
  < /tmp/nrp-accounting-secrets.yaml \
  > apps/nrp-clickhouse/overlays/dev/sealed-secret.yaml
```

4. Remove plaintext file:

```bash
rm -f /tmp/nrp-accounting-secrets.yaml
```

## Deploy With Argo CD

```bash
kubectl apply -f argocd/application-my-app-dev.yaml
```

Argo CD will reconcile:
- `apps/nrp-clickhouse/overlays/dev`

## Local Validation

```bash
kubectl kustomize apps/nrp-clickhouse/overlays/dev
```

Confirm output includes:
- `CronJob/nrp-accounting-etl`
- `Deployment/nrp-accounting-mcp`
- `Service/nrp-accounting-mcp`
- `Ingress/nrp-accounting-mcp` with host `nrp-accounting-mcp.nrp-nautilus.io`
- `ConfigMap/nrp-accounting-config` with your values
- `SealedSecret/nrp-accounting-secrets`
- no plaintext `Secret/nrp-accounting-secrets`
