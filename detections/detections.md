# Detections — mapping the #4 attacks to runtime signals

This is the payoff of the portfolio loop: the attacks documented in
[offensive-writeups](../../offensive-writeups) (#4) are **detected** here.

| #4 attack | Detected by | What you see | How to reproduce |
|-----------|-------------|--------------|------------------|
| **Recon / nmap** (writeup 01) | **Cilium / Hubble** | connection attempts + `DROPPED` verdicts for anything outside the allowed port | `hubble observe --namespace juice-shop` |
| **Egress / exfil attempt** | **Cilium / Hubble** | `DROPPED` egress flows (default-deny egress) | `hubble observe --verdict DROPPED` |
| **Post-exploit shell / tools** (after writeup 02) | **Falco** | `Suspicious process in container` alert (sh, nc, sqlmap, curl…) | `kubectl exec` a tool in a pod → `make falco-events` |
| **Attacker installs tooling** | **Falco** | `Package manager in container` alert | run `apk add` in a pod |
| **Credential access** | **Falco** | `Sensitive file read` alert (`/etc/shadow`, kubeconfig) | `cat /etc/shadow` in a pod |
| **App-layer HTTP abuse** | **Cilium L7 / Hubble** | method + path of every request (L7 parsing) | `hubble observe --protocol http` |

## The two sensors, and why both

- **Cilium / Hubble** = the **network** eye. Identity-aware flow visibility and
  L7 (HTTP) parsing — sees recon, blocked egress, and *which* requests hit the
  app. This is what flannel could not give us.
- **Falco** = the **host/syscall** eye. Sees what happens *inside* a container —
  a shell spawned, a package manager run, a sensitive file read — i.e. the
  post-exploitation an attacker does after a web vuln like the SQLi in #4.

Network detection alone misses an in-container shell; syscall detection alone
misses a port scan. Together they cover the attack from both sides.

## Try it

```bash
# network side
make hubble-ui                       # http://localhost:12000
# or: hubble observe --namespace juice-shop --follow

# host side
make falco-events                    # tail alerts
# then, in another shell, simulate post-exploitation in a throwaway pod:
kubectl -n juice-shop run t --image=busybox:1.36 --restart=Never -- sh -c 'sleep 60'
kubectl -n juice-shop exec t -- nc -w2 1.1.1.1 443   # -> Falco: suspicious tool + Hubble: DROPPED
```
