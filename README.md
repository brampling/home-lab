# home-lab

Infrastructure-as-code for a self-hosted Kubernetes cluster running on
[Talos Linux](https://www.talos.dev/) on a trio of Raspberry Pi 4s.

The repo holds the declarative config needed to rebuild the cluster from scratch.
The full step-by-step bring-up is in **[installation.md](installation.md)**.

## Cluster overview

| | |
|---|---|
| Hardware | 3 × Raspberry Pi 4 |
| OS | Talos Linux |
| Role | 3 × control plane (also schedulable) |
| Boot disk | SD card (`/dev/mmcblk0`) |
| Data disk | 1 TB USB SSD (`/dev/sda`) |
| CNI | Cilium (eBPF, kube-proxy replacement, service mesh) |
| Load balancer | Cilium LB-IPAM + L2 announcements, pool `192.168.50.20`–`.29` |
| GitOps | Argo CD @ `https://192.168.50.20` |
| API endpoint | VIP `192.168.50.10:6443` (floats across nodes) |
| Nodes | pi1/pi2/pi3 @ `192.168.50.11`–`.13` (DHCP reservations) |

### Storage layout (per node, on the SSD)

| Volume | Size | Mount | Purpose |
|---|---|---|---|
| EPHEMERAL | 100 GiB | `/var` | container images, kubelet ephemeral storage, logs |
| longhorn | 400 GiB | `/var/mnt/longhorn` | user data / replicated PVs (Longhorn) |
| _unallocated_ | ~460 GiB | — | reserved for future volumes |

The SD card holds only the Talos system (STATE/META).

## Repository layout

```
.
├── installation.md          # end-to-end bring-up runbook (run from repo root)
├── controlplane-patch.yaml  # Talos machine config patch for the control plane
├── k8s/
│   ├── cilium-values.yaml    # Cilium Helm values (tuned for Talos)
│   ├── cilium-lb-pool.yaml   # LoadBalancer IP pool + L2 announcement policy
│   └── argocd-values.yaml    # Argo CD Helm values (LAN LoadBalancer)
└── talos/                   # generated Talos configs + secrets (gitignored)
```

## Prerequisites

- [`talosctl`](https://www.talos.dev/latest/talos-guides/install/talosctl/)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/)
- [`helm`](https://helm.sh/docs/intro/install/)
- 3 Pis booted from the Talos SBC image, reachable on the LAN (maintenance mode)

## Quick start

See **[installation.md](installation.md)** for the annotated runbook. In short,
from the repo root: generate configs → patch → wipe SSDs & apply → bootstrap →
pull kubeconfig → install Cilium.

## Secrets

The Talos PKI and generated machine configs live in `talos/` and are
**gitignored** — they contain CA keys, tokens, and client certs and must never
be committed. `talos/secrets.yaml` is backed up in **LastPass**; combined with
`controlplane-patch.yaml` it can regenerate the rest of the cluster config.
