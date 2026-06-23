# runtime-security

> Project **#5** of the DevSecOps portfolio: **detect** the attacks from #4 at
> runtime. Cilium + Hubble watch the **network**; Falco watches **syscalls**
> inside containers. Deployed into the **same** lab cluster via ArgoCD.

This closes the loop: #1 builds & isolates the target, #4 attacks it, and #5
catches those attacks from both the network and host sides.

## What's here

| Piece | Role | Delivered by |
|-------|------|--------------|
| **Cilium** (CNI) | identity-aware networking, replaces flannel | bootstrap in #1 (`make cilium`) |
| **Hubble** | network flow + L7 (HTTP) observability | comes with Cilium |
| **Falco** | syscall-level runtime detection | **ArgoCD** (`gitops/falco.yaml`) |
| **Cilium L7 policy** | HTTP-aware visibility/control on Juice Shop | **ArgoCD** (`gitops/cilium-l7.yaml`) |
| **Detections** | maps each #4 attack to its signal | [detections/](detections/detections.md) |

## Why Cilium is bootstrapped (not via ArgoCD)
A CNI must exist before ArgoCD's pods can get IPs, so it can't be deployed by
ArgoCD. `make up` (in #1) installs Cilium first; **everything else here is
GitOps** via a cross-repo app-of-apps entry in #1. Full reasoning in
[docs/architecture.md](docs/architecture.md).

## Run it

```bash
# in secure-k8s-lab (#1): recreate the cluster Cilium-based + sync everything
make down && make up

# network observability
make hubble-ui        # http://localhost:12000

# runtime detection
make falco-events     # tail Falco alerts
```

Then reproduce a #4 attack and watch it light up — see
[detections/detections.md](detections/detections.md).

## Layout

```
runtime-security/
├── gitops/                  # ArgoCD Applications (synced from #1's app-of-apps)
│   ├── falco.yaml           # Falco (upstream chart + our values, multi-source)
│   └── cilium-l7.yaml       # -> policies/
├── falco/
│   └── values.yaml          # driver + JSON output + custom rules (#4 detections)
├── policies/
│   └── l7-visibility.yaml   # Cilium L7 (HTTP) policy for Juice Shop
├── detections/
│   └── detections.md        # attack -> detection map
└── docs/
    └── architecture.md
```

## ⚠️ Where Falco actually works (verified empirically)

| Runtime | Falco result |
|---------|--------------|
| **Docker Desktop** | **Crashes** — LinuxKit kernel has no raw-tracepoint BPF (`BPF_TRACE_RAW_TP`) |
| **Colima + k3d** *(this setup)* | **Runs & loads our rules**, but k3d runs nodes *as containers*, so Falco isn't in the host PID namespace and syscall capture is limited |
| **Real nodes** | **Full detection** — Colima's built-in k3s (`colima start --kubernetes`), minikube with a VM driver, or any cloud node |

This is a **k3d nesting limitation, not a config bug** — Falco loads and
schema-validates the custom rules either way; they fire fully on a real node.
**Cilium, Hubble and the L7 policy work fully** regardless (network observability
is unaffected). So on this lab the network half (Hubble) is the live demo; the
syscall half (Falco) is proven to deploy & load, and runs fully on a real node.

These sensors detect activity **only inside the isolated lab** — same scope
rules as the rest of the portfolio.
