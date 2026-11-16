# Kubernetes AI Monitoring & RCA Agent (starter MVP)

An in-cluster, read-only Kubernetes watcher that collects pod status, restart information,
previous/current container logs and warning events; creates incident records; asks a
configurable OpenAI-compatible chat-completions endpoint for evidence-based RCA and
exactly five Whys; and sends reports through configurable SMTP.

## Scope and limitations of this starter

This is an executable foundation, not yet a fully production-certified SRE platform.
The watcher covers pods and Warning events across namespaces. The dashboard also
queries live pod, node and workload inventory, supports on-demand current/previous
container logs, and queries Prometheus CPU/memory metrics. Robust incident correlation,
retry queues, multi-replica leader election, and Helm packaging remain follow-on work.
Email delivery uses SMTP with STARTTLS or implicit TLS.

The agent does not perform remediation. Review all AI recommendations before executing.

## Repository

- `app/monitor.py`: Kubernetes watches, evidence gathering, incident creation
- `app/ai.py`: configurable LLM analysis adapter
- `app/emailer.py`: SMTP email delivery
- `app/db.py`: PostgreSQL incident persistence
- `app/main.py`: FastAPI health and incident endpoints
- `k8s/`: read-only RBAC and deployment manifests

## Requirements

- Kubernetes 1.24+ and a namespace named `observability`
- PostgreSQL reachable from the agent
- Container registry reachable by cluster nodes
- Egress to configured AI endpoint and the organization SMTP relay
- SMTP relay credentials and an approved sender mailbox
- AI endpoint supporting OpenAI-compatible `/chat/completions` with JSON response mode

For Azure OpenAI, set `AI_BASE_URL` to the resource deployment endpoint compatible
with chat completions and set `AI_MODEL` to the deployment name. Provider-specific
API-version/auth handling may need a small adapter change for your Azure configuration.
For Ollama, use a compatible endpoint such as `http://ollama:11434/v1`.

## Build

```bash
docker build -t YOUR_REGISTRY/k8s-ai-rca-agent:0.1.0 .
docker push YOUR_REGISTRY/k8s-ai-rca-agent:0.1.0
```

## Configure and deploy

1. Create PostgreSQL database `rca` and user with schema creation rights.
2. Provision SMTP username/password and AI/database credentials through your secret manager. Do not commit real secrets.
3. Replace image in `k8s/deployment.yaml`.
4. Apply:

```bash
kubectl create namespace observability
kubectl apply -f k8s/rbac.yaml
kubectl apply -f k8s/configmap.yaml
# Create k8s-ai-rca-secrets from your secret manager or a secured deployment pipeline.
kubectl apply -f k8s/deployment.yaml
kubectl -n observability rollout status deployment/k8s-ai-rca-agent
kubectl -n observability logs deployment/k8s-ai-rca-agent -f
```

For production, provision secrets with your enterprise secret manager rather than
checking plaintext Kubernetes Secret manifests into source control. Ensure etcd
encryption at rest and network policies restrict egress.

## API

Port-forward:

```bash
kubectl -n observability port-forward svc/k8s-ai-rca-agent 8080:8080
```

- `GET /healthz`
- `GET /readyz`
- `GET /incidents?limit=100`
- `GET /incidents/{incident_id}`
- `GET /api/pods` (optional `namespace`, `search`, `phase`, `limit` filters)
- `GET /api/logs?namespace=...&pod=...&container=...&tail=200&previous=false`
- `GET /api/topology` (optional `namespace`): namespace cards, inventory and relationship table
- `GET /api/topology/graph` (optional `namespace`): typed resource graph (`nodes`, `edges`, `revision`)

The starter API has no authentication. Do not expose it outside a trusted internal
network until authentication/authorization is added.

## SMTP email setup

Configure `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_FROM`,
`SMTP_STARTTLS`, `SMTP_USE_SSL`, `EMAIL_RECIPIENTS`, and optional `EMAIL_CC`.
Port 587 with STARTTLS is the default. For implicit TLS (commonly port 465), set
`SMTP_USE_SSL=true` and `SMTP_STARTTLS=false`. Store SMTP credentials in the
Kubernetes Secret / enterprise secret manager, not in the ConfigMap or Git.

## Important production hardening before broad rollout

- Add node, deployment, ReplicaSet, StatefulSet and DaemonSet watchers.
- Add Prometheus metrics collection (Metrics Server is not a replacement for historical
  Prometheus time series).
- Use a durable work queue (e.g. PostgreSQL outbox, RabbitMQ, or Redis Streams) so AI
  and email outages do not block watch processing.
- Implement incident lifecycle and cooldown/deduplication using stable workload-level
  fingerprints and time windows. This starter only deduplicates open incidents by a
  basic fingerprint.
- Add restart-rate windows; current restart trigger uses the observed cumulative count.
- Add leader election or separate watcher/worker deployments before scaling replicas.
- Add API authentication, audit logging, network policies, PDB, resource sizing and
  backup/restore.
- Redact secrets and sensitive data from logs before sending evidence to an external LLM.
- Validate the generated RCA and five-whys structure; do not auto-run generated commands.
- Add tests against a disposable Kubernetes cluster (kind/k3d) and mock SMTP/LLM APIs.

## Security

The ClusterRole grants only `get`, `list`, and `watch` on workload metadata, events,
nodes, namespaces and pod logs. No write verbs are granted. Container logs may contain
credentials or personal data; configure redaction and retention before production use.


## Phase 2 dashboard and ingress

The dashboard is served at `/` and refreshes cluster data every 30 seconds. It includes live pod monitoring, an on-demand container log viewer, and a Light / Dark / System theme selector.
It uses `/api/overview`, `/api/nodes`, `/api/workloads`, `/api/pods`, `/api/logs`, and `/api/metrics`. Configure `PROMETHEUS_URL` to enable CPU and memory query cards.
The metrics queries assume Prometheus has node-exporter metrics
(`node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_memory_MemTotal_bytes`).

Deploy the dashboard ingress after installing an ingress controller and creating
the TLS secret:

```bash
kubectl -n observability create secret tls clusterpulse-tls \
  --cert=./tls.crt --key=./tls.key
kubectl apply -f k8s/traefik-ingressroute.yaml
```

Change `clusterpulse.example.internal` and `ingressClassName: nginx` to match
your internal DNS and ingress controller. Point the internal DNS record at the
ingress controller address. If TLS is terminated upstream, adapt/remove the TLS
section according to your security policy.

Dashboard routes:
- `/` dashboard
- `/api/overview` cluster KPIs
- `/api/nodes` node readiness and pressure
- `/api/workloads` deployments, StatefulSets, DaemonSets and ReplicaSets
- `/api/pods` pod inventory, readiness, restart counts and node placement
- `/api/logs?namespace=...&pod=...` current or previous container logs (read-only)
- `/api/metrics` Prometheus CPU and memory
- `/incidents` incident history API

The dashboard is intentionally unauthenticated in this starter. Expose it only
on a trusted internal network or put SSO/authentication and network restrictions
in front of the Ingress before production use. The current app RBAC is read-only.


# Production deployment: Traefik + Basic Auth + Prometheus

This deployment uses Traefik CRD `IngressRoute` and `Middleware` resources.
It assumes Traefik is already installed with the Kubernetes CRDs and entryPoints
named `web` and `websecure`. Verify your installation's CRD API group/version and
entryPoint names before applying.

## 1. Configure DNS and TLS

Choose an internal DNS name, for example `clusterpulse.example.internal`, and
point it to the internal address of your Traefik LoadBalancer/NodePort.

Create a TLS certificate trusted by your organization's client devices. For an
internal CA, distribute the CA certificate to managed browsers/workstations.

```bash
kubectl -n observability create secret tls clusterpulse-tls \
  --cert=./tls.crt --key=./tls.key
```

Update the hostname in `k8s/traefik-ingressroute.yaml` to your real internal name.

## 2. Create Basic Auth credentials (bcrypt)

Install `htpasswd` (provided by Apache httpd tools) on a trusted admin workstation.
Generate a bcrypt hash interactively; do not put a plaintext password in shell
history or source control.

```bash
htpasswd -nB clusterpulseadmin
```

The output is a single line:
`clusterpulseadmin:$2y$...`

Store that line in a file named `users` with restrictive local permissions:

```bash
umask 077
# Paste the htpasswd output into ./users using a secure editor.
kubectl -n observability create secret generic clusterpulse-users \
  --from-file=users=./users
shred -u ./users 2>/dev/null || rm -f ./users
```

Do not use `--from-literal` with a password. Do not commit the generated hash
or secret YAML to Git. Rotate credentials by recreating the Secret and reviewing
your secret-management process.

## 3. Install Prometheus (not currently installed)

Use the Prometheus Community Helm chart. In an offline/restricted on-prem
environment, mirror all required images to your internal registry and configure
the chart image repositories before installation.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
```

Edit `k8s/prometheus-values.yaml` and set `storageClassName` to a real on-prem
StorageClass. Then install:

```bash
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values k8s/prometheus-values.yaml \
  --wait
```

Check components:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get servicemonitors -A
```

The configured in-cluster Prometheus URL is:
`http://kps-kube-prometheus-prometheus.monitoring.svc:9090`

Confirm it matches the actual Service created by your Helm release. The dashboard's
CPU and memory queries require node-exporter series:
`node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`,
`node_memory_MemTotal_bytes`.

For an air-gapped cluster, mirror chart images and chart packages through your
approved artifact repository; Helm repository access from cluster nodes is not
required, but the admin workstation needs chart access or a mirrored chart.

## 4. Configure ClusterPulse

Update `k8s/configmap.yaml`:
- Set `PROMETHEUS_URL` to the verified in-cluster Prometheus Service.
- Set AI endpoint and model.
Provision `DATABASE_URL`, AI key, SMTP credentials and recipient addresses through your enterprise secret manager. The `k8s/secret-template.yaml`
is a placeholder only; never apply it unchanged and never commit a populated copy.

Update the container image in `k8s/deployment.yaml`, then deploy:

```bash
kubectl apply -f k8s/rbac.yaml
kubectl apply -f k8s/configmap.yaml
# Create k8s-ai-rca-secrets from your secret manager or a secured deployment pipeline.
kubectl apply -f k8s/deployment.yaml
kubectl -n observability rollout status deployment/k8s-ai-rca-agent
```

The existing application needs PostgreSQL reachable at the configured `DATABASE_URL`.
Provision PostgreSQL with persistent storage, backups, TLS where required, and
credentials managed outside Git before starting the agent.

## 5. Apply Traefik authentication and routes

```bash
kubectl apply -f k8s/traefik-middleware.yaml
kubectl apply -f k8s/traefik-ingressroute.yaml
kubectl get ingressroute,middleware -n observability
```

If Traefik is installed in a namespace other than `traefik`, adjust the optional
NetworkPolicy namespace selector before applying it. NetworkPolicy enforcement
depends on the cluster CNI.

## 6. Verify

```bash
kubectl get ingressroute -n observability
kubectl get secret clusterpulse-users clusterpulse-tls -n observability
kubectl logs -n observability deployment/k8s-ai-rca-agent --tail=100
```

From a workstation that trusts the TLS issuer:

```bash
curl -I https://clusterpulse.example.internal/
```

Expected behavior:
- Unauthenticated request returns HTTP 401.
- Authenticated browser session can load the dashboard.
- HTTP request redirects to HTTPS.
- Dashboard API requests are protected by the same BasicAuth middleware.

Do not bypass the Ingress by exposing the Service with NodePort or LoadBalancer.
Keep the ClusterPulse Service as ClusterIP.

## 7. Network and security controls

- Permit inbound HTTPS only from approved corporate/VPN subnets at the firewall.
- Restrict Traefik dashboard/API separately; never expose Traefik's admin API publicly.
- Use TLS certificates from your internal PKI and plan certificate renewal.
- Apply Kubernetes Secret encryption at rest and RBAC restrictions.
- Use a secret manager/External Secrets Operator for production credentials.
- Restrict egress to Kubernetes API, PostgreSQL, Prometheus, configured AI endpoint,
  Microsoft identity endpoints, and SMTP as needed.
- Add authentication rate limiting at the edge and monitor authentication failures.
- Basic Auth is a shared-secret mechanism; use individual credentials/SSO if audit
  attribution and centralized identity lifecycle are required.
- The optional `k8s/clusterpulse-networkpolicy.yaml` only restricts ingress to pods
  labeled as Traefik in namespace `traefik`. Verify actual labels and namespace first.
- The initial FastAPI health/readiness and incident endpoints are not independently
  authenticated. They are protected only when accessed through this Traefik route.
  Keep the ClusterIP service internal and do not add a second unauthenticated route.


## GitHub Actions: automatic image build and deployment manifest update

The workflow at `.github/workflows/build-image.yml` runs on pushes to `main` when
application code or image build inputs change. It:

1. Validates Python syntax.
2. Builds the Docker image with Buildx.
3. Publishes `ghcr.io/<owner>/<repository>:latest` and
   `ghcr.io/<owner>/<repository>:sha-<commit>`.
4. Updates the image reference in `k8s/deployment.yaml` to the repository's
   `:latest` tag and updates the pod-template revision annotation.
5. Commits the manifest update back to the same branch. The workflow path filter
   excludes `k8s/deployment.yaml`, so this commit does not trigger an image-build loop.

### One-time GitHub setup

- Push this project to a GitHub repository and ensure its default branch is named
  `main` (or change the branch filter in the workflow).
- In **Settings → Actions → General → Workflow permissions**, allow read and write
  permissions. The workflow also declares `contents: write` and `packages: write`.
- No personal access token is required for GHCR publishing; the workflow uses
  `GITHUB_TOKEN`.
- After the first successful run, verify the package under the repository's
  **Packages** tab. Configure package visibility and access for your cluster.

### Important deployment notes

- Replace the initial placeholder image in `k8s/deployment.yaml` if you deploy
  before the first successful workflow run. The workflow replaces it automatically.
- The deployment manifest uses `:latest` plus `imagePullPolicy: Always`; the
  revision annotation changes on each source commit and triggers a rollout.
  The immutable `sha-<commit>` tag is also published for traceability and rollback.
- If the GHCR package is private, create an image pull Secret in the `observability`
  namespace and add `imagePullSecrets` to the pod spec. For on-prem clusters, ensure
  nodes can reach `ghcr.io`, or mirror the published image into your internal registry.
- GitHub Actions commits directly to `main`. Branch protection rules that prohibit
  direct pushes by `GITHUB_TOKEN` must be adjusted, or the workflow must be changed
  to open a pull request instead.
- This workflow updates the Kubernetes manifest in Git; it does not directly apply
  the manifest to the cluster. A GitOps controller (such as Argo CD or Flux) should
  watch the repository to deploy the updated revision automatically. Otherwise,
  apply the changed manifest through your normal deployment process.


## Topology graph (v0.3)

Open **Topology → Resource graph** (or click a namespace name / *Graph* on a namespace card; deep link:
`/#topology/<namespace>`). The graph is laid out left to right:

    Middleware -> IngressRoute -> Service -> Deployment | StatefulSet | DaemonSet -> Pod
    Ingress ----------------------^            ^   ^   ^
                       PVC / ConfigMap / ServiceAccount

- Click a resource to trace its full upstream/downstream path and open the details panel
  (health, metadata, neighbours, *View logs* for pods, *Copy kubectl get*).
- Missing objects that something references (a ConfigMap, PVC, Service or Middleware that does not exist) are drawn as
  dashed red "Missing" nodes. This is the usual cause of `FailedMount` and 503s from Traefik.
- Drag to pan, scroll to zoom, `/` search, `F` fit, `Esc` close. Auto-updates every 10 s (Pause/Resume);
  the canvas is only redrawn when the graph actually changed (`revision`).
- Layout is computed in the browser with no external library, so it works in air-gapped clusters.

RBAC needed (already in `k8s/rbac.yaml`): `list` on `configmaps`, `serviceaccounts`, and `traefik.io`
`ingressroutes`/`middlewares`. If a kind is not permitted the graph still renders and shows a
"kind(s) skipped" notice. ConfigMap *contents* are never returned (only the key count); Secrets are never read.
To opt out of ConfigMaps: `TOPOLOGY_INCLUDE_CONFIGMAPS=false` and remove `configmaps` from the ClusterRole.

## Development

```bash
pip install -r requirements-dev.txt
python -m pytest -q                 # unit + API tests (no cluster needed)
python -m dev.mock_server           # dashboard with demo data on http://127.0.0.1:8099
```

## Upgrade notes (0.2 -> 0.3)

- Re-apply `k8s/rbac.yaml`, `k8s/configmap.yaml`, `k8s/deployment.yaml` (adds RBAC, hardening, `Recreate` strategy).
- `/readyz` now returns 503 while PostgreSQL is unreachable (traffic is withheld; the pod is not restarted).
- `/api/logs` now refuses namespaces listed in `MONITOR_EXCLUDED_NAMESPACES` (403).
- `/api/topology` totals changed: `deployments` -> `workloads` (Deployments + StatefulSets + DaemonSets), new `config`.
- If you apply `k8s/clusterpulse-networkpolicy.yaml`, it now admits Traefik from `kube-system` (k3s default) or `traefik`.
