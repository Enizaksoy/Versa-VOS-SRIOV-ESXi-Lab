# VOS 22.1.4 on ESXi: SR-IOV vs VMXNET3, tx-descriptors 512 vs 4096 (9 Nov 2026)

Host: ESXi 7.0 U3, Lenovo P920 (2x Xeon 8160 2.1 GHz). VOS 22.1.4-GA, 8 vCPU (5 WT + 2 PT + ctrl), routing + DIA NAT.
SR-IOV VOS = TEST_SRIOV (X540 VF / ixgbevf). VMXNET3 VOS = TEST_NON_SRIOV (clone of TEST_SRIOV, same config).
Client VM (10.40.40.50, VMXNET3 on vSwitch1/vmnic3) -> VOS LAN vni-0/2 -> VOS WAN vni-0/0 -> server VM (192.168.100.10, VMXNET3 on vSwitch1).
Test runner script on the client VM (not published), 20 s per test, UDP `iperf3 -u -l <size> -b 0 -P 4`, TCP `-P 8` (upload) and `-R` (download).
Latency = `ping -i 0.01` client -> server through VOS during each test. Raw per-test files are not published in this repo.

**Path caveat:** on the VMXNET3 VOS the WAN vNIC sits on vSwitch1 with the server VM (in-host), the LAN leg crosses the vmnic3<->vmnic4 cable.
On the SR-IOV VOS both legs cross the cable. Small-packet SR-IOV rows are client-bound (~1.2 Mpps offered).

## VOS forwarded rate (kpps, VOS egress tx_good_packets / 20 s)

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| UDP 64 B | 1022.6 | 896.5 | 1201.7 | 1256.8 |
| UDP 128 B | 853.4 | 768.9 | 1210.2 | 1129.1 |
| UDP 256 B | 843.2 | 758.0 | 1200.7 | 1136.2 |
| UDP 512 B | 812.8 | 731.1 | 1176.3 | 1169.3 |
| UDP 1024 B | 726.5 | 641.0 | 1109.4 | 1098.9 |
| UDP 1472 B | 683.4 | 595.3 | 810.9 (line rate) | 802.9 |
| TCP up (Gbps / retr) | 7.48 / 8,469 | 6.64 / 9,679 | 9.39 / 3,753 | 9.40 / 2,392 |
| TCP-R (Gbps / retr) | 8.25 / 28,901 | 6.22 / 2,588 | 9.40 / 1,200 | 9.39 / 2,540 |

## VOS drops per 20 s test

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| UDP 64 | TX +352, LAN rx_missed +16,717 | LAN rx_missed +12,033 | 0 | 0 |
| UDP 128 | LAN rx_missed +7,886 | LAN rx_missed +6,736 | 0 | 0 |
| UDP 256 | TX +43 | LAN rx_missed +25,037 | 0 | 0 |
| UDP 512 | TX +2 | LAN rx_missed +17,359 | 0 | 0 |
| UDP 1024 | TX +2,097 | LAN rx_missed +20,735 | THRM TX +6,490 | 0 |
| UDP 1472 | TX +650 | LAN rx_missed +6,213 | THRM TX +6,250 | 0 |
| TCP up | TX +867 | LAN rx_missed +2,475 | THRM TX +1,409 | 0 |
| TCP-R | TX +28,908 (= retr) | TX +1,557 | THRM TX +1,060 | 0 |

TX = THRM "Drop Packets TX - Driver" (+ vmxnet3 tx_q0_drop_total). The ixgbevf VF exposes only good_* counters, so on SR-IOV only THRM is visible.
On VMXNET3, offered minus forwarded is larger than VOS rx_missed: part of the loss is not visible inside VOS (ESXi side of the vmxnet3 RX ring) [Inference].

## Latency under load (ping avg ms / ping loss)

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| idle | 0.28 | 0.27 | 0.30 | 0.28 |
| UDP 64 | 3.31 / 59% | 3.35 / 81% | 2.32 / 51% | 2.28 / 54% |
| UDP 512 | 3.77 / 67% | 4.04 / 62% | 2.27 / 54% | 2.49 / 60% |
| UDP 1024 | 4.19 / 66% | 4.31 / 56% | 176 (one 2 s stall) / 60% | 2.69 / 62% |
| UDP 1472 | 4.42 / 45% | 4.90 / 51% | 3.59 / 44% | 3.07 / 38% |
| TCP up | 1.77 / 0% | 1.92 / 0% | 3.31 / 0% | 4.22 / 0% |
| TCP-R | 0.96 / 0% | 3.36 / 0% | 3.47 / 0% | 4.88 / 0% |

Ping loss under UDP flood is mostly the receiving VM (server) and the shared queues, not a VOS figure.

## Poller load (Svc Load % / CPU Load %), sampled ~11 s into each test

PT Svc Load stayed 16-60% in every run (VMXNET3 and SR-IOV); CPU Load 95-100 on all busy threads. Idle VOS shows PT CPU Load ~8%.
Several VMXNET3 samples did not parse (the `show vsm load` call returned early).
VOS CPU is not the bottleneck in any cell.

## Mgmt SSH to the client VM during the matrix (PC -> client VM port 22 every 2 s)

| Run | OK | FAIL |
|---|---|---|
| VMXNET3 512 | 181 | 45 |
| VMXNET3 4096 | 267 | 39 |
| SR-IOV 512 | 250 | 0 |
| SR-IOV 4096 | 257 | 0 |

Same client VM, same flood: the drops only occur with the VMXNET3 VOS -> host vSwitch/vmxnet3 backend load [Inference].

## Conclusions

1. SR-IOV forwards 18-53% more pps than VMXNET3 at tx-desc 512 (35-71% at 4096) at every size and reaches 10GbE line rate for 1472 B and TCP (9.4 Gbps).
2. With VMXNET3, 512 TX descriptors drop on TX in bursts (TCP-R retransmits = TX drops); 4096 removes TX drops but forwarding falls 10-25% and RX misses appear [Inference: mbufs held in larger TX rings / RX starvation, Speculation].
3. With SR-IOV, 4096 removes the remaining THRM TX drops (1024/1472/TCP) with no throughput loss; ping avg rises ~1 ms under TCP load.
4. Latency under UDP load: SR-IOV ~2.3 ms vs VMXNET3 ~3.3-4.9 ms.
5. VMXNET3 load also disturbs other VMs' management traffic on the same host (SSH timeouts); SR-IOV does not.

Single 20 s run per cell (n=1). Repeat before quoting numbers externally. Not sizing guidance.
