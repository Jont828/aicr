# Update AKS H100 Dynamo Inference Recipe

## Context

The existing `h100-aks-ubuntu-inference-dynamo` recipe is outdated compared to the working AKS ND-H100 cluster. Key gaps: GPU operator v25→v26, network operator v25→v26, dynamo v0.9→v1.0.1, missing RDMA shared device plugin, missing NicClusterPolicy (MOFED driver), missing NFD NodeFeatureRule for Mellanox. This plan updates the AKS recipe so `aicr recipe --service aks --accelerator h100 --intent inference --os ubuntu --platform dynamo` produces a bundle that can recreate the working cluster state.

## Key Design Decisions

### 1. NicClusterPolicy via manifest, not Helm values

The network-operator Helm chart does NOT template a NicClusterPolicy CR — it only installs the CRD and the operator. The current `deployCR: true` / `nicClusterPolicy.enabled: true` values in `recipes/components/network-operator/values.yaml` are dead config (confirmed: chart has no template for it, no AICR Go code references them).

A NicClusterPolicy is the network-operator's declarative config for "what networking DaemonSets should run on my IB nodes." You create the CR and the operator reconciles it by deploying DaemonSets. In our case:
- `ofedDriver` → deploys the MOFED driver DaemonSet
- `rdmaSharedDevicePlugin` → deploys the RDMA shared device plugin DaemonSet + ConfigMap
- `docaTelemetryService` → deploys the DOCA telemetry DaemonSet

On the working cluster, the MOFED part is managed by the NicClusterPolicy CR but the RDMA device plugin was deployed manually as a raw DaemonSet. This plan consolidates both into one CR, delivered as a Go-templated manifest file following the `skyhook-customizations` pattern.

### 2. Keep AICR's external kai-scheduler

- **kai-scheduler** = gang scheduling. Ensures all pods in a multinode deployment get scheduled together (all-or-nothing). Valuable for any GPU workload. AICR already installs it as a base component for all recipes.
- **grove** = multinode orchestration. Manages `PodCliqueSets` — coordinated groups of pods across nodes for distributed inference. This is the multinode-specific piece.

Dynamo values set `global.kai-scheduler.install: false, enabled: true` — AICR manages kai-scheduler as a separate base component. Grove is set to `install: true` since AICR has no separate grove component and it's essential for the dynamo multinode use case.

Both are overridable at bundle time via `--set`:
```bash
# Let Dynamo install its own kai-scheduler instead
aicr bundle -r recipe.yaml --set dynamoplatform:global.kai-scheduler.install=true

# Disable grove for single-node inference
aicr bundle -r recipe.yaml --set dynamoplatform:global.grove.install=false
```

If someone doesn't want multinode at all, they'd use `--intent inference` without `--platform dynamo` (the plain inference recipe doesn't include dynamo).

### 3. Leave dead config in base `values.yaml` with TODO comment

Don't remove `deployCR`/`nicClusterPolicy` from base network-operator values since it's unclear if other clouds rely on them or if they were an attempt to configure something we actually need. Add a TODO comment.

### 4. GPU operator: only add RDMA-critical flags

The only flags that matter for InfiniBand RDMA are:
- `driver.rdma.enabled=true` — already in base values
- `driver.rdma.useHostMofed=true` — missing, must add to AKS values
- `nfd.enabled=false` — prevents duplicate NFD DaemonSets (network-operator deploys its own)

Other flags on the working cluster (`migManager.enabled=false`, `sandboxDevicePlugin.enabled=false`, `vfioManager.enabled=false`, `vgpuDeviceManager.enabled=false`) are just noise reduction for ND-H100 VMs — not functionally required. Skip them.

### 5. AKS-scoped changes only

Other cloud dynamo overlays (EKS, GKE, etc.) are not touched in this change.

### 6. Out-of-band items (not in recipe)

| Item | Why out-of-band |
|------|----------------|
| **Azure Lustre CSI** | AKS managed addon, cluster-specific endpoint/subscription. Install via `az aks update`. |
| **MPI operator** | Provides `MPIJob` CRD for running NCCL all-reduce bandwidth tests across nodes. Dynamo doesn't use MPI for inference — it uses grove for multinode orchestration. Only needed for test/validation. |
| **ib-node-config DaemonSet** | Host-level MEMLOCK workaround (loads ib_umad/rdma_ucm modules, sets MEMLOCK=unlimited on containerd/kubelet). May not be needed with v26 operators. Test first. |
| **nvidia-peermem-reloader** | Loads nvidia-peermem kernel module. Likely redundant with `useHostMofed=true` on GPU operator v26.3.0. |
| **AKSInfinibandSupport feature flag** | Infrastructure provisioning (`az feature register`), not recipe scope. |

## Implementation Steps

### Step 1: Update GPU operator AKS values

**File:** `recipes/components/gpu-operator/values-aks.yaml`

Currently only sets `toolkit.enabled: false`. Add the two RDMA-critical settings:

```yaml
toolkit:
  enabled: false

# Use MOFED installed by network-operator instead of building nvidia-peermem.
# Required for GPUDirect RDMA on AKS ND-series.
driver:
  rdma:
    useHostMofed: true

# Network operator deploys NFD; disable GPU operator's own to avoid duplicate DaemonSets.
nfd:
  enabled: false
```

### Step 2: Create NicClusterPolicy manifest

**New file:** `recipes/components/network-operator/manifests/nic-cluster-policy-aks.yaml`

Go-templated manifest (follows `skyhook-customizations/manifests/tuning.yaml` pattern) that creates the NicClusterPolicy CR with:
- MOFED driver (doca3.2.0-25.10-1.2.8.0-2)
- RDMA shared device plugin (`shared_ib`, rdmaHcaMax=63, vendor 15b3, linkType infiniband)
- DOCA telemetry
- nodeAffinity targeting `pci-15b3.present=true`
- Helm post-install/post-upgrade hooks

Reference configs:
- `aks-rdma-infiniband/configs/nicclusterpolicy/base/nic.yaml` (MOFED + DOCA telemetry)
- `aks-rdma-infiniband/configs/nicclusterpolicy/rdma-shared-device-plugin/rdma.yaml` (RDMA device plugin)

### Step 3: Create NFD NodeFeatureRule manifest

**New file:** `recipes/components/network-operator/manifests/nfd-network-rule.yaml`

Go-templated manifest that creates a NodeFeatureRule for Mellanox PCI devices (IDs `101c`, `101e` = ConnectX-7 NICs). Labels nodes with `feature.node.kubernetes.io/pci-15b3.present=true`. This label is what the NicClusterPolicy nodeAffinity uses to target only IB-capable nodes.

Reference: `aks-rdma-infiniband/tests/setup-infra/network-operator-nfd.yaml` (also deployed on the working cluster by the setup script).

### Step 4: Create AKS network operator values file

**New file:** `recipes/components/network-operator/values-aks.yaml`

Minimal values for AKS — the NicClusterPolicy is a manifest, so values only configure the operator itself:

```yaml
operator:
  fullnameOverride: network-operator
  tolerations:
    - operator: Exists

nfd:
  enabled: false
  deployNodeFeatureRules: false    # AICR deploys targeted NFD rule via manifest

sriovNetworkOperator:
  enabled: false

# nvIpam and secondaryNetwork are for Ethernet secondary interfaces.
# Not needed for InfiniBand RDMA workloads.
nvIpam:
  enabled: false

secondaryNetwork:
  deploy: false
```

### Step 5: Add TODO comment to base network-operator values

**File:** `recipes/components/network-operator/values.yaml`

Add a comment noting `deployCR`, `nicClusterPolicy`, `nvIpam`, `secondaryNetwork` are not consumed by the upstream Helm chart and may need investigation.

### Step 6: Update `aks.yaml` overlay

**File:** `recipes/overlays/aks.yaml`

Changes to componentRefs:
- `gpu-operator`: add `version: "v26.3.0"`
- `network-operator`: version `"25.1.0"` → `"v26.1.0"`, valuesFile → `values-aks.yaml`, add `manifestFiles` for NicClusterPolicy and NFD rule
- `kube-prometheus-stack`: add `version: "83.7.0"`

```yaml
- name: gpu-operator
  type: Helm
  version: "v26.3.0"
  valuesFile: components/gpu-operator/values-aks.yaml

- name: network-operator
  type: Helm
  source: https://helm.ngc.nvidia.com/nvidia
  version: "v26.1.0"
  valuesFile: components/network-operator/values-aks.yaml
  manifestFiles:
    - components/network-operator/manifests/nic-cluster-policy-aks.yaml
    - components/network-operator/manifests/nfd-network-rule.yaml
  dependencyRefs:
    - cert-manager

- name: kube-prometheus-stack
  type: Helm
  version: "83.7.0"
  overrides:
    ...  # existing storage overrides unchanged
```

### Step 7: Update dynamo overlay

**File:** `recipes/overlays/h100-aks-ubuntu-inference-dynamo.yaml`

- `dynamo-platform` version: `"0.9.0"` → `"1.0.1"`
- Keep `dynamo-crds` at `"0.9.0"` (no 1.0 CRD chart exists; platform bundles CRDs via `upgradeCRD`)

### Step 8: Rewrite dynamo-platform values for v1.0.1

**File:** `recipes/components/dynamo-platform/values.yaml`

Per `docs/bugs/dynamo-v1.0-bump.md`. The 1.0 release restructured subchart controls under `global.*` keys:

```yaml
dynamo-operator:
  upgradeCRD: true
  discoveryBackend: "kubernetes"
  nats:
    enabled: false
  dynamo:
    metrics:
      prometheusEndpoint: "http://kube-prometheus-prometheus.monitoring.svc.cluster.local:9090"

global:
  grove:
    install: true           # Let Dynamo manage grove (no separate AICR grove component)
    enabled: true
  kai-scheduler:
    install: false           # AICR manages kai-scheduler as a separate base component
    enabled: true            # Tell Dynamo operator to use the external kai-scheduler
  etcd:
    install: false           # Kubernetes-native discovery replaces etcd

nats:
  enabled: false             # ZMQ event plane replaces NATS
```

**Removed from v0.9 values:**
- `dynamo-operator.controllerManager.manager.image.tag` — 1.0 chart default is correct
- `dynamo-operator.controllerManager.kubeRbacProxy.image` overrides — gcr.io deprecation fixed upstream
- Old top-level `grove.enabled`, `kai-scheduler.enabled` — moved to `global.*`

### Step 9: Update registry.yaml

**File:** `recipes/registry.yaml`

- `dynamo-platform` defaultVersion: `"0.9.1"` → `"1.0.1"`

## Verification

```bash
# 1. Unit tests
make test

# 2. Recipe compilation — verify bundle output
aicr recipe --service aks --accelerator h100 --intent inference \
  --os ubuntu --platform dynamo -o /tmp/recipe.yaml

# 3. Bundle generation — inspect outputs
aicr bundle -r /tmp/recipe.yaml -o /tmp/bundles
cat /tmp/bundles/gpu-operator/values.yaml        # verify useHostMofed, nfd disabled
cat /tmp/bundles/network-operator/values.yaml     # verify AKS values
ls /tmp/bundles/network-operator/manifests/       # verify NicClusterPolicy + NFD rule
cat /tmp/bundles/dynamo-platform/values.yaml      # verify global.* keys

# 4. Lint
make lint

# 5. Full qualify gate
make qualify
```
