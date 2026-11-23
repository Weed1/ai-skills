---
name: network-analyser
description: Fast, evidence-based L2/L3 troubleshooting for any vendor (MikroTik, Cisco, Juniper, Huawei, and others) - VLAN reachability, ARP, "MAC/L2 reachable but IP ping fails", STP path, switch MAC tables, port drops and link saturation. Use when asked why a link or VLAN cannot ping, to check a VLAN, to explain port drops, or to analyse similar L2/L3 symptoms.
---

# Network Analyser (fast L2/L3, multi-vendor)

Goal: a conclusion backed by evidence, quickly. Efficiency beats completeness.

## Hard rules

- At most 2 rounds of parallel read-only queries, then conclude. State what is not yet proven and offer one follow-up check instead of digging further on your own.
- Read-only only. Never change a device during analysis.
- Data source priority: the company's internal network monitoring app / NMS (via its MCP or API) first, because it is fast and already has history; direct device CLI/API only for what the NMS does not have.
- Fixes on production devices: prepare commands plus backup, a safe-rollback mechanism (MikroTik Safe Mode, Cisco `reload in`, Juniper `commit confirmed`, Huawei `commit trial`), and rollback steps. A human operator runs them.
- A query that times out: retry once with a narrower filter; if it still fails, mark it "not read" and move on.
- Never paste customer data (names, public IPs, configs) into external services. Delete temporary logs after reading.
- Confirm the vendor and OS version before giving commands; syntax differs between versions.

## Command cheat sheet (read-only)

| Check | MikroTik RouterOS | Cisco IOS/IOS-XE | Juniper Junos | Huawei VRP |
|---|---|---|---|---|
| IP on interface | `/ip address print where interface=X` | `show ip interface brief` | `show interfaces terse` | `display ip interface brief` |
| ARP | `/ip arp print where interface=X` | `show ip arp vlan N` | `show arp interface X` | `display arp interface X` |
| Neighbors | `/ip neighbor print` | `show cdp neighbors` / `show lldp neighbors` | `show lldp neighbors` | `display lldp neighbor brief` |
| MAC table | `/interface bridge host print where vid=N` | `show mac address-table vlan N` | `show ethernet-switching table vlan-id N` | `display mac-address vlan N` |
| STP roles | `/interface bridge port print` | `show spanning-tree vlan N` | `show spanning-tree interface` | `display stp brief` |
| VLAN on trunk | `/interface bridge vlan print` | `show interfaces trunk` | `show vlans` | `display port vlan` |
| Port rate/drops | `/interface monitor-traffic X once` | `show interfaces X` | `show interfaces X extensive` | `display interface X` |
| Ping | `/ping A count=3` | `ping A repeat 3` | `ping A count 3` | `ping -c 3 A` |

## Round 1 (all in parallel)

1. Find every device that carries the VLAN or link (search by VLAN ID, interface name, or description in the NMS).
2. On the failing device and on one healthy comparison device: IP on the interface, ARP for the peer, L2 neighbors, ping to the peer (3 packets).
3. For drop/congestion questions: current rate on the port, NMS alarms, and traffic history for the same port.

Read the pattern:
- Neighbor visible + ARP incomplete/failed: L2 passes, unicast replies are lost. Go to round 2 (MAC tables).
- Neighbor not visible: L2 is broken. Find which trunk does not carry the VLAN.
- ARP complete but ping fails: ACL/firewall or routing on one end.
- L2 discovery (LLDP/CDP/MNDP, MAC ping) works but IP ping fails: discovery uses broadcast/multicast, while ARP replies and ping use unicast. The problem is unicast forwarding (switch MAC table) or ARP on the far end.
- Port drops: compare current rate with link speed. Near line rate = congestion. Check broadcast/multicast counters before calling it a storm.

## Round 2 (parallel, only if needed)

- Look up both endpoint MACs across switches to see which port each switch learned them on.
- On the switch where the paths meet: MAC table for the VLAN, filtered to non-local entries.
- Compare with the STP path (port roles root/designated/alternate).
- For congestion: traffic history for the same hours on a normal day (same weekday last week) versus now.

Known signature (MAC learned on the wrong port): the meeting switch learns router A's MAC on a port that is not on the STP path toward router A, for example a provider metro trunk. Replies from the far side that enter on that same port are dropped, because a bridge never sends a frame back out its ingress port. Typical cause: a second L2 path through a provider network that does not pass BPDUs (a hidden loop). Fix for VLANs that do not need that trunk: remove the trunk from that VLAN, then wait for MAC ageing (often 5 min) before judging the result.

Known signature (hidden loop or rerouted traffic): a port sits flat at line rate at an hour when it is normally a third of that, while the reverse direction looks normal.

## API/automation notes

- Filters on VLAN lists often fail to match entries stored as ranges; fetch everything with a field list and filter locally.
- Physical ports that are LAG/bond members may appear in VLAN membership lists with no effect on forwarding; check the LAG itself.

## Report format

1. First sentence: the cause or main finding.
2. Short evidence table (device, port/IP, state, numbers).
3. "Not yet proven:" one line.
4. Fix (if any): numbered steps, one command per code block, where to run it, backup and rollback.
