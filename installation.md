# Control plane node IPs (.11/.12/.13) and the K8s API VIP (.10).
# Set via DHCP reservations on the router; the VIP floats across the nodes.
export CONTROL_PLANE_IP=("192.168.50.11" "192.168.50.12" "192.168.50.13")

export YOUR_ENDPOINT=192.168.50.10

# Generate the cluster PKI/secrets bundle (secrets.yaml). Keep this safe -
# it's the root of trust for the cluster and is reused by gen config below.
talosctl gen secrets -o secrets.yaml

export CLUSTER_NAME=picluster

# Generate base machine configs (controlplane.yaml, worker.yaml, talosconfig)
# from the secrets bundle. The endpoint is the VIP, so the kubeconfig/talosconfig
# point at https://192.168.50.10:6443.
talosctl gen config --with-secrets secrets.yaml $CLUSTER_NAME https://192.168.50.10:6443

# Merge our control plane patch (SD-card install, DHCP + VIP networking, and the
# EPHEMERAL/longhorn SSD volume layout) into the base config -> cp.yaml.
# See controlplane-patch.yaml for the details of what's being patched.
talosctl machineconfig patch controlplane.yaml --patch @../controlplane-patch.yaml -o cp.yaml

# Wipe the SSDs (sda) while the nodes are still in maintenance mode, BEFORE
# applying config. New SSDs ship preformatted with one partition filling the
# whole disk, leaving no free space for Talos to create EPHEMERAL/longhorn; and
# on a re-install the disk has old Talos partitions. Wiping first guarantees a
# clean disk so the volumes provision and the node boots straight to 'running'.
# (sda = the 1TB SSD; mmcblk0 is the SD card - do NOT wipe that.)
talosctl -n 192.168.50.11 wipe disk sda --insecure
talosctl -n 192.168.50.12 wipe disk sda --insecure
talosctl -n 192.168.50.13 wipe disk sda --insecure

# Apply cp.yaml to each control plane node. --insecure is required here because
# the nodes are still in maintenance mode (no trusted config yet). Each node
# installs Talos to the SD card and reboots into the configured system.
talosctl apply-config --insecure -n 192.168.50.11 -f cp.yaml

talosctl apply-config --insecure -n 192.168.50.12 -f cp.yaml

talosctl apply-config --insecure -n 192.168.50.13 -f cp.yaml

# Step 12: merge the generated client config into your default talosconfig
talosctl config merge ./talosconfig

# Step 13: point talosctl at the control plane nodes (Talos API, port 50000)
talosctl config endpoint 192.168.50.11 192.168.50.12 192.168.50.13
talosctl config node 192.168.50.11

# NOTE: control plane nodes stay in STAGE=booting until etcd is bootstrapped -
# that's expected, do NOT wait for 'running' here (it won't happen until after
# bootstrap). Just confirm the node is reachable and EPHEMERAL provisioned:
#   talosctl -n 192.168.50.11 get volumestatus EPHEMERAL   # PHASE=ready
#   talosctl -n 192.168.50.11 get machinestatus            # STAGE=booting is fine

# Bootstrap etcd on ONE node only, over the Talos API (not the K8s VIP).
# Endpoints/node are configured above, so no --insecure / -e flags needed.
# After this, the node progresses booting -> running and the rest join.
talosctl bootstrap -n 192.168.50.11

# After bootstrap, pull kubeconfig (this one uses the K8s API VIP):
talosctl kubeconfig -n 192.168.50.11

# Nodes will be NotReady and CoreDNS Pending until a CNI is installed - this is
# expected because the bundled CNI/kube-proxy are disabled in the patch.

# Install Cilium as the CNI + kube-proxy replacement (run from the repo root).
# Values (Talos capabilities, cgroup handling, KubePrism endpoint) live in
# k8s/cilium-values.yaml.
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium -n kube-system -f ../k8s/cilium-values.yaml

# Watch it come up; nodes should go Ready once Cilium is running:
#   cilium status --wait        # if the cilium CLI is installed
#   kubectl get nodes
#   kubectl -n kube-system get pods -l k8s-app=cilium