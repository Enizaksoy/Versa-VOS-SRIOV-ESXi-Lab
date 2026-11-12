# Poller CPU, forwarded rate, TCP, drops and latency per test (9 Nov 2026)

Source: per-test summaries and `show vsm load` samples from the lab runs (raw files not published).

Cells: VMXNET3 512, VMXNET3 4096, SR-IOV 512, SR-IOV 4096 (tx-descriptors-count). One `show vsm load` sample per test (~11 s into a 20 s test).
"-" = sample not captured (vsm load output did not contain the table).

## A. Measured poller load: PT-0 Svc % / PT-1 Svc % (CPU Load column reads 95-100 on all busy threads, so it is not used)

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| UDP 64 B | - | 43 / 32 | 39 / 43 | 42 / 48 |
| UDP 128 B | - | 47 / 23 | 39 / 41 | 38 / 40 |
| UDP 256 B | - | - | 39 / 42 | 40 / 43 |
| UDP 512 B | - | - | 37 / 49 | 39 / 50 |
| UDP 1024 B | 45 / 21 | - | 37 / 59 | 38 / 60 |
| UDP 1472 B | 44 / 21 | 42 / 19 | 29 / 45 | 29 / 46 |
| TCP upload | 44 / 20 | 42 / - | 28 / 46 | 30 / 47 |
| TCP download | 22 / 50 | 16 / 49 | 47 / 28 | 49 / 30 |

## B. VOS forwarded rate (kpps) in the same tests

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| UDP 64 B | 1022.6 | 896.5 | 1201.7 | 1256.8 |
| UDP 128 B | 853.4 | 768.9 | 1210.2 | 1129.1 |
| UDP 256 B | 843.2 | 758.0 | 1200.7 | 1136.2 |
| UDP 512 B | 812.8 | 731.1 | 1176.3 | 1169.3 |
| UDP 1024 B | 726.5 | 641.0 | 1109.4 | 1098.9 |
| UDP 1472 B | 683.4 | 595.3 | 810.9 | 802.9 |
| TCP upload | 645.9 | 573.2 | 811.5 | 811.8 |
| TCP download | 712.1 | 537.6 | 811.5 | 811.0 |

## C. Poller cost per packet = (PT-0 Svc % + PT-1 Svc %) / forwarded Mpps. Lower = cheaper.
Read as "% of one poller-core per 1 Mpps"; divide by 100 for microseconds of poller CPU per packet (RX + TX poller, workers excluded).
Worked example: SR-IOV 512 UDP 64 B: (39+43) = 82 % at 1.2017 Mpps -> 82 / 1.2017 = 68 -> 0.68 us/packet.
VMXNET3 4096 UDP 64 B: (43+32) = 75 % at 0.8965 Mpps -> 84 -> 0.84 us/packet.

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| UDP 64 B | - | 84 | 68 | 72 |
| UDP 128 B | - | 91 | 66 | 69 |
| UDP 256 B | - | - | 67 | 73 |
| UDP 512 B | - | - | 73 | 76 |
| UDP 1024 B | 91 | - | 87 | 89 |
| UDP 1472 B | 95 | 102 | 91 | 93 |
| TCP upload | 99 | - | 91 | 95 |
| TCP download | 101 | 121 | 92 | 97 |

Takeaways [Inference]: SR-IOV is ~20-27% cheaper per packet at 64-128 B, ~5-10% at 1500 B. 4096 descriptors add ~3-6% per-packet cost on SR-IOV and ~7-20% on VMXNET3. Highest PT Svc Load in any cell is 60%, so VOS CPU is never the limit; the VMXNET3 shortfall is the NIC path.
Port-to-poller mapping differs between the two VMs (VMXNET3: PT-0 busier on upload; SR-IOV: PT-1), so compare the sum, not PT-0 vs PT-0.

## D. TCP, 8 streams, 20 s (iperf3 receiver rate, sender retransmits, VOS egress kpps)

| Direction | Cell | Receiver Gbps | Retransmits | VOS egress kpps |
|---|---|---|---|---|
| Upload (client -> server) | VMXNET3 512 | 7.48 | 8,469 | 645.9 |
| Upload | VMXNET3 4096 | 6.64 | 9,679 | 573.2 |
| Upload | SR-IOV 512 | 9.39 | 3,753 | 811.5 |
| Upload | SR-IOV 4096 | 9.40 | 2,392 | 811.8 |
| Download (-R, server -> client) | VMXNET3 512 | 8.25 | 28,901 | 712.1 |
| Download | VMXNET3 4096 | 6.22 | 2,588 | 537.6 |
| Download | SR-IOV 512 | 9.40 | 1,200 | 811.5 |
| Download | SR-IOV 4096 | 9.39 | 2,540 | 811.0 |

10GbE line rate at 1514 B frames = ~812.7 kpps. VMXNET3 512 download retransmits (28,901) = VOS TX-driver drops (28,908) in the same test.

## E. VOS drops per 20 s test, split TX vs RX
TX = THRM "Drop Packets TX - Driver" (on VMXNET3 equal to vni tx_q0_drop_total). RX = DPDK rx_missed_errors (LAN + WAN). Total = TX + RX.
The ixgbevf VF exposes no per-port drop counters beyond THRM. On VMXNET3, part of the loss before VOS is not visible in VOS counters (ESXi side) [Inference], so these are lower bounds.

| Test | VMXNET3 512 (TX / RX) | VMXNET3 4096 (TX / RX) | SR-IOV 512 (TX / RX) | SR-IOV 4096 (TX / RX) |
|---|---|---|---|---|
| UDP 64 B | 352 / 16,717 | 0 / 12,033 | 0 / 0 | 0 / 0 |
| UDP 128 B | 0 / 7,886 | 0 / 6,736 | 0 / 0 | 0 / 0 |
| UDP 256 B | 43 / 0 | 0 / 25,037 | 0 / 0 | 0 / 0 |
| UDP 512 B | 2 / 0 | 0 / 17,359 | 0 / 0 | 0 / 0 |
| UDP 1024 B | 2,097 / 0 | 0 / 20,735 | 6,490 / 0 | 0 / 0 |
| UDP 1472 B | 650 / 0 | 0 / 6,213 | 6,250 / 0 | 0 / 0 |
| TCP upload | 867 / 0 | 0 / 2,475 | 1,409 / 0 | 0 / 0 |
| TCP download | 28,908 / 0 | 1,557 / 284 | 1,060 / 0 | 0 / 0 |

## F. Latency under load: ping -i 0.01 client -> server through VOS during each test (avg RTT ms, ping loss %)
Idle avg RTT: VMXNET3 512 0.28, VMXNET3 4096 0.27, SR-IOV 512 0.30, SR-IOV 4096 0.28 ms (0% loss).

| Test | VMXNET3 512 | VMXNET3 4096 | SR-IOV 512 | SR-IOV 4096 |
|---|---|---|---|---|
| UDP 64 B | 3.31 / 59% | 3.35 / 81% | 2.32 / 51% | 2.28 / 54% |
| UDP 128 B | 3.73 / 68% | 3.74 / 55% | 2.45 / 66% | 2.39 / 62% |
| UDP 256 B | 3.62 / 78% | 3.66 / 72% | 2.29 / 59% | 2.46 / 67% |
| UDP 512 B | 3.77 / 67% | 4.04 / 62% | 2.27 / 54% | 2.49 / 60% |
| UDP 1024 B | 4.19 / 66% | 4.31 / 56% | 176.33* / 60% | 2.69 / 62% |
| UDP 1472 B | 4.42 / 45% | 4.90 / 51% | 3.59 / 44% | 3.07 / 38% |
| TCP upload | 1.77 / 0.07% | 1.92 / 0.07% | 3.31 / 0.14% | 4.22 / 0% |
| TCP download | 0.96 / 0.5% | 3.36 / 0.07% | 3.47 / 0% | 4.88 / 0% |

* SR-IOV 512 UDP 1024 B: one stall (max 2,076 ms) inflates the average; the min was 1.93 ms. Exclude from the chart or mark as outlier.
Ping loss under UDP flood mostly reflects the saturated receiving VM and shared queues, not VOS alone (iperf -b 0 floods the path).
