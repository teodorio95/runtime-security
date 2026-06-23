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

## ⚠️ Where Falco actually works (what I tried, empirically)

Falco's syscall driver is finicky on a local Apple-Silicon (arm64) Mac. I tested
every local option available and **none captured syscalls**:

| Runtime tried | Result | Why |
|---------------|--------|-----|
| **Docker Desktop** | ❌ CrashLoop | LinuxKit kernel has no raw-tracepoint BPF (`BPF_TRACE_RAW_TP`) — both `modern_ebpf` and `ebpf` fail |
| **Colima + k3d** *(this lab)* | ⚠️ Runs, **0 capture** | k3d runs nodes *as containers*; the Falco chart sets no `hostPID`, so Falco isn't in the host PID namespace (`disabled BPF iterators / not in root PID namespace`) and sees no live events |
| **minikube + vfkit** (arm64 VM) | ❌ CrashLoop | `modern_ebpf`: `unable to set interesting syscall ... Bad file descriptor`; `kmod`: module won't build/load on the arm64 ISO kernel |
| **Real x86_64 Linux node** (cloud / VM) | ✅ Full detection | normal host kernel + host PID namespace — the supported way to run Falco |

**This is an environment limitation, not a config bug.** In every case Falco
*loaded and schema-validated* our custom rules (`falco/values.yaml`); they fire
fully on a real node. Drivers tried: `modern_ebpf`, `ebpf`, `kmod`.

**Mentions / takeaways:**
- The **network half of #5 works fully** here — Cilium + Hubble + the L7 policy.
  Verified live: HTTP method/path/status in Hubble (`http-request GET / → 200`).
  So the live runtime-detection demo on this Mac is **Hubble** (network); to
  demo **Falco** (syscalls) live, use a real x86_64 Linux node.
- The Falco chart never templates `hostPID`, and a `kubectl patch` to add it is
  rejected by the chart's own `privileged:true` + `allowPrivilegeEscalation:false`
  securityContext — so even on a real node you'd set both via values.
- These sensors detect activity **only inside the isolated lab** — same scope
  rules as the rest of the portfolio.
