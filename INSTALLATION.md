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

> Note: `cilium-values.yaml` enables L2 announcements, so a fresh `helm install`
> already has them. If you instead change the file and `helm upgrade` an existing
> install, restart the agents to load the new config:
> `kubectl -n kube-system rollout restart ds/cilium`.

## 8. LoadBalancer IP pool (Cilium L2)

```bash
# Pool of LAN IPs (192.168.50.20-.29) for type=LoadBalancer services, announced
# on the LAN via ARP. See k8s/cilium-lb-pool.yaml.
kubectl apply -f k8s/cilium-lb-pool.yaml
```

## 9. Install Argo CD

```bash
# Exposed on the LAN via a Cilium LoadBalancer IP pinned to 192.168.50.20
# (see k8s/argocd-values.yaml).
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace \
  -f k8s/argocd-values.yaml --timeout 10m

# Wait for the LoadBalancer IP and pods, then log in:
#   kubectl -n argocd get svc argocd-server          # EXTERNAL-IP 192.168.50.20
#   kubectl -n argocd get pods
```

Access: <https://192.168.50.20> (self-signed cert), user `admin`. Initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Change the admin password after first login, then delete the bootstrap secret:
`kubectl -n argocd delete secret argocd-initial-admin-secret`.

## 10. Cloudflare Tunnel (cloudflared)

Per-cluster connector for exposing apps publicly via Cloudflare. Token-based
(remotely-managed): public hostname -> service routing is configured in the
Cloudflare Zero Trust dashboard, not in this repo. The connector is managed by
Argo CD (see step 11); only the token secret is created imperatively.

```bash
# Create the tunnel token secret (token from Cloudflare Zero Trust > Networks >
# Tunnels). Not in git; Argo CD does not manage it. The namespace is created by
# the Argo CD Application, so create it here first if it doesn't exist yet.
kubectl create namespace cloudflared --dry-run=client -o yaml | kubectl apply -f -
kubectl -n cloudflared create secret generic cloudflared-token \
  --from-literal=token='<YOUR_TUNNEL_TOKEN>'

# Verify connectors registered (after Argo CD syncs the deployment):
#   kubectl -n cloudflared get pods
#   kubectl -n cloudflared logs -l app=cloudflared | grep -i registered
```

## 11. Argo CD GitOps (repo access + apps)

This repo is private, so Argo CD needs read access. Register it with a GitHub
fine-grained PAT (read-only Contents on this repo). The PAT stays out of git.

```bash
# Create the repository credential secret (replace <YOUR_PAT>).
kubectl -n argocd create secret generic repo-home-lab \
  --from-literal=type=git \
  --from-literal=url=https://github.com/brampling/home-lab.git \
  --from-literal=username=brampling \
  --from-literal=password='<YOUR_PAT>'
kubectl -n argocd label secret repo-home-lab \
  argocd.argoproj.io/secret-type=repository
```

Apply the app-of-apps root once. It watches `k8s/apps/` and manages every
Application defined there, so apps are added/updated by commit alone afterward:

```bash
kubectl apply -f k8s/apps-root.yaml
#   kubectl -n argocd get applications
```

No apps are exposed by default. Expose a service publicly only when explicitly
intended: add a public hostname in the dashboard (Tunnel > Public Hostname)
mapping to its in-cluster service, and protect it with a **Cloudflare Access**
policy. Internal-only tools (e.g. the Argo CD admin UI) stay off the tunnel and
are reached via their LAN LoadBalancer IP instead.

## 12. Longhorn (storage)

Longhorn requires the `iscsi-tools` + `util-linux-tools` Talos extensions on
every node. These are baked into the install image (see `controlplane-patch.yaml`
`machine.install.image`) via a Talos Image Factory schematic:

```yaml
# Schematic (factory.talos.dev) -> id f8a903f1...:
overlay:
    image: siderolabs/sbc-raspberrypi
    name: rpi_generic
customization:
    systemExtensions:
        officialExtensions:
            - siderolabs/iscsi-tools
            - siderolabs/util-linux-tools
```

For a FRESH install, download the RPi SD boot image from the factory using this
schematic so the extensions are present from first boot. To add them to an
existing cluster, upgrade each node (one at a time):

```bash
talosctl upgrade -n <node> \
  --image factory.talos.dev/installer/f8a903f101ce10f686476024898734bb6b36353cc4d41f348514db9004ec0a9d:v1.13.3
#   talosctl -n <node> get extensions   # confirm iscsi-tools + util-linux-tools
```

Longhorn itself is GitOps-managed (`k8s/apps/longhorn.yaml`, Helm chart). With
the app-of-apps root running, committing that file deploys it; data lands on the
`/var/mnt/longhorn` user volume.

```bash
#   kubectl -n longhorn-system get pods
#   kubectl get storageclass        # 'longhorn' (set as default by the chart)
```
