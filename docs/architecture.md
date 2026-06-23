# Architecture & design decisions

## One cluster, not two
Project #5 runs on the **same** k3d cluster as project #1 — no separate
environment. It upgrades that cluster's networking to Cilium and adds Falco.

## Why Cilium is bootstrapped, not GitOps'd
A CNI must exist **before** any pod (including ArgoCD's own) can get an IP — so
it can't be deployed *by* ArgoCD (chicken-and-egg). k3s also ships flannel + a
network-policy controller, and you can't run two CNIs at once. Therefore:

- `secure-k8s-lab/cluster/k3d-config.yaml` disables flannel + the netpol
  controller (`--flannel-backend=none`, `--disable-network-policy`).
- `make cilium` (called by `make up`) installs Cilium via Helm **before** ArgoCD.
- Everything else — **Falco**, **Cilium L7 policies** — is deployed **by ArgoCD**,
  pulled from this repo through a cross-repo app-of-apps entry in #1
  (`apps/argocd-apps/runtime-security.yaml`).

## What each sensor covers
- **Cilium / Hubble** — identity-aware **network** observability + L7 (HTTP)
  visibility and policy. Replaces flannel; this is the capability #1 noted it was
  missing. See [detections/](../detections/detections.md).
- **Falco** — **syscall-level** runtime detection inside containers (shells,
  package managers, sensitive-file reads) — the post-exploitation half.

## GitOps wiring
```
#1 root (app-of-apps)
└── runtime-security (Application -> this repo's gitops/)
    ├── falco            (Application: upstream chart + our values/rules)
    └── cilium-l7-policies (Application: this repo's policies/)
```
Falco uses an ArgoCD **multi-source** Application: the upstream chart plus our
`falco/values.yaml` referenced via `$rs`.

## Runtime caveats (honest notes)
- **Falco on Docker Desktop — known dead end (verified):** the LinuxKit kernel
  lacks raw-tracepoint BPF (`BPF_TRACE_RAW_TP`), which both `modern_ebpf` and
  `ebpf` require, and `kmod` can't build without kernel headers — so Falco
  CrashLoops here. The rules still load & schema-validate (config is correct);
  run on Colima/Lima, minikube with a VM driver, or a cloud node for a working
  syscall sensor.
- **Cilium on k3d:** runs with kube-proxy kept (no `kubeProxyReplacement`) for
  simplicity; that's enough for CNI + Hubble + policies in a lab.
