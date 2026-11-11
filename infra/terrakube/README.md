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

## Resolving the Tailscale hostname inside the cluster

Tailscale MagicDNS hostnames (`*.ts.net`) don't resolve inside the cluster by
default, but in-cluster workloads need to reach
`https://terrakube-api.serval-mark.ts.net`:

- the **API** pod, for Dex OIDC discovery (`/dex`);
- the **ephemeral executor jobs**, because Terrakube 2.34+ uses the API as a
  Terraform HTTP state backend (`TerrakubeApiUrl/tfstate/v1/...`).

Two pieces, both managed by the `tailscale` ArgoCD app, make this work:

### 1. Proxy Service (`infra/tailscale/tailscale-api-proxy-svc.yaml`)

A ClusterIP Service in the `tailscale` namespace that selects the Tailscale
ingress proxy pod for `terrakube-api`. It gives a stable address that
survives proxy pod restarts. The proxy serves the ts.net TLS certificate, and
the API ingress has
`tailscale.com/experimental-forward-cluster-traffic-via-ingress: "true"`, so
in-cluster HTTPS works end to end.

### 2. CoreDNS rewrite (`infra/tailscale/coredns-custom.yaml`)

k3s CoreDNS imports `*.override` keys from the `coredns-custom` ConfigMap in
`kube-system`. A `rewrite` rule there answers
`terrakube-api.serval-mark.ts.net` with the proxy Service's ClusterIP for
every pod in the cluster. Nothing needs patching by hand, and the rule keeps
working if the Service is recreated with a new IP.

To add another tailnet host, create a proxy Service for its ingress and add a
`rewrite` line to the same ConfigMap.
