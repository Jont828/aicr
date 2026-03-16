# AKS GPU Driver Setup

AKS GPU nodepools install NVIDIA drivers by default. This conflicts with the
GPU Operator, which also installs drivers by default. Use one of the approaches
below to avoid the conflict.

## Recommended: Let GPU Operator Manage the Driver

Create nodepools with `--gpu-driver none` so AKS skips its driver installation
and the GPU Operator handles it:

```shell
az aks nodepool add \
  --cluster-name <cluster> \
  --resource-group <rg> \
  --name gpupool \
  --node-vm-size Standard_NC80adis_H100_v5 \
  --gpu-driver none \
  --node-count 1
```

No changes to AICR recipes are needed — this is the default configuration.

## Alternative: Use the AKS-Managed Driver

If you prefer the AKS-managed driver (e.g., for driver version pinning by AKS),
disable the GPU Operator driver:

```shell
aicr bundle -r recipe.yaml --set gpuoperator:driver.enabled=false
```

Or add to your values override file:

```yaml
driver:
  enabled: false
```

## References

- [GPU Operator on AKS](https://learn.microsoft.com/en-us/azure/aks/nvidia-gpu-operator)
- [AKS GPU Node Pools](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster)
