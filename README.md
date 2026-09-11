# kubernetes-gitops

A GitOps-managed Kubernetes platform reference: namespaces, network policy,
resource governance, and a sample workload — all declared in git, all
deployed by ArgoCD. This is the pattern I ran at Verizon and IBM: every
change is a pull request, every deploy is an ArgoCD sync, and the mean time
from merge-to-running dropped ~30% versus the old ticket-driven kubectl
workflow because there is no human in the deploy path to page.

## Repository layout

```
kubernetes-gitops/
├── bootstrap/                  # applied first: namespaces with env labels
│   └── namespaces.yaml         # platform, monitoring, apps-dev, apps-prod
├── platform/
│   ├── networkpolicies/
│   │   ├── default-deny-all.yaml  # zero-trust default for the platform ns
│   │   └── allow-dns.yaml         # explicit CoreDNS (UDP/TCP 53) exception
│   └── resourcequotas/
│       ├── namespace-quotas.yaml  # CPU/mem/pod ceilings per app namespace
│       └── limitrange.yaml        # default requests/limits + max guardrails
├── apps/
│   └── sample-api/             # the reference workload (stateless HTTP API)
│       ├── base/               # raw manifests: deploy, svc, hpa, ingress,
│       │                       # serviceaccount, pdb, networkpolicy
│       └── overlays/
│           ├── dev/            # thin shim -> kustomize/overlays/dev
│           └── prod/           # thin shim -> kustomize/overlays/prod
├── kustomize/
│   ├── base/                   # re-exports apps/sample-api/base
│   └── overlays/
│       ├── dev/                # 2 replicas, lean resources, HPA min 1
│       └── prod/               # 5 replicas, prod resources, HPA min 5, PDB 4
├── argocd/
│   └── app-of-apps.yaml        # ApplicationSet: auto-discovers apps/*
├── README.md
├── LICENSE
└── .gitignore
```

## How the app-of-apps flow works

1. `argocd/app-of-apps.yaml` defines a single ArgoCD **ApplicationSet** in the
   `argocd` namespace. Its `git` generator scans the `apps/*` directories of
   this repo on the `main` branch.
2. For each directory it templates one ArgoCD **Application**
   (`apps-<name>-prod`) whose source is `apps/<name>/overlays/prod` and whose
   destination is the `apps-prod` namespace on the local cluster.
3. Each Application has an **automated sync policy** with `prune: true` and
   `selfHeal: true`:
   - **prune** — resources deleted from git are deleted from the cluster.
   - **selfHeal** — manual `kubectl` edits are reverted to the git state.
   - Retry with backoff (5 attempts) absorbs transient API flakes.
4. Result: pushing to `main` is the deploy. Nothing else runs `kubectl apply`.

## How to add a new app

Copy the `sample-api` pattern — no changes to the ApplicationSet needed:

```bash
cp -r apps/sample-api apps/<new-app>
# edit apps/<new-app>/base/*.yaml: names, image, probes, ports
git add apps/<new-app> && git commit -m "feat: onboard <new-app>" && git push
```

ArgoCD discovers `apps/<new-app>` on its next repo refresh (default 3
minutes) and creates + syncs `apps-<new-app>-prod` automatically. The
per-app `overlays/<env>/kustomization.yaml` shims point at
`kustomize/overlays/<env>`, so new apps inherit the environment sizing
until you give them their own patches.

## How to promote dev → prod

Dev and prod are the same base with different overlays — promotion is a
config change, not a rebuild:

1. Validate the change in dev first:
   `kustomize build kustomize/overlays/dev | kubectl diff -f -`
2. Make the same edit to the **base** manifest in `apps/<app>/base/`
   (or add an env-specific patch under `kustomize/overlays/prod/` if the
   change is prod-only).
3. Open a PR, get review, merge to `main`. ArgoCD syncs prod.
4. Watch it: `argocd app get apps-sample-api-prod` or the ArgoCD UI.

Image tags are the one thing that changes per deploy — tag images
immutably in CI (`:1.4.2`, never `:latest`) and bump the tag in
`apps/<app>/base/deployment.yaml` via PR, same as any other change.

## Prerequisites

- A Kubernetes cluster (v1.27+; tested against kustomize v5 APIs)
- **ArgoCD** installed (`argocd` namespace), with this repo registered
- **ingress-nginx** installed (`ingress-nginx` namespace) for the sample Ingress
- **cert-manager** with a `letsencrypt-prod` ClusterIssuer for TLS
  (or delete the `tls:` block and cert-manager annotation from
  `apps/sample-api/base/ingress.yaml`)
- `kubectl`, `kustomize` (v5+), and the `argocd` CLI

## Quickstart

```bash
# 1. Point the ApplicationSet at your fork
sed -i 's|example-org/kubernetes-gitops|YOUR-ORG/kubernetes-gitops|' \
  argocd/app-of-apps.yaml

# 2. Set your real hostname in the sample Ingress
sed -i 's|api.example.com|api.yourdomain.com|' \
  apps/sample-api/base/ingress.yaml

# 3. Sanity-check what ArgoCD will render (no cluster needed)
kustomize build kustomize/overlays/dev
kustomize build kustomize/overlays/prod

# 4. Install the app-of-apps (bootstrap namespaces first if ArgoCD
#    sync-waves aren't covering them yet)
kubectl apply -f bootstrap/namespaces.yaml
kubectl apply -f argocd/app-of-apps.yaml

# 5. Watch ArgoCD pick up the app
argocd appset list
argocd app get apps-sample-api-prod
argocd app sync apps-sample-api-prod   # usually unnecessary: sync is automated

# 6. Verify the workload
kubectl -n apps-prod get deploy,svc,hpa,ingress,pdb
kubectl -n apps-prod get networkpolicy
```

## Design notes

- **Zero-trust networking**: `default-deny-all` + explicit allows. The sample
  API only accepts ingress from the ingress-nginx controller and only
  egresses to CoreDNS. Any database/cache the app needs gets its own
  explicit `NetworkPolicy` — this is how you pass a FedRAMP-style audit
  without a last-minute scramble.
- **Resource governance**: `ResourceQuota` caps each app namespace and
  `LimitRange` gives forgetful containers sane defaults, so one bad
  manifest can't starve a node or blow up the cloud bill.
- **Availability by default**: pod anti-affinity across nodes and zones,
  `maxUnavailable: 0` rolling updates, PDBs, and HPA on CPU *and* memory
  with patient scale-down. The deployment is boring on purpose — boring
  is what stays up at 3 AM.
- **No `:latest`, no hand-applied YAML**: if it isn't in git, it doesn't
  exist. `selfHeal` enforces that.

## License

MIT — see [LICENSE](LICENSE).
