# Terrakube

[Terrakube](https://docs.terrakube.io/) is deployed via the
[terrakube-helm-chart](https://github.com/terrakube-io/terrakube-helm-chart) and managed by ArgoCD (`infra/argo-apps/terrakube.yaml`).

## Architecture

| Component | How it's exposed |
|-----------|-----------------|
| UI        | `https://terrakube.serval-mark.ts.net` (Tailscale ingress) |
| API + Dex | `https://terrakube-api.serval-mark.ts.net` (Tailscale ingress) |

Authentication is handled by **Dex** (OIDC) with a **GitHub** connector,
restricted to the `code4romania` org. The `code4romania:infrastructure` team
is the admin group.

## Secrets

Sensitive values are stored in a Bitnami **SealedSecret** named
`terrakube-credentials` (see `sealed-secret.yaml`). It provides:

| Key | Used by |
|-----|---------|
| `PatSecret` | API — personal access token signing |
| `InternalSecret` | API — internal token signing |
| `AwsStorageAccessKey` / `AwsStorageSecretKey` | API — S3 storage |
| `AwsTerraformStateAccessKey` / `AwsTerraformStateSecretKey` | Executor — Terraform state |
| `AwsTerraformOutputAccessKey` / `AwsTerraformOutputSecretKey` | Executor — Terraform output |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | Dex — GitHub OAuth app |

The AWS keys are the same IAM credentials repeated under different names
because the chart uses different env var names per component.

## Tailscale ingress DNS workaround

The Terrakube API pod needs to reach Dex at
`https://terrakube-api.serval-mark.ts.net/dex` for OIDC discovery. Tailscale
MagicDNS hostnames (`*.ts.net`) are not resolvable from inside the cluster by
default, so we work around this with two pieces:

### 1. Proxy Service (`infra/tailscale/tailscale-api-proxy-svc.yaml`)

A ClusterIP Service in the `tailscale` namespace that selects the Tailscale
ingress proxy pod for `terrakube-api`. This gives a **stable IP** that
survives proxy pod restarts.

### 2. `hostAliases` on the API Deployment (manual, one-time)

The API Deployment is patched to map the Tailscale hostname to the proxy
Service's ClusterIP in the pod's `/etc/hosts`:

```bash
# Get the proxy Service ClusterIP
PROXY_IP=$(kubectl get svc terrakube-api-ts-proxy -n tailscale \
  -o jsonpath='{.spec.clusterIP}')

# Patch the API Deployment
kubectl patch deploy terrakube-api -n terrakube --type=strategic \
  -p "{\"spec\":{\"template\":{\"spec\":{\"hostAliases\":[{\"ip\":\"$PROXY_IP\",\"hostnames\":[\"terrakube-api.serval-mark.ts.net\"]}]}}}}"
```

The ArgoCD Application is configured with `ignoreDifferences` and
`RespectIgnoreDifferences=true` for this field, so ArgoCD will not revert the
patch on sync.

**When to re-run:** only if the `terrakube-api-ts-proxy` Service is deleted
and recreated (which assigns a new ClusterIP), or if the API Deployment is
fully replaced (e.g. major chart upgrade that renames it).
