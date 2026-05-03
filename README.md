# Grout DPDK Router on OpenShift — Performance Evaluation

This repository contains the Kubernetes manifests, test results, and deployment guide
for running [Grout](https://github.com/DPDK/grout), a DPDK-based L3 software router,
on OpenShift with SR-IOV networking.

The test evaluates Grout's single-core L3 forwarding throughput across packet sizes
from 64 B to 1518 B over 100 GbE Mellanox ConnectX SR-IOV virtual functions.

## Test Topology

```
 Traffic Generator (cnfdr27)              DUT (cnfdr26)
 ┌──────────────┐                      ┌──────────────┐
 │   testpmd    │                      │    grout     │
 │  (txonly)    │                      │   router     │
 │              │    100 GbE SR-IOV    │              │
 │  port0 (TX)  ├─────────────────────►│ p0 (RX)      │
 │  net-a VF    │   192.168.1.0/24     │              │
 │              │                      │   route      │
 │              │                      │   lookup     │
 │              │    100 GbE SR-IOV    │              │
 │     (sink)   │◄─────────────────────┤ p1 (TX)      │
 │  net-b VF    │   192.168.2.0/24     │              │
 └──────────────┘                      └──────────────┘
```

Unidirectional: testpmd generates traffic on net-a, grout receives on p0,
performs L3 route lookup + TTL decrement + checksum update, and forwards out p1.

## Key Results

| Packet Size (B) | Forwarding Rate (Mpps) | Wire-rate (Gbps) | Line Rate % |
|:----------------:|:----------------------:|:-----------------:|:-----------:|
| 64               | 12.45                  | 8.37              | 8.4%        |
| 128              | 12.32                  | 14.59             | 14.6%       |
| 256              | 12.62                  | 27.85             | 27.9%       |
| 512              | 12.72                  | 54.07             | 54.1%       |
| 1024             | 11.96                  | 99.88             | **99.9%**   |
| 1518             | 5.48                   | 67.44             | 67.4%       |

- **12.5 Mpps** sustained on a single core — ~176 CPU cycles per packet at 2.20 GHz
- **99.9% line rate** at 1024 B packets (NIC bandwidth becomes the bottleneck)
- **Zero packet loss** inside grout (`rx_drops = 0`, `tx_errors = 0`)
- **Sub-100 us latency** through the router (avg 61 us cross-node ICMP)
- ~3.7 billion packets / ~940 GB forwarded during the full test run

## Test Environment

| Component | Details |
|-----------|---------|
| Platform | OpenShift 4.21 (Kubernetes v1.34.6) |
| Grout | v0.15.0 (DPDK 25.11.0) |
| CPU | Intel Xeon Gold 6338N @ 2.20 GHz |
| NIC | Mellanox ConnectX-6 Dx (mlx5), 100 GbE SR-IOV VFs |
| Kernel | 5.14.0-570.107.1.el9_6.x86_64 |
| Grout pod | 4 CPUs, 2 Gi hugepages, 1 RX queue per port |
| Traffic gen | DPDK testpmd (txonly), 4 CPUs, 2 Gi hugepages |

## Repository Layout

```
manifests/          Kubernetes YAML files to deploy the full test setup
  00-sriov-operator-install.yaml     SR-IOV Network Operator (OLM subscription)
  01-namespace.yaml                  grout namespace with privileged PSA
  02-sriov-network-node-policy.yaml  VF allocation on each node
  03-sriov-network.yaml              Logical networks (NetworkAttachmentDefinitions)
  04-performance-profile.yaml        CPU isolation, hugepages, NUMA tuning
  05-grout-config.yaml               Grout L3 forwarding configuration (ConfigMap)
  06-grout-pod.yaml                  Grout router pod
  07-trafficgen-testpmd.yaml         testpmd traffic generator pod
  08-trafficgen-trex.yaml            TRex traffic generator (alternative)

results/
  summary.md        Performance results with tables and observations
  raw-data.md       Full evidence: pod logs, grcli output, raw stats per test

docs/
  deployment-guide.md   Step-by-step deployment instructions
```

## Quick Start

1. Edit the manifests to match your cluster (node hostnames, NIC names, CPU topology) — see the [deployment guide](docs/deployment-guide.md) for details
2. Apply in order:
   ```bash
   oc apply -f manifests/00-sriov-operator-install.yaml
   # wait for operator...
   oc apply -f manifests/01-namespace.yaml
   oc apply -f manifests/02-sriov-network-node-policy.yaml
   # wait for VF configuration...
   oc apply -f manifests/03-sriov-network.yaml
   oc apply -f manifests/04-performance-profile.yaml
   # wait for node reboot...
   oc apply -f manifests/05-grout-config.yaml
   oc apply -f manifests/06-grout-pod.yaml
   oc apply -f manifests/07-trafficgen-testpmd.yaml
   ```
3. Generate traffic and measure forwarding rate — see [results](results/summary.md)
