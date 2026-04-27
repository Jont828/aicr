# AKS H100 Dynamo Recipe Update

## Summary

This document describes the changes made to bring the `h100-aks-ubuntu-inference-dynamo` recipe in sync with a working AKS ND-H100 cluster running Dynamo. The cluster was used as the reference state — each change maps to something observed running on that cluster.

**Recipe changed:** `h100-aks-ubuntu-inference-dynamo` (via `aks.yaml` base overlay)

---

## RDMA / InfiniBand Stack

AKS ND-H100 nodes have ConnectX-7 InfiniBand NICs. The full RDMA stack requires five layers, each handled by a different component.

### Layer 1 — NFD NodeFeatureRule (`nfd-network-rule.yaml`)

**What:** A `NodeFeatureRule` CR that labels IB-capable nodes with `feature.node.kubernetes.io/pci-15b3.present=true`.

**Why:** The NicClusterPolicy (layer 2) uses this label as a nodeAffinity selector so its DaemonSets only run on ND-series nodes, not CPU nodes. The rule targets ConnectX-7 PCI device IDs `101c` and `101e` (vendor 15b3 = Mellanox/NVIDIA).

**How:** Delivered as a manifest file at `recipes/components/network-operator/manifests/nfd-network-rule.yaml`, applied with Helm hook weight 1 so it runs before the NicClusterPolicy (hook weight 5).

The chart's built-in `nfd.deployNodeFeatureRules` targets generic Ethernet device classes (0200/0207); AICR's rule targets ConnectX-7 specifically. Set `nfd.deployNodeFeatureRules: false` in `values-aks.yaml` to suppress the broader match.

### Layer 2 — NicClusterPolicy (`nic-cluster-policy-aks.yaml`)

**What:** A `NicClusterPolicy` CR that tells the network-operator to deploy the RDMA stack on labeled nodes.

**Why:** The network-operator Helm chart installs only the CRD and the operator — it does not template a NicClusterPolicy. You create the CR and the operator reconciles it by deploying DaemonSets. On the working cluster this CR managed the MOFED driver (ofedDriver) and DOCA telemetry. The RDMA shared device plugin was a separate standalone DaemonSet; this PR folds it into the same CR.

**Contents:**

- `ofedDriver` — MOFED (DOCA) kernel driver. Version `doca3.2.0-25.10-1.2.8.0-2`. Includes the `OFED_BLACKLIST_MODULES_FILE` env var as a workaround for a kernel module race condition on AKS ([upstream issue](https://github.com/Mellanox/doca-driver-build/issues/52)).
- `rdmaSharedDevicePlugin` — RDMA device plugin. Exposes the `rdma/hca_shared_devices_a` resource (resource name preserved from the standalone DaemonSet the cluster was using). `rdmaHcaMax: 1000` allows up to 1000 concurrent pod allocations per node. Targets vendor `15b3` (Mellanox), driver `mlx5_core`.
- `docaTelemetryService` — DOCA telemetry sidecar. Matches what was on the cluster.

**How:** Delivered as a manifest file at `recipes/components/network-operator/manifests/nic-cluster-policy-aks.yaml`, applied with Helm hook weight 5.

### Layer 3 — ib-node-config DaemonSet (`ib-node-config-aks.yaml`)

**What:** A DaemonSet that runs host-level IB setup: loads `ib_umad`, `rdma_ucm`, `ib_ucm` kernel modules, and sets `LimitMEMLOCK=infinity` on the containerd and kubelet systemd units.

**Why:** Without `LimitMEMLOCK=infinity` on the containerd unit, every container on the node inherits the default MEMLOCK limit (typically 64 KB). GPUDirect RDMA requires pinning large memory regions; pods would fail with permission errors unless they requested `IPC_LOCK`. Setting it at the node daemon layer means pods inherit unlimited MEMLOCK from containerd automatically — confirmed via live pod test (`cat /proc/self/limits` showed `RLIMIT_MEMLOCK: unlimited`).

**Note on Skyhook:** The right long-term fix is to use Skyhook to apply these host settings declaratively. Skyhook does not yet support AKS (as of this writing). A `# TODO: Remove once Skyhook gains AKS support` comment marks this file.

**Node selector change:** The working cluster used `kubernetes.azure.com/agentpool: ndh100pool` (cluster-specific pool name). This recipe uses `feature.node.kubernetes.io/pci-15b3.present: "true"` instead — portable across any AKS cluster with IB NICs, regardless of node pool name.

### Layer 4 — GPU Operator: `useHostMofed` (`values-aks.yaml`)

**What:** `driver.rdma.useHostMofed: true` in the GPU operator AKS values.

**Why:** By default, GPU operator builds `nvidia-peermem` from source at runtime. With MOFED installed by network-operator (layer 2), the GPU operator should use the host-installed MOFED headers instead. Builds are flaky and slow; `useHostMofed: true` skips the build and links against the pre-installed MOFED.

`nfd.enabled: false` is also set to prevent duplicate NFD DaemonSets — network-operator deploys NFD, and a second instance from GPU operator would conflict.

**Other commented flags:** `migManager`, `vgpuDeviceManager`, `vfioManager`, `sandboxDevicePlugin` are set in the aks-rdma-infiniband reference but are not required for RDMA. They are included as commented-out values with a link to the reference repo.

### Layer 5 — Network Operator version pin

**What:** `version: "v26.1.0"` in `aks.yaml`.

**Why:** Matches the version running on the working cluster. The `rdmaSharedDevicePlugin` image tag in the NicClusterPolicy is `network-operator-v26.1.0` — these must match.

---

## Dynamo Platform: v0.9 → v1.0.1

### Why upgrade

Dynamo v1.0 restructured subchart controls under `global.*` keys and made the operator responsible for injecting the NATS address into DynamoGraphDeployments. The v0.9 values file would silently apply to the wrong keys in v1.0, producing a broken deployment.

### `dynamo-crds` component removed

Dynamo v1.0 bundles CRD management in the platform chart itself (`upgradeCRD` behavior). A separate `dynamo-crds` component at v0.9 alongside a `dynamo-platform` at v1.0 would install incompatible CRDs. The `dynamo-crds` componentRef is removed from `h100-aks-ubuntu-inference-dynamo.yaml`.

**Note:** `dynamo-crds` remains in `registry.yaml` unchanged — other non-AKS overlays may still reference it at v0.9.

### Values rewrite (`components/dynamo-platform/values.yaml`)

**What changed vs v0.9:**

| Setting | v0.9 | v1.0.1 | Reason |
|---------|------|--------|--------|
| `dynamo-operator.controllerManager.manager.image.tag` | `"0.9.0"` | removed | Chart default is correct |
| `dynamo-operator.controllerManager.kubeRbacProxy.image` | gcr.io override | removed | Upstream fixed gcr.io deprecation |
| `grove.enabled` / `kai-scheduler.enabled` (top-level) | present | removed | Moved to `global.*` in v1.0 |
| `global.grove.install` | — | `true` | Dynamo manages grove; no separate AICR grove component |
| `global.kai-scheduler.install` | — | `false` | AICR installs kai-scheduler as a base component |
| `global.kai-scheduler.enabled` | — | `true` | Tells Dynamo operator to use the external AICR-managed instance |
| `global.etcd.install` | — | `false` | Kubernetes-native discovery; etcd not needed |
| `dynamo-operator.nats.enabled: false` | present | removed | NATS IS used in v1.0 — operator injects natsAddress into workloads |
| Prometheus endpoint | `kube-prometheus-prometheus:9090` | `kube-prometheus-kube-prome-prometheus:9090` | Service name derives from `fullnameOverride: kube-prometheus` in AICR's kube-prometheus-stack values |

### NATS JetStream storage (`h100-aks-ubuntu-inference-dynamo.yaml`)

NATS JetStream requires persistent storage for the message log. The `managed-csi` storage class override is kept in the overlay overrides so NATS uses Azure Disk Standard SSD (same storage class as Prometheus).

### `registry.yaml`

`dynamo-platform` defaultVersion bumped from `0.9.1` → `1.0.1`.

---

## Version Summary

| Component | Before | After |
|-----------|--------|-------|
| gpu-operator | unversioned | v26.3.0 |
| network-operator | unversioned | v26.1.0 |
| kube-prometheus-stack | unversioned | 83.7.0 |
| dynamo-platform | 0.9.1 (registry default) | 1.0.1 |

---

## Out-of-Band Items (not in this recipe)

| Item | Why out-of-band |
|------|----------------|
| Azure Lustre CSI | AKS managed addon with cluster-specific endpoint. Install via `az aks update`. |
| MPI operator | Only needed for NCCL bandwidth tests, not for Dynamo inference. |
| AKSInfinibandSupport feature flag | Infrastructure provisioning (`az feature register`), not recipe scope. |

---

## References

- [Azure aks-rdma-infiniband reference](https://github.com/Azure/aks-rdma-infiniband)
- [Azure network-operator configuration guide](https://azure.github.io/aks-rdma-infiniband/configurations/network-operator)
- [DOCA driver build race condition workaround](https://github.com/Mellanox/doca-driver-build/issues/52)
