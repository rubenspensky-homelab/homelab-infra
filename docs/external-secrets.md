# External Secrets Operator

External Secrets Operator is installed with Helm through Argo CD. It uses `ClusterSecretStore` resources for AWS-backed secrets in `us-east-2`.

Configured stores:

- `aws-secrets-manager`: AWS Secrets Manager
- `aws-parameter-store`: AWS Systems Manager Parameter Store

## Argo CD Sync Order

Argo CD sync waves express the intended dependency order:

1. Wave `-9`: `external-secrets-crds` installs the CRDs with server-side apply.
2. Wave `-8`: `external-secrets` installs the operator with Helm.
3. Wave `-7`: `external-secrets-store` creates both AWS `ClusterSecretStore` resources.
4. Later applications create their own `ExternalSecret` resources.

Waves order the Argo CD `Application` resources but do not prove that an AWS
store is ready or that a generated Kubernetes Secret already exists. Check the
resource conditions when troubleshooting a consumer.

## AWS Credentials Secret

The AWS IAM credentials are intentionally not committed to Git. Create this Secret out-of-band before the `ClusterSecretStore` reconciles:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: external-secrets
type: Opaque
stringData:
  access-key-id: <aws-access-key-id>
  secret-access-key: <aws-secret-access-key>
```

## Manual Refresh

The infrastructure `ExternalSecret` resources use `refreshPolicy: CreatedOnce`,
so changing a remote value or a force-sync annotation does not update an existing
target Secret. During a planned rotation, update the remote value, delete the
generated Secret, and wait for External Secrets Operator to recreate it:

```bash
kubectl delete secret tunnel-token -n cloudflare
kubectl wait externalsecret/cloudflare-tunnel-token \
  -n cloudflare \
  --for=condition=Ready \
  --timeout=60s
```

Restart or reload the consumer if it does not watch Secret changes. Database
credential rotation must also update the database role and should be handled in
a maintenance window. Do not delete a generated Secret until its remote value
and recovery procedure have been verified.

## Infrastructure Secrets

| ExternalSecret | Namespace | AWS remote key | Generated Kubernetes Secret |
| --- | --- | --- | --- |
| `cloudflare-tunnel-token` | `cloudflare` | `homelab/cloudflare/tunnel-token` | `tunnel-token` |
| `arc-github-app` | `arc-systems` | `homelab/github/arc-app` | `arc-github-app` |
| `umami-db-auth` | `databases` | `/homelab/umamik` | `umami-db-auth` |
| `umami-config` | `analytics` | `/homelab/umamik` | `umami-config` |
| `authentik-db-auth` | `databases` | `/homelab/authentik` | `authentik-db-auth` |
| `authentik-config` | `authentik` | `/homelab/authentik` | `authentik-config` |
| `grafana-db-auth` | `databases` | `/homelab/grafana` | `grafana-db-auth` |
| `grafana-database-credentials` | `monitoring` | `/homelab/grafana` | `grafana-database-credentials` |
| `grafana-admin-credentials` | `monitoring` | `/homelab/grafana` | `grafana-admin-credentials` |
| `aws-s3-backup` | `velero` | `homelab/aws/s3-backup` | `aws-s3-backup` |

## AWS Secrets Manager Values

Expected value for `homelab/cloudflare/tunnel-token`:

```json
{
  "token": "<cloudflare-tunnel-token>"
}
```

Expected value for `homelab/github/arc-app`:

```json
{
  "github_app_id": "<github-app-id>",
  "github_app_installation_id": "<github-app-installation-id>",
  "github_app_private_key": "<github-app-private-key>"
}
```

Expected JSON value for SSM Parameter Store parameter `/homelab/umamik`:

```json
{
  "databasePassword": "<raw-postgres-password>",
  "appSecret": "<umami-app-secret>"
}
```
