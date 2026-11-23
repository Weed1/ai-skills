---
name: network-analyser
description: Fast, evidence-based L2/L3 troubleshooting for MikroTik RouterOS networks (VLAN, ARP, "MAC ping works but IP ping fails", STP path, switch MAC table). Use when asked why a link/VLAN cannot ping, to check a VLAN, or to analyse similar L2/L3 symptoms.
---

# Network Analyser (fast L2/L3)

Goal: a conclusion backed by evidence, quickly. Efficiency beats completeness.

## Hard rules

- At most 2 rounds of parallel read-only queries, then conclude. State what is not yet proven and offer a follow-up check instead of digging further on your own.
- Read-only commands only (`print`, `ping`, `monitor ... once`). Never change a device during analysis.
- Fixes on production routers/switches: prepare commands plus backup (`/export`, `/system backup save`), Safe Mode, and rollback steps. The human operator runs them.
- A query that times out: retry once with a narrower filter; if it still fails, mark it "not read" and move on.
- Never paste customer data (names, public IPs, configs) into external services. Delete temporary logs after reading.

## Round 1 (all in parallel)

1. Find every router/switch that carries the VLAN (search interfaces by VLAN ID or name).
2. On the failing router and on one healthy comparison router:
   - `/ip address print where interface=<vlan>`
   - `/ip arp print where interface=<vlan>`
   - `/ip neighbor print where interface=<vlan>` (identity, MAC, address)
   - `/ping <peer> count=3`

Read the pattern:
- Neighbor visible + ARP incomplete/failed: L2 passes, unicast replies are lost. Go to round 2 (MAC tables).
- Neighbor not visible: L2 is broken. Trace which trunk port does not carry the VLAN.
- ARP complete but ping fails: firewall or routing on one end.
- MAC ping / MNDP work but IP ping fails: MAC ping and MNDP use broadcast, while ARP replies and ping use unicast. The problem is unicast forwarding (switch MAC table) or ARP on the far end.

## Round 2 (parallel, only if needed)

- Look up both endpoint MACs across switches to see which port each switch learned them on.
- On the switch where paths meet: `/interface bridge host print where vid=<vlan> and local=no`.
- Compare with the STP path (`/interface bridge port print` roles: root/designated/alternate).

Known signature: the meeting switch learns router A's MAC on a port that is NOT on the STP path toward router A (for example a provider metro trunk). Replies from the far side that enter on that same port are dropped, because a bridge never sends a frame back out its ingress port. Typical cause: a second L2 path through a provider network that does not pass BPDUs. Fix for VLANs that do not need that trunk: remove the trunk from the VLAN's bridge VLAN entry, then wait for MAC ageing (default 5 min) before judging the result.

## RouterOS API notes

- `?vlan-ids=N` does not match entries stored as ranges/lists; fetch all entries with `.proplist` and filter locally.
- Regex `~` filters do not work on list fields (tagged/untagged) over the API.
- Physical ports that are bonding slaves show `disabled` as bridge ports; their presence in a tagged list has no effect on traffic.
- RouterOS v6 and v7 syntax differ: confirm the version first.

## Report format

1. First sentence: the cause or main finding.
2. Short evidence table (device, IP, ARP state, ping result).
3. "Not yet proven:" one line.
4. Fix (if any): numbered steps, one command per code block, where to run it, backup and rollback.
