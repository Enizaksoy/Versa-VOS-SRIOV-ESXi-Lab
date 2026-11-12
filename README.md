# Versa VOS SR-IOV on VMware ESXi: lab data

Companion data for the article **[Past the vSwitch: Enabling SR-IOV for Versa VOS on VMware vSphere](https://enizaksoy.github.io/articles/versa-vos-sriov-esxi.html)**.
The article has the step-by-step setup (host VFs, the VM passthrough adapter, verification inside VOS and on ESXi), the gotchas and the charts. This repo has the result tables and the full command outputs behind it.

> **Lab results only.** Every number here comes from one lab setup built by the author: one ESXi host, one VOS VM, one 20-second run per test (n=1).
> These are not official Versa Networks performance figures, they have not been validated by any vendor, and they are not sizing guidance.
> For sizing, use Versa's published specifications.

Method: written with AI assistance; all commands and outputs are from the author's lab (9 Nov 2026).

## Test bed

| Item | Value |
|---|---|
| VOS | 22.1.4-GA, 8 vCPU (5 worker + 2 poller threads + control), 8 GB, routing + DIA NAT |
| ESXi | 7.0 U3 (build 21424296), 2x Intel Xeon Platinum 8160 2.1 GHz |
| vCenter | 8.0.3 (build 24022515) |
| NIC | Intel X540-AT2 (ixgben on ESXi, ixgbevf VF / `net_ixgbe_vf` in VOS); vmnic3 <-> vmnic4 cabled back-to-back, 10G |
| Not usable | ConnectX-4 Lx: VFs work on ESXi, but VOS 22.1.4 has no mlx5 driver (`Unsupported device type`) |
| Path | client VM 10.40.40.50 -> VOS LAN `vni-0/2` -> VOS WAN `vni-0/0` (NAT) -> server VM 192.168.100.10 |
| Load | iperf3, 20 s per test. UDP `-u -l <size> -b 0 -P 4`; TCP `-P 8` (upload) and `-R` (download) |
| Cells | VMXNET3 and SR-IOV (same VOS config, cloned VM), each with `tx-descriptors-count` 512 and 4096 |

## Key results

| Metric (one VOS LAN/WAN pair, 20 s tests) | VMXNET3 (512 desc) | SR-IOV (4096 desc) |
|---|---|---|
| TCP, 8 streams (Gbps) | 7.48 up / 8.25 down | 9.40 / 9.39 |
| UDP 64 B forwarded (kpps) | 1,023 | 1,257 (client-bound) |
| UDP 1472 B forwarded (kpps) | 683 | 803 (line rate ~813) |
| VOS drops per test | up to 28,908 (TX ring full) | 0 |
| Ping RTT under UDP load (ms) | 3.3 to 4.4 | 2.3 to 3.1 |
| SSH failures to another VM on the host | 39 to 45 per run | 0 |

Full matrix, drops split TX/RX, latency and poller cost per packet: [results/COMPARISON.md](results/COMPARISON.md) and [results/CPU-AND-TCP-DATA.md](results/CPU-AND-TCP-DATA.md).

## Files

| Path | Content |
|---|---|
| `results/COMPARISON.md` | Forwarded kpps, TCP, drops, latency and SSH impact for the four cells, with conclusions |
| `results/CPU-AND-TCP-DATA.md` | Poller Svc Load per test, poller % per Mpps, TCP detail, drops TX/RX, latency |
| `results/esxcli-sriov-2026-11-09.txt` | `esxcli network sriovnic list`, `ixgben` module parameters, VF list on vmnic4 (VM off) |
| `results/step3-vos-outputs-2026-11-09.txt` | VOS shell and CLI checks, vsmd load/stats during TCP tests, ESXi VF list and per-VF counters before/after, idle counter cross-check |
| `images/` | The six figures from the article (screenshots with host name, MAC addresses and vCenter name masked) |

## Sanitisation

Management and lab-internet IPs are replaced with documentation ranges (192.0.2.0/24, 198.51.100.0/24), MAC addresses with `00:50:56:aa:bb:xx` / `aa:bb:cc:00:00:xx`, the VOS hostname with `BRANCH-1` and the organisation name with `ACME`.
The data-plane test addresses (10.40.40.0/24, 192.168.100.0/24) and all counter values are unchanged. Test scripts and raw per-test files are not published.

## References

- VMware: [SR-IOV Support](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vsphere-networking/managing-network-resources/sr-iov/sr-iov-support.html)
- Versa: [Install on VMware ESXi (VOS)](https://learn.versa-networks.com/docs/install-on-vmware-esxi-installation)
- Versa: [Hardware and Software Requirements for Branch](https://learn.versa-networks.com/docs/hardware-and-software-requirements-for-branch)
