# Talos Linux Kubernetes Cluster on Raspberry Pi 4

> **Highly available Kubernetes cluster on Talos Linux — 3 control planes + 3 workers on Raspberry Pi 4, with Virtual IP for API server redundancy. Immutable OS, no SSH, fully API-driven.**

![Talos](https://img.shields.io/badge/Talos-v1.12.6-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-latest-326CE5?logo=kubernetes)
![Architecture](https://img.shields.io/badge/arch-ARM64-lightgrey)
![CNI](https://img.shields.io/badge/CNI-Cilium-F8C517?logo=cilium)
![License](https://img.shields.io/badge/license-MIT-blue)

---

## Features

- **Highly available control plane** — 3 control plane nodes with shared Virtual IP via `etcd` leader election
- **Immutable OS** — Talos Linux: no SSH, no shell, no package manager — all management through `talosctl` API
- **Static networking** — each node has a fixed IP, hostname, DNS, and NTP configured via machine config patches
- **Virtual IP failover** — control plane VIP (`192.168.1.110`) automatically moves between healthy nodes
- **Cilium CNI with GatewayAPI** — replaces default Flannel and kube-proxy with Cilium for advanced networking
- **Declarative configuration** — all node configs generated from a single secrets file and per-node YAML patches
- **Reproducible setup** — patch files for all 6 nodes included in this repository

---

## Architecture

| Role | Hostname | IP | Description |
|------|----------|----|-------------|
| Control Plane 1 | `talos-cp-01` | `192.168.1.101` | etcd + API server |
| Control Plane 2 | `talos-cp-02` | `192.168.1.102` | etcd + API server |
| Control Plane 3 | `talos-cp-03` | `192.168.1.103` | etcd + API server |
| Worker 1 | `talos-wr-01` | `192.168.1.104` | Workload node |
| Worker 2 | `talos-wr-02` | `192.168.1.105` | Workload node |
| Worker 3 | `talos-wr-03` | `192.168.1.106` | Workload node |
| VIP | — | `192.168.1.110` | Kubernetes API endpoint (shared by control planes) |

```
                             ┌───────────────────────────┐
                             │  VIP  192.168.1.110:6443  │
                             └─────────────┬─────────────┘
               ┌───────────────────────────┼───────────────────────────┐
               ▼                           ▼                           ▼
      ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
      │ talos-cp-01 .101 │        │ talos-cp-02 .102 │        │ talos-cp-03 .103 │
      │ etcd + API       │        │ etcd + API       │        │ etcd + API       │
      └──────────────────┘        └──────────────────┘        └──────────────────┘
               │                           │                           │
               └───────────────────────────┼───────────────────────────┘
               ┌───────────────────────────┼───────────────────────────┐
               ▼                           ▼                           ▼
      ┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
      │ talos-wr-01 .104 │        │ talos-wr-02 .105 │        │ talos-wr-03 .106 │
      └──────────────────┘        └──────────────────┘        └──────────────────┘
```

---

## File Structure

```
repository/
├── patches/
│   ├── cilium.yaml                 # Cilium CNI — disables default CNI and kube-proxy
│   ├── controlplane-patch-1.yaml   # talos-cp-01 — 192.168.1.101
│   ├── controlplane-patch-2.yaml   # talos-cp-02 — 192.168.1.102
│   ├── controlplane-patch-3.yaml   # talos-cp-03 — 192.168.1.103
│   ├── worker-patch-1.yaml         # talos-wr-01  — 192.168.1.104
│   ├── worker-patch-2.yaml         # talos-wr-02  — 192.168.1.105
│   └── worker-patch-3.yaml         # talos-wr-03  — 192.168.1.106
├── diagram.png
└── README.md
```

---

## Prerequisites

- 6× Raspberry Pi 4 (4 GB or 8 GB RAM)
- SD cards (16 GB minimum per node)
- DHCP server for initial IP assignment
- [`talosctl`](https://www.talos.dev/v1.12/introduction/getting-started/#talosctl), [`kubectl`](https://kubernetes.io/docs/tasks/tools/), [`helm`](https://helm.sh/docs/intro/install/), and [`cilium` CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli) installed on your workstation
- Local workstation with access to the same LAN

---

## Step 1 — Update EEPROM on Each Raspberry Pi

This ensures the Pi boots correctly from SD card.

1. Flash using **Raspberry Pi Imager** → **OS > Misc Utility Images > Bootloader > SD Card Boot**
2. Insert the SD card and boot the device
3. Wait for rapid green LED blinking (~10 seconds), then power off

---

## Step 2 — Flash Talos Image

Download the Talos ARM64 image:

```bash
wget https://factory.talos.dev/image/ee21ef4a5ef808a9b7484cc0dda0f25075021691c8c09a276591eedb638ea1f9/v1.12.6/metal-arm64.raw.xz
```

Flash using **Raspberry Pi Imager** (select the `.raw.xz` file directly) or with `dd`:

```bash
xz -d metal-arm64.raw.xz
dd if=metal-arm64.raw of=/dev/sdX bs=4M status=progress
```

Boot all 6 nodes. DHCP assigns temporary IPs — note them for the next steps.

---

## Step 3 — Generate Cluster Secrets

```bash
talosctl gen secrets -o secrets.yaml
```

> Keep `secrets.yaml` safe — it contains PKI material for your entire cluster. Never commit it to version control.

---

## Step 4 — Generate Base Configs

Generate configs with the Cilium patch to disable the default CNI and kube-proxy:

```bash
talosctl gen config --with-secrets secrets.yaml democluster https://192.168.1.110:6443 \
  --config-patch @patches/cilium.yaml
```

This generates:

| File | Purpose |
|------|---------|
| `controlplane.yaml` | Base config for all control plane nodes (Flannel and kube-proxy disabled) |
| `worker.yaml` | Base config for all worker nodes (Flannel and kube-proxy disabled) |
| `talosconfig` | Client config for `talosctl` |

---

## Step 5 — Inspect Nodes

Before patching, verify the network interface name and disk path on each node:

```bash
# List network interfaces
talosctl --nodes <dhcp-ip> get links --insecure

# List available disks
talosctl --nodes <dhcp-ip> get disks --insecure
```

The patch files in this repo use `end0` (interface) and `/dev/mmcblk0` (SD card) — standard for Raspberry Pi 4.

---

## Step 6 — Apply Node Patches

Patch files for all 6 nodes are in the `patches/` directory. Apply them:

```bash
# Control planes
talosctl machineconfig patch controlplane.yaml --patch @patches/controlplane-patch-1.yaml --output controlplane-1.yaml
talosctl machineconfig patch controlplane.yaml --patch @patches/controlplane-patch-2.yaml --output controlplane-2.yaml
talosctl machineconfig patch controlplane.yaml --patch @patches/controlplane-patch-3.yaml --output controlplane-3.yaml

# Workers
talosctl machineconfig patch worker.yaml --patch @patches/worker-patch-1.yaml --output worker-1.yaml
talosctl machineconfig patch worker.yaml --patch @patches/worker-patch-2.yaml --output worker-2.yaml
talosctl machineconfig patch worker.yaml --patch @patches/worker-patch-3.yaml --output worker-3.yaml
```

---

## Step 7 — Apply Configurations to Nodes

Apply each patched config to the corresponding node using its DHCP-assigned IP:

```bash
talosctl apply-config --insecure --nodes <dhcp-ip-cp-01> --file controlplane-1.yaml
talosctl apply-config --insecure --nodes <dhcp-ip-cp-02> --file controlplane-2.yaml
talosctl apply-config --insecure --nodes <dhcp-ip-cp-03> --file controlplane-3.yaml
talosctl apply-config --insecure --nodes <dhcp-ip-w-01>  --file worker-1.yaml
talosctl apply-config --insecure --nodes <dhcp-ip-w-02>  --file worker-2.yaml
talosctl apply-config --insecure --nodes <dhcp-ip-w-03>  --file worker-3.yaml
```

Nodes will reboot and come up with their static IPs. Then merge the talosconfig:

```bash
talosctl config merge ./talosconfig
```

---

## Step 8 — Bootstrap the Cluster

Set the control plane endpoints and bootstrap etcd on the first node:

```bash
talosctl config endpoint 192.168.1.101 192.168.1.102 192.168.1.103
talosctl bootstrap --nodes 192.168.1.101
```

> Run `bootstrap` **once**, on **one** control plane only. Running it again will break the cluster.

---

## Step 9 — Install Cilium CNI with GatewayAPI

After bootstrapping, nodes will be in a `NotReady` state waiting for a CNI. Retrieve the kubeconfig first:

```bash
talosctl kubeconfig --nodes 192.168.1.101
```

Install Cilium with kube-proxy replacement and GatewayAPI support using Helm:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

helm install \
    cilium \
    cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set gatewayAPI.enabled=true \
    --set gatewayAPI.enableAlpn=true \
    --set gatewayAPI.enableAppProtocol=true
```

Verify Cilium is running:

```bash
cilium status --wait
```

---

## Step 10 — Access Kubernetes

Retrieve the kubeconfig (skip if already done in the previous step):

```bash
talosctl kubeconfig --nodes 192.168.1.101
```

If managing multiple clusters, save to a named file:

```bash
talosctl kubeconfig talos-cluster --nodes 192.168.1.101
export KUBECONFIG=./talos-cluster
```

---

## Verification

Check all nodes are ready:

```bash
kubectl get nodes -o wide
```

Expected output (all nodes in `Ready` state):

```
NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP
talos-cp-01   Ready    control-plane   5m    v1.xx.x   192.168.1.101
talos-cp-02   Ready    control-plane   5m    v1.xx.x   192.168.1.102
talos-cp-03   Ready    control-plane   5m    v1.xx.x   192.168.1.103
talos-wr-01   Ready    <none>          5m    v1.xx.x   192.168.1.104
talos-wr-02   Ready    <none>          5m    v1.xx.x   192.168.1.105
talos-wr-03   Ready    <none>          5m    v1.xx.x   192.168.1.106
```

---

## Troubleshooting

### Node does not come up after applying config

```bash
# Check kernel/boot messages
talosctl --nodes <node-ip> dmesg --insecure

# Verify the node accepted the config
talosctl --nodes <node-ip> get machineconfig --insecure
```

### etcd not forming quorum

```bash
# Check etcd member status
talosctl --nodes 192.168.1.101 etcd members

# Check etcd service health on a control plane
talosctl --nodes 192.168.1.101 service etcd
```

### VIP not responding

The VIP is assigned to whichever control plane currently holds the etcd leader lease. Verify at least one control plane is healthy:

```bash
talosctl --nodes 192.168.1.101 health
```

### kubectl cannot connect

```bash
# Confirm the API server is reachable via VIP
curl -k https://192.168.1.110:6443/healthz

# Re-merge talosconfig if context is missing
talosctl config merge ./talosconfig
```

---

## License

MIT — free to use, modify, and distribute.

---

## Author

**Przemysław Pradela**

[![GitHub](https://img.shields.io/badge/GitHub-ppradela-181717?logo=github)](https://github.com/ppradela)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Przemysław%20Pradela-0A66C2?logo=linkedin)](https://www.linkedin.com/in/przemyslaw-pradela)
[![Website](https://img.shields.io/badge/Website-pradela.ovh-4A90D9?logo=globe)](https://pradela.ovh)

Contributions and issue reports welcome.
