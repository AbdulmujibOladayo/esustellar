# Kubernetes manifests

Two manifest trees exist:

| Tree | Role |
| ---- | ---- |
| `infra/k8s/base` + `infra/k8s/overlays/{mainnet,staging,testnet}` | Kustomize-based, authoritative for the web app. CI renders each overlay and checks Service selectors against Deployment labels (`k8s-overlay-validation.yml`). |
| `k8s/{namespaces,deployments,config,web,monitoring}` | Flat per-environment manifests (namespaces, config maps, placeholder secrets, monitoring). |

## Namespaces (#1232)

| Environment | `k8s/namespaces/*.yaml` | Kustomize overlay |
| ----------- | ----------------------- | ----------------- |
| testnet | `esustellar-testnet` | `esustellar-testnet` |
| staging | `esustellar-staging` | `esustellar-staging` |
| mainnet | `esustellar-mainnet` | `esustellar` (differs; `infra/k8s/network-policies.yaml` also targets `esustellar`) |
| monitoring | n/a | `esustellar-monitoring` |

The mainnet mismatch is existing behaviour; changing a live namespace requires a
planned migration, so it is documented rather than altered here.

## Flat deployments vs base (#1231)

`k8s/deployments/*.yaml` define resource requests/limits and readiness and
liveness probes, matching the structure of `infra/k8s/base/deployment.yaml`.
Keep resource values and probe paths in sync when editing either tree.

## `environments/testnet/k8s` (#1233)

`environments/testnet/` contains no `k8s/` directory, so there is no duplicate
of `infra/k8s/overlays/testnet`; the overlay is the only source of truth.

## Network policy to monitoring mapping (#1297)

`esustellar` runs a default-deny ingress policy. Scraping still works because
`infra/k8s/network-policies.yaml` contains an explicit exception:

| Policy | Namespace / selector | Port | Consumer |
|---|---|---|---|
| `deny-all-ingress` | all pods in `esustellar` | — | baseline deny |
| `allow-monitoring` | pods labelled `app: esustellar-web` from namespace `esustellar-monitoring` | 9464 | Prometheus `esustellar-web` scrape job |
| `allow-only-api-egress` monitoring leg | to namespace `esustellar-monitoring` (all ports) | any | return path for scrapes and Alertmanager traffic |

Prometheus's `kubernetes_sd_configs` is scoped to `namespaces: [esustellar]` and
its relabel rule rewrites every target to port `9464`, so the policy port and the
scrape port agree.

Verification after any policy change (do not assume the manifest is enough):

```bash
kubectl -n esustellar get netpol allow-monitoring -o yaml
kubectl -n esustellar-monitoring run netpol-probe --rm -it --image=curlimages/curl -- \
  curl -sS --max-time 5 http://esustellar-web.esustellar.svc.cluster.local:9464/metrics
kubectl -n esustellar-monitoring port-forward svc/prometheus 9090:9090
# check the `esustellar-web` target is UP at http://localhost:9090/targets
```

If the probe times out, check the namespace label
`kubernetes.io/metadata.name=esustellar-monitoring` and the pod label
`app=esustellar-web` before changing the policy.
