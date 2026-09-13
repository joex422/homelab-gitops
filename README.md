# homelab-gitops

Everything ArgoCD watches for the prod k8s cluster. App-of-apps pattern:
one Application (`root-prod`) is applied by hand, once — everything else
gets discovered and synced automatically from then on.

```
bootstrap/{prod,dev}/root.yaml   — app-of-apps root Application per cluster
apps/{prod,dev}/{infrastructure,workloads}/
                                 — one folder per component, each holding
                                   its own application.yaml (+ raw manifests
                                   if it needs any, e.g. metallb-config/)
projects/                       — ArgoCD AppProjects (permission boundary:
                                   which repos/namespaces an Application
                                   is allowed to use)
```

## One-time bootstrap (only needed once, on a brand-new cluster)

```bash
export KUBECONFIG=~/.kube/config

kubectl apply -f projects/prod.yaml
kubectl apply -f bootstrap/prod/root.yaml
```

That's it. `root-prod` recursively scans `apps/prod/` for any file named
`application.yaml` and creates it as a real Application. From here on,
**you never run `kubectl apply` for a new service — you just commit.**

## Deploying something new

1. Create `apps/prod/{infrastructure,workloads}/<name>/application.yaml`
   (an ArgoCD `Application` pointing at a Helm chart, or at a path in this
   repo, or another repo entirely)
2. If it needs raw manifests alongside it (like `metallb-config/ipaddresspool.yaml`),
   put them in the same folder — `root-prod`'s `include: "**/application.yaml"`
   filter only picks up files literally named `application.yaml`, so the
   raw manifests don't get double-applied
3. `git add . && git commit -m "..." && git push`
4. ArgoCD picks it up on its next poll (a few minutes), or force it immediately:

```bash
kubectl -n argocd annotate application root-prod \
  argocd.argoproj.io/refresh=hard --overwrite
```

## Sync waves (ordering dependencies between components)

Set via `argocd.argoproj.io/sync-wave: "N"` annotation on the child
Application (lower runs first). Only matters when one component's CRDs
must exist before another's resources can apply — e.g. `metallb` (wave 20)
must install its `IPAddressPool` CRD before `metallb-config` (wave 21) can
create one. Note this only orders *when Applications get created*, not
when they finish becoming healthy — see troubleshooting below for what
happens when that assumption doesn't hold.

## Useful commands

Check everything's status at a glance:
```bash
kubectl -n argocd get applications
```

Get the actual error for something stuck:
```bash
kubectl -n argocd get application <name> -o jsonpath='{.status.conditions}'
```

If a new namespace's resources fail with `namespaces "X" not found`, the
Application is missing `CreateNamespace=true`:
```yaml
syncPolicy:
  automated: { prune: true, selfHeal: true }
  syncOptions:
    - CreateNamespace=true
```

## Troubleshooting a genuinely stuck sync

Sometimes a sync operation gets wedged retrying against a stale git
revision (e.g. right after a fix was pushed) and manual `kubectl patch`
attempts on `.operation` don't clear it reliably. The clean reset:

```bash
kubectl -n argocd delete application <name>
kubectl -n argocd annotate application root-prod \
  argocd.argoproj.io/refresh=hard --overwrite
```

Deleting the child Application is safe — it's declarative, so `root-prod`
immediately recreates it fresh from whatever's currently in git, with a
clean retry counter. (This does **not** delete whatever the Application
already deployed, unless that Application also carries the cascade
finalizer — check `metadata.finalizers` before relying on that.)

## Secrets

Never committed here. Anything requiring a credential (e.g. `cloudflared`'s
tunnel token) is created directly with `kubectl create secret` and
referenced by name from the Deployment/Application — the actual secret
value lives only in the cluster, never in git.

**Watch out when an Application's `destination.namespace` changes**: a
Secret doesn't move with it. If you retarget a component to a new
namespace, recreate its secrets there too, or it'll fail with
`CreateContainerConfigError` even though the Application itself deploys
fine.
