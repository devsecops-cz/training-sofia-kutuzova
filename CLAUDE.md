# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

The "objednávky" (orders) service: a tiny Go HTTP app plus the Kubernetes, ArgoCD and monitoring setup that deploys it to a local practice cluster (`onb`) in two environments, dev and prod. It is a DevOps training repo; the owner is learning, so changes should be explainable, not just correct.

## Stack

| Layer | What | Where |
|---|---|---|
| App (local exercise) | Go 1.27, stdlib only; Docker multi-stage build (alpine, non-root uid 10001); Docker Compose with `sklad` (podinfo) as a dependency | `main.go`, `Dockerfile`, `compose.yaml`, `start.sh` |
| App (in the cluster) | `ghcr.io/stefanprodan/podinfo:6.15.0` as a stand-in, port 9898, `/readyz`, `/healthz` | `base/deployment.yaml` |
| Manifests | Kustomize: base + overlays (dev, prod) | `infrastructure/kubernetes/apps/objednavky/` |
| GitOps | ArgoCD v3.5.6, AppProject `training`, one Application per environment, auto-sync with prune + selfHeal | `infrastructure/argocd/` |
| Git server for ArgoCD | soft-serve inside the cluster (remote `cluster`) | — |
| Monitoring | kube-prometheus-stack chart 92.13.1 (Prometheus v3.15.0, Alertmanager v0.34.1), ServiceMonitor, PrometheusRule, Grafana dashboard as ConfigMap | `infrastructure/kubernetes/monitoring/values.yaml`, `base/`, `overlays/prod/dashboards/` |
| CI | GitHub Actions: promtool, amtool, gitleaks, trivy | `.github/workflows/ci.yml` |

The cluster does **not** run the Go app from this repo; `main.go`/`Dockerfile`/`compose.yaml` are only for the local Docker exercise.

## Layout

```
infrastructure/
  argocd/
    bootstrap/
      argocd/            ArgoCD install: namespace + upstream install.yaml v3.5.6
      appproject.yaml    project "training": only namespaces objednavky-dev / objednavky-prod,
                         only Namespace as a cluster-scoped kind, only the soft-serve repo
    environments/
      dev/application.yaml   Application objednavky-dev  -> overlays/dev,  branch main
      prod/application.yaml  Application objednavky-prod -> overlays/prod, branch main
  kubernetes/
    apps/objednavky/
      base/              shared: Deployment, Service, ConfigMap, PDB, ServiceMonitor,
                         PrometheusRule, migrace Job (ArgoCD PreSync hook)
      overlays/dev/      namespace objednavky-dev, 3 replicas, PrometheusRule patch
      overlays/prod/     namespace objednavky-prod, 3 replicas, image tag pin,
                         PrometheusRule patch, Grafana dashboard ConfigMap
    monitoring/values.yaml   Helm values for kube-prometheus-stack (install command in header)
docs/runbooks/objednavky.md  runbook per alert; alerts link here via runbook_url
```

How the pieces connect: ArgoCD reads `main` of the **`cluster`** remote, renders `overlays/<env>` with Kustomize and applies it to `objednavky-<env>`. Prometheus picks up ServiceMonitor and PrometheusRule via the label `release: monitoring`; the ServiceMonitor finds the Service by `app: objednavky`, port name `http`. Alertmanager sends alerts with `team = "objednavky"` to the `pager` webhook. Grafana is disabled in the chart; the dashboard ConfigMap carries `grafana_dashboard: "1"`.

### Two remotes

| Remote | URL | Role |
|---|---|---|
| `origin` | GitHub `devsecops-cz/training-sofia-kutuzova` | PRs, CI, runbook links in alerts |
| `cluster` | `ssh://localhost:23231/training` | what ArgoCD deploys from |

Merging on GitHub does not deploy. Pushing to `cluster` `main` deploys to dev **and** prod at once.

## Conventions

- **Czech names** for resources, rules, comments and alert texts (`migrace`, `pravidla`, `zaznamy` = recording rules, `alerty`, `sklad`). The runbook body is in English, headings in Czech (`Příznak`, `Diagnostika`, `Náprava`, `Ověření`, `Eskalace`). Keep each file's existing language.
- **Commits:** Conventional Commits, optional scope: `feat(monitoring): …`, `fix: …`, `ci: …`. GitHub appends the PR number on merge.
- **Branches:** `<type>/<topic>` (`feat/`, `fix/`, `ci/`, `test/`); `ai/<topic>` for AI-assisted work.
- **Pin every version:** container images by tag (CI images and actions also by SHA digest), ArgoCD manifests by release tag, Helm chart by `--version`. CI tool versions must match the cluster.
- **Comments explain why**, one line, next to the setting (see `application.yaml`, `ci.yml`).
- **Every new base file goes into `base/kustomization.yaml`** (missed twice before, see `fix: missing files from kustomization.yaml`).
- **Alert rules:** edit `base/prometheusrule.yaml`; each overlay's `prometheusrule-patch.yaml` (JSON6902) rewrites every `expr` to its namespace and adds a `namespace` label to alerts. Patches address rules **by index** (`/spec/groups/<g>/rules/<n>`), so adding, removing or reordering rules in base means updating both patches. Dev and prod patches differ only in the namespace name.
- **Alerts** carry `severity` and `team: objednavky`; critical alerts have a `runbook_url` pointing to an anchor (alert name) in `docs/runbooks/objednavky.md`. Renaming an alert means renaming the runbook heading too.
- **No secrets in Git.** Secret `objednavky-db` (key `DB_PASSWORD`) is used by the Deployment but not defined in this repo. `.env` is git-ignored.

## Gates

A change passes these, in order:

| # | Gate | What it checks | Where |
|---|---|---|---|
| 1 | Local render | `kubectl kustomize` of both overlays succeeds and shows what you expect | your machine |
| 2 | PR to `main` | Ruleset "main protection": PR required (0 approvals), no force-push, no deleting `main` | GitHub |
| 3 | **`ci-gate`** | Required status check. Passes only if both jobs below succeeded | `ci.yml` |
| 3a | `monitoring` | Extracts `.spec` from the base rules and the rendered dev/prod rules, plus Alertmanager config from `values.yaml`; fails on empty files; `promtool check rules`, `amtool check-config` | `ci.yml` |
| 3b | `security` | gitleaks over full Git history; trivy fs (vuln + secret, HIGH/CRITICAL fails) | `ci.yml` |
| 4 | Push to `cluster` | The actual deploy; done by a person, not CI | soft-serve |
| 5 | `migrace` PreSync Job | Runs before every sync; `backoffLimit: 0`, so one failure blocks the sync | ArgoCD |
| 6 | Pod readiness + PDB | `readinessProbe` gates traffic; PDB keeps at least 1 pod during disruptions | cluster |

Adding a CI job: add it to `ci-gate`'s `needs` **and** add a step checking its result, otherwise it doesn't gate anything.

## Commands

Render what ArgoCD will apply:
```
kubectl kustomize infrastructure/kubernetes/apps/objednavky/overlays/dev
kubectl kustomize infrastructure/kubernetes/apps/objednavky/overlays/prod
```

Compare dev and prod output (today they differ only in namespace and the prod dashboard ConfigMap; the prod image pin equals the base tag, 6.15.0):
```
diff <(kubectl kustomize infrastructure/kubernetes/apps/objednavky/overlays/dev) <(kubectl kustomize infrastructure/kubernetes/apps/objednavky/overlays/prod)
```

Reproduce the CI `monitoring` job locally (needs `yq`, `kubectl`, Docker). `out/` is scratch, not committed:
```
mkdir -p out
yq '.spec' infrastructure/kubernetes/apps/objednavky/base/prometheusrule.yaml > out/rules-base.yml
kubectl kustomize infrastructure/kubernetes/apps/objednavky/overlays/dev | yq 'select(.kind == "PrometheusRule") | .spec' > out/rules-dev.yml
kubectl kustomize infrastructure/kubernetes/apps/objednavky/overlays/prod | yq 'select(.kind == "PrometheusRule") | .spec' > out/rules-prod.yml
yq '.alertmanager.config' infrastructure/kubernetes/monitoring/values.yaml > out/alertmanager.yml
docker run --rm -v "$PWD/out":/c -w /c --entrypoint "" quay.io/prometheus/prometheus:v3.15.0 promtool check rules rules-base.yml rules-dev.yml rules-prod.yml
docker run --rm -v "$PWD/out":/c -w /c --entrypoint "" quay.io/prometheus/alertmanager:v0.34.1 amtool check-config alertmanager.yml
```

Run the Go app locally with `sklad`: `docker compose up --build`, then http://localhost:18080 (needs a `.env` file).

Check what ArgoCD sees in the cluster (read-only):
```
kubectl -n argocd get applications
kubectl -n objednavky-prod get deploy,pods
```

## Danger zones

| Path / action | Why it's dangerous |
|---|---|
| **`overlays/prod/`** | Production. Auto-sync with `prune: true`: removing a resource here deletes it from prod. Holds the image tag pin, replicas and the dashboard. Change prod only on purpose, never as a side effect of a dev change. |
| `base/` | Affects dev **and** prod at the same time. |
| `prometheusrule.yaml` + both `prometheusrule-patch.yaml` | Index-based patches: a reordered rule silently gets the wrong `expr`/label, or the patch fails to apply. |
| Pushing to the `cluster` remote | Deploys immediately to both environments. Only when the owner explicitly asks. |
| `kubectl apply/edit/delete` in `objednavky-*` | `selfHeal` reverts it within minutes; it hides the real fix. Fix in Git. |
| `infrastructure/argocd/` | Project and Application settings: wrong `path`, `targetRevision` or allowed namespaces break or redirect deploys for everything. |
| `base/migrace.yaml` | PreSync hook with `backoffLimit: 0`: if it fails, no sync happens in either environment. |
| `monitoring/values.yaml` | Alert routing to the pager; a bad route means alerts go nowhere. Applied by `helm upgrade`, not ArgoCD. |
| `.github/workflows/ci.yml` | Weakening `ci-gate` or a scanner removes a gate for everyone. Keep SHA pins. |
| Secrets | Never write secret values into files, commits or chat; gitleaks scans all history, so a leaked secret stays there. |

## How the assistant should work here

- **Git is the source of truth.** Every change goes through a branch and a PR to `main` on `origin`. Never commit to `main` directly, never force-push, never push to `cluster` unless the owner asks for that specific push.
- **Read-only in the cluster by default.** `get`, `describe`, `logs` are fine; anything that changes the cluster needs the owner's yes.
- **Verify before claiming.** Run `kubectl kustomize` (and the local CI checks for rule or Alertmanager changes) before saying a change works. Check the live state (`gh pr list`, `git fetch`, the cluster) rather than trusting local files, old notes or error messages. An empty result may mean the check itself is broken.
- **Check what exists before adding** a workflow, rule or file: look at `main` on GitHub and open PRs first.
- **Say which environment** a change affects (dev, prod, both). Flag anything that touches `overlays/prod/` or `base/` explicitly.
- **Keep changes small and in scope.** Side findings go on a "parked" list, not into the current change.
- **Explain, don't just do.** The owner is learning DevOps: say what each command and change does and why, give short commands one at a time, and leave decisions to them.
