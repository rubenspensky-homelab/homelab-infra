# External Secrets Operator

External Secrets Operator is installed with Helm through Argo CD. It uses a `ClusterSecretStore` named `aws-secrets-manager` configured for AWS Secrets Manager in `us-east-2`.

## Argo CD Sync Order

Argo CD sync waves apply the resources in dependency order:

1. `external-secrets-crds`: installs the External Secrets CRDs with server-side apply.
2. `external-secrets`: installs the operator with Helm.
3. `external-secrets-store`: creates the AWS Secrets Manager `ClusterSecretStore`.
4. `infra-secrets`: creates infrastructure `ExternalSecret` resources that generate Kubernetes Secrets.

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

The infrastructure `ExternalSecret` resources use `refreshInterval: 0`, so they do not poll AWS Secrets Manager periodically after syncing. To force a refresh, update the `external-secrets.io/force-sync` annotation with a new value:

```bash
kubectl annotate externalsecret cloudflare-tunnel-token \
  -n cloudflare \
  external-secrets.io/force-sync=$(date +%s) \
  --overwrite
```

For a GitOps-driven refresh, change the same annotation value in Git and let Argo CD sync it.

## Infrastructure Secrets

The current infrastructure secret targets use alternate Kubernetes Secret names so they can be tested before replacing the manually created Secrets:

| ExternalSecret | Namespace | AWS Secrets Manager key | Generated Kubernetes Secret |
| --- | --- | --- | --- |
| `cloudflare-tunnel-token` | `cloudflare` | `homelab/infra/cloudflare-tunnel` | `tunnel-token-eso` |
| `arc-github-app` | `arc-systems` | `homelab/infra/arc-github-app` | `arc-github-app-eso` |

After validation, change the generated Secret names to `tunnel-token` and `arc-github-app`, then remove the manually created Secrets.

## AWS Secrets Manager Values

Expected value for `homelab/infra/cloudflare-tunnel`:

```json
{
  "token": "<cloudflare-tunnel-token>"
}
```

Expected value for `homelab/infra/arc-github-app`:

```json
{
  "github_app_id": "<github-app-id>",
  "github_app_installation_id": "<github-app-installation-id>",
  "github_app_private_key": "<github-app-private-key>"
}
```
