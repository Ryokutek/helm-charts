# helm-charts

Helm charts for Ryokutek applications. Every chart lives under `charts/`, one directory per
chart, versioned independently (SemVer in its `Chart.yaml`).

| Chart            | Purpose |
|------------------|---------|
| `dotnet-backend` | A .NET backend service — Deployment, Service, ServiceAccount, HTTPRoute, optional ExternalSecret |
| `react-frontend` | A React SPA served as a static build (e.g. nginx-unprivileged) — same resource set |

## Conventions

- **Gateway API only.** Exposure is an `HTTPRoute` attached to the shared Gateway
  (`traefik-reverse-proxy/traefik-reverse-proxy` by default, overridable via `route.gateway.*`);
  there is no Ingress template. The release namespace must carry the
  `traefik-reverse-proxy/allow-routes: "true"` label or the route will not attach.
- **Secrets via External Secrets Operator.** `externalSecret.data` takes ExternalSecret
  `spec.data` entries verbatim (default store: the `aws-ssm` `ClusterSecretStore`); they sync
  into `<release>-env`, which the pod `envFrom`s. No secret values in git.
- **Stateless only.** No PVC/HPA/PDB templates — the target is the single-node cluster.
- **Health endpoints.** `dotnet-backend` probes `/livez` (liveness) and `/readyz` (readiness)
  on the container port (default 8080); `react-frontend` probes `/`. Override
  `livenessProbe`/`readinessProbe` wholesale if an app differs.
- `image.repository` and `route.hostname` are required (template-time `required` guards).

## Consumption — OCI from ECR

Charts are consumed as OCI artifacts from ECR (repository `charts/<chart>`; the ECR
repositories are Terraform-owned in `ryokutek-infrastructure`). Publish:

```sh
aws ecr get-login-password --region <region> \
  | helm registry login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
helm package charts/dotnet-backend
helm push dotnet-backend-0.1.0.tgz oci://<account>.dkr.ecr.<region>.amazonaws.com/charts
```

An ArgoCD Application then sources `repoURL: <account>.dkr.ecr.<region>.amazonaws.com/charts`,
`chart: dotnet-backend`, `targetRevision: 0.1.0`. The ECR repositories, push automation, and
ArgoCD's ECR credential are wired in `ryokutek-infrastructure` when the first app lands.

## Development

```sh
helm lint charts/dotnet-backend --set image.repository=example --set route.hostname=example.com
helm template x charts/dotnet-backend --set image.repository=example --set route.hostname=example.com
```
