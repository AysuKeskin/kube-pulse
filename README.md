# kube-pulse

A small Go HTTP service, taken end to end: app → container → Helm/minikube →
CI/CD → observability → a public HTTPS URL.

The service itself is deliberately tiny. The point of the project is everything
around it — a hardened image, a real Helm chart with dev/prod values, a pipeline
that scans and signs what it ships, Terraform-provisioned infrastructure, and
metrics and alerts that actually fire.

Live: `curl https://devops-case.aysu-keskin.uk/ping` → `pong`

> ❗ **The public URL is offline for now.** I've stopped the EC2 instance to save on
> server costs, so the URL won't answer until it's started again. Nothing was
> torn down: the Elastic IP, DNS record, cluster, and TLS cert are all still in
> place. A captured run against the live URL is in
> [`docs/evidence/public-url/`](docs/evidence/public-url/public-url-ping.png).
> To bring it back (takes about 2–3 minutes):
>
> ```sh
> ID=$(terraform -chdir=infra/terraform output -raw ec2_instance_id)
> aws ec2 start-instances --instance-ids "$ID" --region eu-north-1
> aws ec2 wait instance-status-ok --instance-ids "$ID" --region eu-north-1
> curl https://devops-case.aysu-keskin.uk/ping   # → pong once minikube is back up
> ```
>
> minikube runs as a systemd unit and restarts on boot. If `/ping` times out
> while the pods are Running, the host → minikube port forwards didn't
> reconnect after the reboot. Restart them from an SSM session
> (`aws ssm start-session --target "$ID" --region eu-north-1`):
>
> ```sh
> sudo systemctl restart minikube-http-forward minikube-https-forward
> ```
>
> More in [RUNBOOK.md → Common issues](RUNBOOK.md#common-issues).

Security posture and reporting: see [SECURITY.md](SECURITY.md). Design decisions: [`docs/adr/`](docs/adr/).

## Architecture

![Architecture](docs/architecture.png)

Client → Cloudflare (HTTPS) → Elastic IP → EC2 (minikube: ingress-nginx → Service → pods); TLS terminates at the ingress with a Let's Encrypt cert from cert-manager. CI/CD: GitHub Actions builds, scans (Trivy), signs (cosign), pushes to GHCR, then deploys to EC2 over SSM via OIDC. Observability (local): the app's `/metrics` → Prometheus → Grafana + Alertmanager. Vector version: [`docs/architecture.svg`](docs/architecture.svg).

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/ping` | returns `pong` |
| GET | `/healthz` | liveness probe (always 200 if process is up) |
| GET | `/readyz` | readiness probe (503 during shutdown drain) |
| GET | `/version` | build info: version, commit, built_at |
| GET | `/metrics` | Prometheus exposition: `http_requests_total`, `http_request_duration_seconds`, Go/process collectors |

## Quick start

```sh
make help          # list all targets
make docker-build  # build the container image
make docker-run    # run it on :8080
curl localhost:8080/ping        # → pong
```

## All Makefile targets

| Target | What it does | Needs |
|---|---|---|
| `make help` | list targets | — |
| `make run` | run server with `go run` | Go 1.25+ |
| `make build` | build local binary into `bin/server` | Go 1.25+ |
| `make test` | unit tests with race detector + coverage | Go 1.25+ |
| `make docker-test` | run tests inside a `golang:1.25` container | Docker only |
| `make lint` | `go vet ./...` | Go 1.25+ |
| `make docker-build` | multi-stage build, tags image with git SHA + `latest` | Docker |
| `make docker-run` | run container on `:8080`, read-only, all caps dropped | Docker |
| `make compose-up` | `docker compose up --build` (local dev) | Docker |
| `make compose-down` | `docker compose down` | Docker |
| `make scan` | Trivy scan, fails on `CRITICAL`/`HIGH` | Trivy |
| `make helm-lint` | lint the Helm chart against dev values | Helm |
| `make helm-template` | render templates locally for inspection | Helm |
| `make minikube-load` | load the locally-built image into minikube | minikube |
| `make deploy-dev` | `helm upgrade --install` with `values-dev.yaml` | Helm + kubectl |
| `make deploy-prod` | `helm upgrade --install` with `values-prod.yaml` | Helm + kubectl |
| `make rollout-status` | wait for the current Deployment rollout (2 min cap) | kubectl |
| `make rollout-dev` | `deploy-dev` then wait for pods to be Ready | Helm + kubectl |
| `make rollout-prod` | `deploy-prod` then wait for pods to be Ready | Helm + kubectl |
| `make rollback` | `helm rollback` to previous revision, then wait for Ready | Helm + kubectl |
| `make helm-history` | show release revision history | Helm |
| `make helm-uninstall` | remove the release | Helm |
| `make clean` | remove `bin/` | — |

If Go isn't installed locally, use `make docker-test` / `make docker-build` / `make docker-run` — Docker is enough.

## Environment variables

| Var | Default | Notes |
|---|---|---|
| `PORT` | `8080` | container listen port (must be ≥ 1024, non-root) |
| `HOST_PORT` | `8080` | host port mapped by docker compose only |
| `LOG_LEVEL` | `info` | `debug` / `info` / `warn` / `error` |

See `.env.example`. In production (Kubernetes), config comes from the ConfigMap in the Helm chart, not `.env`.

## Deploy to Kubernetes (local minikube)

```sh
minikube start --driver=docker --cpus=2 --memory=4096
minikube addons enable ingress
minikube addons enable metrics-server   # for HPA in prod values
make docker-build
make minikube-load                       # push the local image into minikube
make rollout-dev                         # deploy dev (1 replica, debug) then wait
make rollout-prod                        # deploy prod (3 replicas, HPA) then wait
make helm-history
make rollback                            # revert to the previous revision and wait for Ready
```

`make deploy-dev` and `make deploy-prod` set `image.pullPolicy=Never` only for
this local minikube flow, because the image is loaded directly into minikube.
For the EC2/GHCR deployment, the chart default `IfNotPresent` is used instead,
or CI overrides it explicitly.

The `values-dev.yaml` and `values-prod.yaml` deliberately differ on replica count, log level, resources, ingress host, and HPA settings — see those files for the deltas. Resource requests/limits are conservative starting estimates for a static Go binary that idles near zero CPU and ~10 Mi memory; dev sits at the floor (50m / 64Mi requests) and prod doubles both for traffic headroom. They can be tuned against real `kubectl top pod` numbers under sustained load.

### Environments: dev = local, prod = cloud

The two values files map to where each is actually run:

| | `values-dev.yaml` | `values-prod.yaml` |
|---|---|---|
| Runs on | local minikube (laptop) | EC2 minikube (cloud) |
| Driven by | `make rollout-dev` | the CD pipeline |
| Replicas / logs | 1 / debug | 3 + HPA / info |
| Ingress | `tiny.dev.local` | `devops-case.aysu-keskin.uk` (HTTPS) + raw-IP catch-all |

`dev` is the local developer loop; `prod` is the real cloud target. I intentionally do **not** run a second `dev` release on the same EC2 host — one node gives no real isolation, so it would be a duplicate rather than a distinct environment, and prod's `helm --atomic` already guards against a bad image. See [ADR 015](docs/adr/cicd-and-infrastructure.md).

### Rollout strategy

Deployments use an explicit `RollingUpdate` with `maxSurge: 1, maxUnavailable: 0`. The intent is **safe and verifiable rollout/rollback over raw speed** — I'd rather have a predictable pod footprint in small clusters (minikube on a laptop, single-node EC2) than save a handful of seconds by surging to 2× replicas. Even when HPA scales the deployment to 10 replicas, only one extra pod ever exists during a rollout. At `maxUnavailable: 0` the chart never drops below desired capacity, so client requests keep flowing while the new image takes over. This pairs with the app's graceful SIGTERM drain to deliver true zero-downtime rollouts.

### Rollout & rollback

The `/version` endpoint makes rollouts observable end to end. With
`helm upgrade --set image.tag=<new>` the response flips to the new tag, and
`helm rollback` immediately flips it back — `helm history` shows the full
revision chain. This is the fastest way to confirm that a deploy actually
replaced the running image rather than merely reporting success.

## Chaos test

Killing one pod from a 3-replica deployment to observe Kubernetes self-healing:

```sh
kubectl delete pod <one-of-three-pods>
```

A replacement pod is scheduled within ~2 seconds and reaches `Ready` shortly after. The replica count stays at 3 throughout because HPA's `minReplicas: 3` floor enforces it independent of CPU metrics. What this shows: the Deployment controller doesn't wait for the dead pod to be `Terminated` before creating the replacement — it reacts to the desired-vs-actual delta immediately, so a graceful shutdown drain and a new pod's startup overlap cleanly.

## Production-aware extras

- **Multi-stage Dockerfile** → distroless `static-debian12:nonroot` base pinned by digest, non-root uid `65532`, ~7 MB image.
- **Docker `HEALTHCHECK`** → uses `/server healthcheck`, so the distroless runtime image does not need curl, wget, or a shell.
- **OCI image labels** (`org.opencontainers.image.source` etc.) so GHCR links the image back to the repo.
- **`/healthz` vs `/readyz`** wired to liveness vs readiness probes separately.
- **Request-ID middleware** generates `X-Request-ID` per request and emits it on every log line.
- **Structured JSON logs** via `log/slog` — fields: `timestamp, level, msg, request_id, method, path, status, duration_ms`. `/healthz`, `/readyz`, `/metrics` are demoted to `DEBUG` to avoid probe noise.
- **Graceful shutdown** — on SIGTERM, flips `/readyz` to 503 and drains in-flight requests (15s deadline) so Kubernetes rolling updates don't drop traffic.
- **HPA** — CPU-based autoscaling, min 3 / max 10 at 70%. Needs metrics-server plus resource requests to compute utilization; `minReplicas` is a hard floor independent of CPU.

## CI/CD

`.github/workflows/ci-cd.yml` runs on every PR and push: lint + race tests, gitleaks secret scan, helm lint, and a Trivy image scan that fails on CRITICAL/HIGH. On `main` and `v*` tags it also pushes to GHCR, signs the image with cosign (keyless OIDC), and produces + attests an SPDX SBOM. A push to `main` ends with an auto-deploy to EC2 over SSM; a `v*` tag cuts a GitHub Release instead (deploy is skipped).

Keyless signing needs `id-token: write` on the job, and the SBOM is bound to the
image as a cosign attestation rather than shipped as a loose file, so it can be
verified against the digest:

```sh
cosign verify ghcr.io/aysukeskin/kube-pulse:0.1.0 \
  --certificate-identity-regexp 'https://github.com/AysuKeskin/kube-pulse/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Cloud infrastructure

A single EC2 host in `eu-north-1` running minikube, reachable on an Elastic IP. Provisioned with Terraform under `infra/terraform/`.

```sh
cd infra/terraform
terraform init
terraform apply             # ~3 min; spins up EC2 + EIP + SG + GitHub OIDC role
terraform output            # AWS_REGION, AWS_ROLE_ARN, EC2_INSTANCE_ID for GH secrets
terraform destroy           # full teardown
```

The stack is intentionally narrow: no SSH (port 22 is closed at the SG), no long-lived AWS keys (GitHub Actions assumes the `gha_deploy` role via OIDC), and no GHCR secret on the host (the image is published as a public package, so minikube pulls anonymously). See [`docs/adr/cicd-and-infrastructure.md`](docs/adr/cicd-and-infrastructure.md) for the deploy-via-SSM and `t3.medium` tradeoffs.

## Observability

The app exposes Prometheus metrics at `/metrics` (`http_requests_total`, `http_request_duration_seconds`, plus Go/process collectors). Monitoring runs on **local minikube** — the EC2 box is sized for the app, not a full stack (see [ADR 016](docs/adr/observability-and-tls.md)).

```sh
make monitoring-install                  # kube-prometheus-stack into the `monitoring` ns
# deploy the app with the Prometheus Operator objects switched on:
helm upgrade --install kube-pulse helm/kube-pulse \
  -f helm/kube-pulse/values-dev.yaml \
  --set image.pullPolicy=Never \
  --set metrics.serviceMonitor.enabled=true \
  --set metrics.prometheusRule.enabled=true
make monitoring-prometheus               # localhost:9090 → Status/Targets shows the app endpoint UP
make monitoring-grafana                  # localhost:3000 → import docs/dashboards/kube-pulse-http.json
make load-gen                            # generate traffic so the panels move
```

Grafana admin password: `kubectl -n monitoring get secret monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d`.

The dashboard (`docs/dashboards/kube-pulse-http.json`) shows RPS, p50/p95 latency, 5xx error rate, and pod restarts. Two alerts ship as a `PrometheusRule`: `HighErrorRate` (5xx ratio > 5% for 5m) and `AppDown` (target unscrapable for 2m). The `serviceMonitor`/`prometheusRule` toggles default to **off** so a normal install never needs the Prometheus Operator CRDs.

## TLS

HTTPS via cert-manager + Let's Encrypt with a DNS-01 (Cloudflare) challenge; the ingress terminates TLS behind Cloudflare Full (strict). DNS-01 is more reliable than HTTP-01 behind a Cloudflare proxy, since it only needs a TXT record rather than reaching the origin on port 80. Setup steps in [RUNBOOK.md](RUNBOOK.md) → "TLS / HTTPS".

## Evidence

Captured runs, not claims. Each file shows the named behaviour actually happening.

| Artifact | Shows |
|---|---|
| [`public-url/public-url-ping.png`](docs/evidence/public-url/public-url-ping.png) | `curl -i` against the live HTTPS URL — 200, `pong`, Cloudflare in front |
| [`cicd/pipeline-green.png`](docs/evidence/cicd/pipeline-green.png) | the full pipeline green: test, gitleaks, helm lint, Trivy-gated build, cosign + SBOM, SSM deploy |
| [`observability/prometheus-targets.png`](docs/evidence/observability/prometheus-targets.png) | Prometheus scraping the app through the chart's `ServiceMonitor`, target UP |
| [`observability/alert-rules.png`](docs/evidence/observability/alert-rules.png) | `HighErrorRate` and `AppDown` loaded from the chart's `PrometheusRule` |
| [`observability/grafana-dashboard.png`](docs/evidence/observability/grafana-dashboard.png) | RPS, p50/p95 latency, 5xx ratio, pod restarts under load |
| [`observability/metrics-cardinality.txt`](docs/evidence/observability/metrics-cardinality.txt) | 50 distinct unknown paths collapsing into one `route="unmatched"` series |
| [`rollout/rollout-strategy.txt`](docs/evidence/rollout/rollout-strategy.txt) | the Deployment's `RollingUpdate` settings (`maxSurge: 1, maxUnavailable: 0`) |
| [`rollout/rollout.txt`](docs/evidence/rollout/rollout.txt) | an upgrade to a new image tag, revision bump, `/version` following it |
| [`rollout/rollback.txt`](docs/evidence/rollout/rollback.txt) | `helm rollback` reverting that upgrade in one command |
| [`chaos/chaos.txt`](docs/evidence/chaos/chaos.txt) | the pod deleted and rebuilt by its ReplicaSet, Ready again in 3s |

## Layout

| Path | Contents |
|---|---|
| `cmd/server/` | entrypoint + lifecycle (graceful shutdown) |
| `internal/handlers/` | `/ping` `/healthz` `/readyz` `/version` |
| `internal/middleware/` | request-id, access log, metrics |
| `internal/metrics/` | Prometheus collectors (`http_requests_total`, duration histogram) |
| `internal/version/` | ldflags-populated build info |
| `helm/kube-pulse/` | chart: Deployment, Service, Ingress, ConfigMap, Secret, HPA, ServiceMonitor, PrometheusRule, `values-dev/prod.yaml` |
| `infra/terraform/` | EC2, EIP, SG, IAM OIDC role |
| `.github/workflows/` | `ci-cd.yml`: build → scan → sign → push → SSM deploy |
| `docs/adr/` | architecture decision records |
| `docs/dashboards/` | Grafana dashboard JSON |
| `docs/evidence/` | captured screenshots and command output backing the claims above |
| `docs/architecture.*` | architecture diagram (draw.io — PNG embed + SVG vector/source) |

