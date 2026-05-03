# Deployment Guide

Step-by-step instructions to reproduce the Grout L3 forwarding test on OpenShift.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| OpenShift | 4.18+ with bare-metal or on-prem nodes |
| Nodes | 2 worker nodes, each with an SR-IOV capable NIC (Mellanox ConnectX-6 Dx or similar, adjust the SriovNetworkNodePolicy accordingly) |
| NICs | 100 GbE ports on the same physical fabric (same switch / direct cable) |
| Hugepages | At least 4 Gi of 1 GiB hugepages per node |
| CPUs | 4+ isolated cores per node (same NUMA node as the NIC) |

## 1. Install Operators

### SR-IOV Network Operator

Check if already installed:

```bash
oc get csv -A | grep -i sriov-network-operator
```

If not, install it:

```bash
oc apply -f manifests/00-sriov-operator-install.yaml
```

Wait for the operator to be ready:

```bash
oc wait --for=condition=Available csv \
  -l operators.coreos.com/sriov-network-operator.openshift-sriov-network-operator \
  -n openshift-sriov-network-operator --timeout=300s
```

### Node Tuning Operator

The Node Tuning Operator is installed by default on OpenShift. Verify:

```bash
 oc get clusteroperators.config.openshift.io | grep tuning
```

## 2. Create the Namespace

```bash
oc apply -f manifests/01-namespace.yaml
```

## 3. Configure SR-IOV Virtual Functions

Edit `manifests/02-sriov-network-node-policy.yaml` to match your environment:

- **`pfNames`**: physical function name on each node (run `ip link` on the node)
- **`nodeSelector`**: FQDN of each worker node
- **`numVfs`**: number of VFs to create (2 is enough for this test)
- **`deviceType`**: `netdevice` for Mellanox (kernel driver), `vfio-pci` for Intel

```bash
oc apply -f manifests/02-sriov-network-node-policy.yaml
```

Wait for the SR-IOV operator to configure the nodes (this may trigger a node reboot):

```bash
oc wait sriovnetworknodestates --all \
  -n openshift-sriov-network-operator \
  --for=jsonpath='{.status.syncStatus}'=Succeeded --timeout=600s
```

## 4. Create SR-IOV Networks

Edit `manifests/03-sriov-network.yaml` to match your `resourceName` from the previous step.

```bash
oc apply -f manifests/03-sriov-network.yaml
```

This creates NetworkAttachmentDefinitions in the `grout` namespace.

## 5. Apply the Performance Profile

Edit `manifests/04-performance-profile.yaml`:

- **`reserved`** / **`isolated`** CPU sets: match your node's CPU topology (`lscpu -e`)
- **`hugepages`**: adjust page count based on available memory
- Ensure isolated CPUs are on the **same NUMA node** as the NIC:
  ```bash
  cat /sys/bus/pci/devices/<NIC_PCI_ADDR>/numa_node
  ```

```bash
oc apply -f manifests/04-performance-profile.yaml
```

**This triggers a node reboot.** Wait for the MachineConfigPool to finish:

```bash
oc wait mcp worker --for condition=Updated --timeout=1800s
```

## 6. Deploy Grout

### ConfigMap

Edit `manifests/05-grout-config.yaml` if you need different subnets. The default config sets up:
- p0: 192.168.1.10/24
- p1: 192.168.2.10/24

The `__VF0__` and `__VF1__` placeholders are automatically resolved at pod startup.

```bash
oc apply -f manifests/05-grout-config.yaml
```

### Grout Pod

Edit `manifests/06-grout-pod.yaml`:

- **`nodeSelector`**: set to your DUT (Device Under Test) node
- **`k8s.v1.cni.cncf.io/networks`**: match the SriovNetwork names for this node
- **`openshift.io/<resourceName>`**: match the SR-IOV resource name

```bash
oc apply -f manifests/06-grout-pod.yaml
```

Verify grout is running:

```bash
oc wait pod grout -n grout --for=condition=Ready --timeout=60s
oc exec -n grout grout -- grcli interface show
oc exec -n grout grout -- grcli address show
oc exec -n grout grout -- grcli route show
```

## 7. Deploy the Traffic Generator

Choose one: **testpmd** (simpler) or **TRex** (more features).

### Option A: testpmd

Edit `manifests/07-trafficgen-testpmd.yaml`:

- **`nodeSelector`**: set to your traffic generator node
- **`k8s.v1.cni.cncf.io/networks`**: match the SriovNetwork names for this node

```bash
oc apply -f manifests/07-trafficgen-testpmd.yaml
```

Run traffic (exec into the pod):

```bash
oc exec -it -n grout testpmd -- bash

# Discover VF PCI addresses
env | grep PCIDEVICE

# Generate traffic at 64B toward grout
testpmd -l 0-3 -n 4 --socket-mem 1024 \
  -a <VF0_PCI> \
  -- --forward-mode=txonly --txpkts=64 --stats-period=5
```

### Option B: TRex

Edit `manifests/08-trafficgen-trex.yaml`:

- Update PCI address placeholders in the TRex config
- Adjust IP addresses if using different subnets

```bash
oc apply -f manifests/08-trafficgen-trex.yaml
```

## 8. Verify Connectivity

From a helper pod or the testpmd pod, ping through grout:

```bash
ping 192.168.2.10   # grout's p1 address
```

Expected: sub-millisecond RTT with TTL decremented by 1 (grout performs L3 forwarding).

## 9. Measure Performance

While traffic is running, measure grout's forwarding rate:

```bash
# Take two samples 10 seconds apart
oc exec -n grout grout -- grcli -j interface stats
sleep 10
oc exec -n grout grout -- grcli -j interface stats
```

Compute the delta between samples to get packets-per-second and bytes-per-second rates.

## Customization Reference

| Parameter | File | What to Change |
|-----------|------|----------------|
| Node hostnames | 02, 06, 07, 08 | `nodeSelector` and network annotation names |
| NIC PF name | 02 | `pfNames` in SriovNetworkNodePolicy |
| CPU topology | 04 | `reserved` / `isolated` CPU sets |
| Hugepages count | 04 | `pages.count` |
| IP subnets | 03, 05, 08 | Subnet in IPAM config, grout.init, TRex profile |
| Packet sizes | (runtime) | `--txpkts` flag in testpmd |
| RX queue count | 05 | `rxqs` in grout.init |
