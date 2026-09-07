# apidocs-api-chart

Helm chart for `apidocs-api-server`:
aggregated OpenAPI documentation for the bouc.io platform, served at
`https://api.<CLUSTER_DOMAIN>/v1/api-docs`.

## What it deploys

A single stateless Deployment plus a ClusterIP Service. **No database and no Redis**: the
service holds an in-memory spec cache only, which is why this chart has no `dependencies:`
(unlike its `*-api-chart` siblings).

| Object | Name |
|---|---|
| Deployment | `<release>-apidocs-api-chart` |
| Service | `<release>-apidocs-api-chart` (port 80 → container 3000) |
| ServiceAccount | `<release>-apidocs-api-chart` (when `serviceAccount.create`) |

The container name inside the pod is `{{ .Chart.Name }}`, i.e. `apidocs-api-chart` — use
`-c apidocs-api-chart` with `kubectl exec`.

## Values

Values live in three files, matching the convention used across bouc.io charts. There is
no plain `values.yaml`.

| File | Purpose |
|---|---|
| `base.values.yaml` | Values that never change between environments |
| `lcl.values.yaml` | Local cluster overrides |
| `snbx.values.yaml` | Sandbox overrides |

> In the cluster, FluxCD supplies values from generated ConfigMaps via `valuesFrom:`, not
> from these files directly. They are the source the ConfigMaps are generated from.

### Upstream services

`environment.*_API_URL` selects which services are aggregated. Defaults are the in-cluster
API service names:

```yaml
environment:
  AGENT_API_URL: "http://agent-api-chart.default.svc.cluster.local:80"
  MEMORY_API_URL: "http://memory-api-chart.default.svc.cluster.local:80"
  ADMIN_API_URL: "http://admin-api-chart.default.svc.cluster.local:80"
  PORTAL_API_URL: "http://portal-api-chart.default.svc.cluster.local:80"
  CHATBOT_API_URL: "http://chatbot-api-chart.default.svc.cluster.local:80"
  DISTILLER_API_URL: "http://memory-distiller-chart.default.svc.cluster.local:80"
```

These are the `*-api-chart` services (the APIs). `*-chart-service` names belong to the UIs
and will not serve a spec.

Each upstream must expose `/openapi.json`. That traffic is east-west only: those paths are
not routed at the Istio edge and need no CORS entry.

### Spec cache

| Value | Default | Meaning |
|---|---|---|
| `environment.SPEC_TTL_MS` | `300000` | How long a fetched spec is reused before refetching |
| `environment.SPEC_FETCH_TIMEOUT_MS` | `5000` | Per-upstream HTTP timeout |

## Probes

Both probes hit `/healthz`, which returns 200 whenever the process is up. This is
deliberate: a failing upstream must not fail the pod's probe, because the other specs
still load and the page still renders. Per-upstream state is in the `/healthz` body.

## Edge routing (not part of this chart)

Reaching the docs from outside the cluster requires `/v1/api-docs` in **both** places, in
**both** environments:

1. Istio `boucio-api-virtualservice` — a `prefix: /v1/api-docs` route to
   `apidocs-api-chart.default.svc.cluster.local:80`.
2. oauth2-proxy `upstreams` — the exact **and** trailing-slash pair.

Both matchers are prefix-based, so that single pair covers the HTML page, the Swagger UI
assets, and every `/v1/api-docs/specs/*.json`. Missing the trailing-slash entry produces a
redirect loop.

Because all external `api.<DOMAIN>` traffic passes through oauth2-proxy, the docs inherit
Keycloak login.

## Local usage

```bash
helm lint .
helm template test . -f base.values.yaml -f lcl.values.yaml
helm install apidocs . -f base.values.yaml -f lcl.values.yaml
```

> The chart must be published to the GitLab package registry by CI before FluxCD can
> reconcile it. Pushing chart source to git is not enough.

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
