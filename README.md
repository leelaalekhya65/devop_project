
> Open **`docs/index.html`** in a browser for the visual walkthrough: Docker, Kubernetes, Observability and Logs each have their own page.

## Architecture

```
git push -> GitHub Actions CI (test, lint, build)
         -> CD: push image to GHCR, bump tag in helm/webapp/values.yaml
         -> Argo CD detects the commit and syncs
         -> Kubernetes (kind) rolls out the new version

webapp --/metrics--> Prometheus --> Grafana
webapp --JSON logs-> Promtail ----> Loki ----> Grafana
```

## Repository layout

| Path | Contents |
|------|----------|
| `app/` | Flask service with Prometheus metrics, JSON logs, tests, Dockerfile |
| `docker-compose.yml` | App, load generator, Prometheus, Loki, Promtail, Grafana |
| `helm/webapp/` | Helm chart: Deployment, Service, HPA, Ingress, ServiceMonitor |
| `argocd/` | AppProject, root "app of apps", webapp, kube-prometheus-stack, loki-stack |
| `k8s/` | kind config, namespace quota and network policies |
| `observability/` | Prometheus config and alerts, Loki, Promtail, Grafana provisioning and dashboard |
| `terraform/` | Local kind cluster, ingress-nginx and Argo CD |
| `ansible/` | Roles to provision Docker, kubectl, kind, Helm and SSH keys on a host |
| `.github/workflows/` | `ci.yml` (test, lint, build) and `cd.yml` (publish and GitOps bump) |
| `docs/` | Static HTML documentation |
| `scripts/` | Key generation and GitHub push helpers |

## Prerequisites

Docker, `make`, and for the Kubernetes path: Terraform, kubectl, Helm. (Ansible can install the Kubernetes tools for you.)

## Quick start (Docker Compose)

```bash
cp .env.example .env      # or: make keys   (also generates an SSH key pair)
make up
```

| Service | URL |
|---------|-----|
| App | http://localhost:8080 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |

Grafana login comes from `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` in `.env`.

## Kubernetes path (Terraform + Argo CD)

```bash
make tf-init && make tf-apply            # kind cluster + ingress-nginx + Argo CD
export KUBECONFIG=$(cd terraform && terraform output -raw kubeconfig_path)

# point Argo CD at your fork
sed -i 's|REPO_URL|https://github.com/<you>/devops-platform.git|' argocd/root-app.yaml argocd/apps/webapp.yaml
git commit -am "Set repo URL" && git push
make argocd-bootstrap
```

Argo CD UI: `kubectl -n argocd port-forward svc/argocd-server 8082:443`, then https://localhost:8082. The initial admin password is printed by the `argocd_admin_password_cmd` Terraform output.

## Secrets and keys

All secrets live in `.env`, which is **git-ignored**. `.env.example` documents every variable and is the only env file committed.

- `make keys` runs `scripts/gen-keys.sh`, which generates an ed25519 key pair and stores both halves (base64) as `SSH_PUBLIC_KEY_B64` and `SSH_PRIVATE_KEY_B64` in `.env`. No key files are left on disk.
- `.gitignore` also blocks `*.pem`, `*.key`, `id_*`, Terraform state and kubeconfigs.
- CI/CD uses GitHub's built-in `GITHUB_TOKEN` for GHCR; add any extra secrets under repository Settings, Secrets.

## Ansible

```bash
# set ANSIBLE_HOST / ANSIBLE_USER in .env (defaults to localhost)
make ansible
```

## Make targets

Run `make help` for the full list.

## CI/CD

- **CI** (`ci.yml`) on pull requests and pushes: pytest, `helm lint`, `terraform fmt/validate`, `ansible-lint`, Docker build.
- **CD** (`cd.yml`) on merges that change `app/`: build and push to `ghcr.io`, then commit the new image tag into `helm/webapp/values.yaml`. Argo CD does the actual deployment.

## Pushing to GitHub

```bash
./scripts/push-to-github.sh <your-github-username> devops-platform
```
