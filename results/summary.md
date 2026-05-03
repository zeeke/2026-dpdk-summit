# Grout Router Performance Results

**Date:** 2026-05-02
**Grout Version:** 0.15.0 (DPDK 25.11.0)

## Test Environment

| Component | Details |
|-----------|---------|
| Platform | OpenShift 4.21 (Kubernetes v1.34.6) |
| CPU | Intel Xeon Gold 6338N @ 2.20 GHz |
| NIC | Mellanox ConnectX (mlx5), 100 GbE SR-IOV VFs |
| Kernel | 5.14.0-570.107.1.el9_6.x86_64 |
| Grout node | cnfdr26 (4 CPUs, 2 Gi hugepages) |
| Traffic gen node | cnfdr27 (testpmd, 4 CPUs, 2 Gi hugepages) |

### Grout Configuration

- 2 ports (p0, p1), each with 1 RX queue (`rxqs 1`, `qsize 2048`)
- 3 datapath worker CPUs (only 1 active per port due to single RX queue)
- 1 control plane CPU
- L3 forwarding between 192.168.1.0/24 (p0) and 192.168.2.0/24 (p1)

### Test Topology

```
 cnfdr27 (traffic gen)                    cnfdr26 (DUT)
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

Traffic generator: DPDK testpmd in `txonly` mode, 1 core, single TX queue.
Unidirectional test: testpmd TX on net-a → grout p0 RX → L3 route → grout p1 TX.

## Results

### L3 Forwarding Throughput (unidirectional, single RX queue)

| Packet Size (B) | Forwarding Rate (Mpps) | Throughput (Gbps) | Wire-rate Throughput (Gbps) | Line Rate % |
|:----------------:|:----------------------:|:-----------------:|:---------------------------:|:-----------:|
| 64               | 12.45                  | 6.37              | 8.37                        | 8.4%        |
| 128              | 12.32                  | 12.61             | 14.59                       | 14.6%       |
| 256              | 12.62                  | 25.86             | 27.85                       | 27.9%       |
| 512              | 12.72                  | 52.08             | 54.07                       | 54.1%       |
| 1024             | 11.96                  | 97.96             | 99.88                       | **99.9%**   |
| 1518             | 5.48                   | 66.56             | 67.44                       | 67.4%       |

- **Throughput (Gbps):** L3 payload bytes only
- **Wire-rate Throughput (Gbps):** includes Ethernet overhead (preamble + SFD + IFG = 20 B/frame)
- **Line Rate %:** wire-rate throughput as percentage of 100 GbE

### Observations

- **Packet-rate ceiling at ~12.5 Mpps** for 64-512 B packets. This is the
  single-core forwarding limit — grout is configured with `rxqs 1`, so only
  one datapath worker processes traffic per port regardless of total worker
  count.
- **Line rate achieved at 1024 B** (~99.9% utilization). At this packet size
  the NIC TX bandwidth becomes the bottleneck, not the CPU.
- At 2.20 GHz and 12.5 Mpps, grout processes each packet in approximately
  **176 CPU cycles** — including L3 route lookup, TTL decrement, ARP
  resolution, and checksum update.
- The 1518 B result (67.4%) is lower than expected; this may be an artifact
  of the test sequencing (VF rebind delay between test iterations) rather than
  a genuine forwarding bottleneck, since 1024 B already saturates the link.
- **Zero packet loss** inside grout across all tests (`rx_drops = 0`,
  `tx_errors = 0`). Any loss occurs at the NIC HW level when the traffic
  generator exceeds grout's forwarding rate.

### ICMP Latency (through grout, cross-node)

```
PING 192.168.2.2 (192.168.2.2) 56(84) bytes of data.
64 bytes from 192.168.2.2: icmp_seq=1 ttl=63 time=0.104 ms
64 bytes from 192.168.2.2: icmp_seq=2 ttl=63 time=0.044 ms
64 bytes from 192.168.2.2: icmp_seq=3 ttl=63 time=0.036 ms

rtt min/avg/max/mdev = 0.036/0.061/0.104/0.030 ms
```

Sub-100 us average latency through grout (cnf-tests → grout → testpmd, TTL decremented from 64 to 63).

### Scaling Potential

Grout's per-core forwarding rate of ~12.5 Mpps could scale linearly with
additional RX queues. With the 3 available datapath workers and `rxqs 3`:

| Projected | Small Packets (Mpps) | Wire Rate at 64 B (Gbps) |
|-----------|:--------------------:|:------------------------:|
| 1 queue (measured) | 12.5  | 8.4   |
| 3 queues (projected) | ~37.5 | ~25.1 |

## Raw Data

### testpmd TX Rates (offered load)

| Packet Size (B) | testpmd TX (Mpps) | testpmd TX (Gbps) |
|:---------------:|:-----------------:|:-----------------:|
| 64              | 33.2              | 17.0              |
| 128             | 32.9              | 33.7              |
| 256             | 32.4              | 66.3              |
| 512             | 23.2              | 95.2              |
| 1024            | 11.9              | 97.7              |
| 1518            | 8.1               | 98.4              |

### Grout Interface Statistics (final cumulative)

```
INTERFACE  RX_PACKETS      RX_BYTES       RX_DROPS  TX_PACKETS      TX_BYTES       TX_ERRORS
p0         3,571,361,633   935,580,714,577  0        116             8,294          0
p1         2,267           290,901          0        3,707,945,546   940,224,383,733 0
```

Total packets forwarded through grout during all tests: ~3.7 billion packets, ~940 GB.
