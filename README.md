# Grout DPDK Router on OpenShift

This repository contains the Kubernetes manifests and deployment guide
for running [Grout](https://github.com/DPDK/grout), a DPDK-based L3 software router,
on OpenShift with SR-IOV networking.

## Test Topology

```
 TRex (cnfdr27)                          Grout (cnfdr26)
 ┌──────────────┐                      ┌──────────────┐
 │  Traffic gen │    100 GbE SR-IOV    │  L3 router   │
 │              │                      │              │
 │  port 0 ─────┼──────────────────────┼───── p0      │
 │  net-a VF    │   192.168.1.0/24     │              │
 │              │                      │  route +     │
 │              │                      │  forward     │
 │              │                      │              │
 │  port 1 ─────┼──────────────────────┼───── p1      │
 │  net-b VF    │   192.168.2.0/24     │              │
 └──────────────┘                      └──────────────┘
```

Bidirectional L3 forwarding: TRex sends multi-flow IPv4 traffic on both ports.
Grout receives, performs route lookup + TTL decrement + checksum update, and
forwards out the opposite port.

## Test Environment

| Component | Details |
|-----------|---------|
| Platform | OpenShift 4.21 (Kubernetes v1.34.6) |
| Grout | v0.15.0 (DPDK 25.11.0) |
| CPU | Intel Xeon Gold 6338N @ 2.20 GHz |
| NIC | Mellanox ConnectX-6 Dx (mlx5), 100 GbE SR-IOV VFs |
| Grout pod | 10 CPUs (1 control + 9 datapath), 2 Gi hugepages, 4 RX queues/port |
| TRex pod | 8 CPUs (1 master + 1 latency + 6 workers), 2 Gi hugepages |

## Repository Layout

```
manifests/
  00-sriov-operator-install.yaml     SR-IOV Network Operator (OLM subscription)
  01-namespace.yaml                  grout namespace with privileged PSA
  02-sriov-network-node-policy.yaml  VF allocation on each node
  03-sriov-network.yaml              NetworkAttachmentDefinitions
  04-performance-profile.yaml        CPU isolation, hugepages, NUMA tuning
  05-grout-config.yaml               Grout L3 forwarding configuration (ConfigMap)
  06-grout-pod.yaml                  Grout router pod
  07-trafficgen-trex.yaml            TRex traffic generator pod + config

docs/
  deployment-guide.md   Step-by-step deployment instructions
```

## Quick Start

1. Edit the manifests to match your cluster (node hostnames, NIC names, CPU
   topology) — see the [deployment guide](docs/deployment-guide.md)
2. Apply in order:
   ```bash
   oc apply -f manifests/00-sriov-operator-install.yaml
   # wait for operator to be ready
   oc apply -f manifests/01-namespace.yaml
   oc apply -f manifests/02-sriov-network-node-policy.yaml
   # wait for VF configuration (may trigger node reboot)
   oc apply -f manifests/03-sriov-network.yaml
   oc apply -f manifests/04-performance-profile.yaml
   # wait for node reboot
   oc apply -f manifests/05-grout-config.yaml
   oc apply -f manifests/06-grout-pod.yaml
   oc apply -f manifests/07-trafficgen-trex.yaml
   ```
3. Generate traffic and measure
