# Home-lab Kubernetes cluster install

Bring-up runbook for a 3-node Talos Linux Kubernetes cluster on Raspberry Pi 4.

**Run all commands from the repository root.** Talos files live in `talos/`,
Kubernetes manifests/values in `k8s/`; every path below is relative to the root.

## Topology

- Control plane nodes: `192.168.50.11`, `192.168.50.12`, `192.168.50.13` (pi1/pi2/pi3)
- Kubernetes API VIP / endpoint: `192.168.50.10`
- Node IPs and hostnames are set via DHCP reservations on the router.

```bash
export CLUSTER_NAME=picluster
```

## 1. Generate secrets and base configs

```bash
# Cluster PKI/secrets bundle -> talos/secrets.yaml. Keep this safe and back it
# up privately: it's the root of trust and is gitignored.
talosctl gen secrets -o talos/secrets.yaml

# Base machine configs (controlplane.yaml, worker.yaml, talosconfig) into talos/.
# The endpoint is the VIP, so kubeconfig/talosconfig point at 192.168.50.10:6443.
talosctl gen config --with-secrets talos/secrets.yaml \
  "$CLUSTER_NAME" https://192.168.50.10:6443 -o talos
```

## 2. Patch the control plane config

```bash
# Merge controlplane-patch.yaml (SD-card install, DHCP + VIP networking,
# EPHEMERAL/longhorn SSD volumes, and CNI/kube-proxy disabled for Cilium)
# into the base config -> talos/cp.yaml.
talosctl machineconfig patch talos/controlplane.yaml \
  --patch @controlplane-patch.yaml -o talos/cp.yaml
```

## 3. Wipe SSDs and apply config

```bash
# Wipe the SSDs (sda) while nodes are in maintenance mode, BEFORE applying.
# New SSDs ship preformatted (one full-disk partition) and re-installs have old
# Talos partitions; wiping first guarantees free space so EPHEMERAL/longhorn
# provision and the node boots straight to 'running'.
# (sda = the 1TB SSD; mmcblk0 is the SD card - do NOT wipe that.)
talosctl -n 192.168.50.11 wipe disk sda --insecure
talosctl -n 192.168.50.12 wipe disk sda --insecure
talosctl -n 192.168.50.13 wipe disk sda --insecure

# Apply cp.yaml to each node. --insecure because the nodes are still in
# maintenance mode (no trusted config yet). Each installs Talos to the SD card
# and reboots into the configured system.
talosctl apply-config --insecure -n 192.168.50.11 -f talos/cp.yaml
talosctl apply-config --insecure -n 192.168.50.12 -f talos/cp.yaml
talosctl apply-config --insecure -n 192.168.50.13 -f talos/cp.yaml
```

## 4. Configure talosctl

```bash
# Merge the generated client config into your default talosconfig (~/.talos/config).
talosctl config merge talos/talosconfig

# Point talosctl at the control plane nodes (Talos API, port 50000).
talosctl config endpoint 192.168.50.11 192.168.50.12 192.168.50.13
talosctl config node 192.168.50.11
```

## 5. Bootstrap etcd

```bash
# Control plane nodes stay in STAGE=booting until etcd is bootstrapped - that's
# expected, do NOT wait for 'running' here. Just confirm the node is reachable
# and EPHEMERAL provisioned:
#   talosctl -n 192.168.50.11 get volumestatus EPHEMERAL   # PHASE=ready
#   talosctl -n 192.168.50.11 get machinestatus            # STAGE=booting is fine

# Bootstrap etcd on ONE node only, over the Talos API (not the K8s VIP).
# After this, the node progresses booting -> running and the others join.
talosctl bootstrap -n 192.168.50.11
```

## 6. Kubernetes access

```bash
# Merge cluster admin kubeconfig into ~/.kube/config (points at the K8s API VIP).
talosctl kubeconfig -n 192.168.50.11

# Nodes will be NotReady and CoreDNS Pending until the CNI is installed - this is
# expected because the bundled CNI/kube-proxy are disabled in the patch.
kubectl get nodes
```

## 7. Install Cilium (CNI + kube-proxy replacement)

```bash
# Values (Talos capabilities, cgroup handling, KubePrism endpoint) in
# k8s/cilium-values.yaml.
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium -n kube-system -f k8s/cilium-values.yaml

# Nodes go Ready once Cilium is running:
#   kubectl get nodes
#   kubectl -n kube-system get pods -l k8s-app=cilium
#   kubectl -n kube-system exec ds/cilium -- cilium-dbg status | grep -i kubeproxy
```
