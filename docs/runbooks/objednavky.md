# Runbook: objednávky

- Team: objednavky (alerts with `team: objednavky` go to the pager)
- Runs as: Deployment `objednavky`, deployed by ArgoCD from Git (self-heal and prune on)

  | Environment | Namespace | ArgoCD Application | Overlay in Git |
  |---|---|---|---|
  | dev | `objednavky-dev` | `objednavky-dev` | `infrastructure/kubernetes/apps/objednavky/overlays/dev` |
  | prod | `objednavky-prod` | `objednavky-prod` | `infrastructure/kubernetes/apps/objednavky/overlays/prod` (3 replicas) |

- Commands below use `objednavky-prod`; for dev swap in `objednavky-dev`.
- Git: training repo; ArgoCD syncs from the Git server in the cluster (remote `cluster`). Shared parts are in `infrastructure/kubernetes/apps/objednavky/base`.
- ArgoCD UI: Applications → `objednavky-prod` (sync status, history, last commit)
- Dashboard: Grafana → Dashboards → Objednávky (uid `objednavky`)
- Prometheus: Alerts, Status → Target health, Query
- Logs: `kubectl -n objednavky-prod logs deploy/objednavky`

**Rule:** the cluster always returns to what Git says. A fix made only with `kubectl` is undone by ArgoCD self-heal within minutes. Fix in Git, or pause auto-sync first (see Náprava, emergency).

## ObjednavkyNedostupne

### Příznak

No objednavky pod in the environment answers Prometheus for 1 minute: the service is completely down. Customers cannot place or view orders. Severity: critical (total outage).

### Diagnostika

1. What does ArgoCD say? ArgoCD UI → `objednavky-prod`.

   `OutOfSync` → the cluster differs from Git (someone changed it by hand, or sync failed). `Degraded` / `Progressing` → pods are not healthy (continue). Look at **Last Sync**: a sync just before the outage → a new commit is the likely cause; note the commit.

2. What changed in Git recently?

   ```
   git log --oneline -5
   ```

   Run in the training repo. A commit touching `objednavky` (image tag, replicas, limits, ConfigMap) just before the outage → suspect it.

3. Does the Deployment want any pods, and how many are ready?

   ```
   kubectl -n objednavky-prod get deploy objednavky
   ```

   `READY 0/0` → replicas are 0 in Git (or set by hand). `READY 0/3` → pods exist but are not ready (continue).

4. What state are the pods in?

   ```
   kubectl -n objednavky-prod get pods
   ```

   `CrashLoopBackOff` → the app crashes (steps 5–6). `Pending` → no room on the nodes (step 5, Events). `ImagePullBackOff` → wrong image tag in Git.

5. Why? Read the pod's Events at the bottom.

   ```
   kubectl -n objednavky-prod describe pod <pod-name>
   ```

   Look for `OOMKilled`, `Liveness probe failed`, `FailedScheduling`, `Failed to pull image`.

6. What did the app log before it died?

   ```
   kubectl -n objednavky-prod logs deploy/objednavky --previous
   ```

   Errors at startup point to bad config (ConfigMap `objednavky-config`) or a bad release.

7. Prometheus → Status → Target health → `serviceMonitor/objednavky-prod/objednavky/0`. No targets = no ready pods; DOWN = pods run but `/metrics` does not answer.

### Náprava

Mitigate first, find the cause later. Every fix goes through Git; ArgoCD applies it.

1. Bad commit (new image, config, limits) → revert it and push:

   ```
   git revert <commit>
   ```

   ```
   git push cluster main
   ```

   `git revert` makes a new commit that undoes the old one; history stays intact. ArgoCD syncs it within a few minutes (or press **Sync** in the UI).

2. Replicas 0 or too few → fix `replicas` in the overlay (`overlays/prod/kustomization.yaml`, `count: 3`), commit, push.

3. `OOMKilled` → raise `limits.memory` in `base/deployment.yaml` (temporary), commit, push. Find the leak after the incident.

4. Cluster problem (`Pending`, nodes `NotReady`) → not fixable in this repo, see Eskalace.

**Emergency: Git is too slow.** Pause ArgoCD first, otherwise it undoes your fix:

1. ArgoCD UI → `objednavky-prod` → Details → Sync Policy → **Disable Auto-Sync**.
2. Fix by hand, e.g. `kubectl -n objednavky-prod rollout undo deployment objednavky`, or ArgoCD UI → **History and Rollback** → previous revision.
3. Put the same fix into Git, push, then **Enable Auto-Sync** again. Do not leave auto-sync off.

### Ověření

1. ArgoCD UI → `objednavky-prod`: **Synced** and **Healthy**, on the commit with your fix.
2. Pods are ready again:

   ```
   kubectl -n objednavky-prod get deploy objednavky
   ```

   Expect `READY 3/3` in prod.

3. Prometheus → Status → Target health → `serviceMonitor/objednavky-prod/objednavky/0`: all targets UP.
4. Prometheus → Alerts: `ObjednavkyNedostupne` for this environment is inactive.
5. The pager log shows a notification with `"status": "resolved"`.

### Eskalace

If the service is not back within 15 minutes: notify the team `objednavky` lead and the platform team. If ArgoCD itself or the Git server does not work, or the cause is in the cluster (nodes `NotReady`, `FailedScheduling` everywhere), hand it over to the platform team.

## ObjednavkyPomale

### Příznak

The orders p95 latency (`objednavky:latence:p95_5m`, measured over 5 minutes) has been above 0.5 s for 2 minutes: at least every 20th request takes longer than half a second. The service is up but degraded: customers see slow pages and timeouts, and some orders may fail. Severity: critical.

### Diagnostika

1. How slow, and since when? Prometheus → Query → `objednavky:latence:p95_5m` → Graph. A value in seconds; normal is about 0.03. Note when it jumped.

2. More traffic than usual? Prometheus → Query → `objednavky:pozadavky:rate5m` → Graph. A spike at the same time → overload.

3. Was there a release at that time? ArgoCD UI → `objednavky-prod` → Last Sync / History, and in the training repo:

   ```
   git log --oneline -5
   ```

   A commit synced just before the slowdown → likely a bad release.

4. Are all pods ready, or are fewer pods carrying the load?

   ```
   kubectl -n objednavky-prod get pods
   ```

   Restarts or not-ready pods → see ObjednavkyNedostupne, Diagnostika steps 5–6.

5. Is a pod near its memory limit? Prometheus → Query → the expression of `ObjednavkyDochaziPamet` without `> 0.9` → Graph. Close to 1.0 → the pod is struggling (and may be OOMKilled soon).

### Náprava

Mitigate first, find the cause later. Every fix goes through Git.

1. Overload → more replicas: raise `count` in `overlays/prod/kustomization.yaml` (e.g. 3 → 5), commit, push.
2. Slow since a release → `git revert <commit>`, `git push cluster main` (see ObjednavkyNedostupne, Náprava step 1).
3. Pod near its memory limit → raise `limits.memory` in Git (see ObjednavkyNedostupne, Náprava step 3).
4. Cause outside the service (cluster CPU, database) → see Eskalace.

Emergency without Git: same as ObjednavkyNedostupne (pause auto-sync first).

### Ověření

1. ArgoCD UI → `objednavky-prod`: **Synced** and **Healthy**, on the commit with your fix.
2. Prometheus → Query → `objednavky:latence:p95_5m` → Graph: below 0.5 (normal about 0.03). It is a 5-minute window, so it drops a few minutes after the fix.
3. Prometheus → Alerts: `ObjednavkyPomale` is inactive.
4. The pager log shows a notification with `"status": "resolved"`.
5. If you raised replicas only for the incident, lower them back in Git once traffic is normal.

### Eskalace

If p95 stays above 0.5 s for 15 minutes after mitigation: notify the team `objednavky` lead. If the slowness comes from outside the service (cluster CPU, database, the Git server or ArgoCD), bring in the owner of that component or the platform team.
