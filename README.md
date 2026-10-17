# Enterprise Multi-Site Network Architecture & Secure VPN Implementation

**Role:** Lead Network Designer | **Duration:** Oct 2024 – Dec 2024 | **Tools:** GNS3 (Cisco IOSv / IOSv-L2), Cisco Packet Tracer

## 1. Objective
Design and simulate a three-tier hierarchical enterprise network connecting a centralized HQ data center to three branch offices over a simulated ISP, with:
- OSPF multi-area routing
- Site-to-site IPsec VPN (GRE over IPsec, so OSPF runs across the tunnels)
- HSRP gateway redundancy at HQ
- 802.1Q VLAN segmentation
- Standard and Extended ACLs isolating Finance and Guest traffic

## 2. Topology

```
                         ┌──────────┐
        ┌────────────────┤   ISP    ├────────────────┐
        │                └──┬────┬──┘                │
        │ 203.0.113.0/30    │    │ 198.51.100.4/30   │ 198.51.100.8/30
   ┌────┴─────┐      198.51.100.0/30             ┌────┴───┐
   │ HQ-Edge  │    ┌──┴───┐     ┌──────┐        │  BR3   │
   │ (ABR,    │    │ BR1  │     │ BR2  │        └───┬────┘
   │ VPN hub) │    └──┬───┘     └──┬───┘            │
   └──┬────┬──┘      Area 1       Area 2          Area 3
 Area 0│    │Area 0   (BR switch + VLANs 10/30/40 per branch)
 ┌─────┴┐  ┌┴─────┐
 │Core1 ├──┤Core2 │   HSRP pair (distribution/core layer)
 └──┬───┘  └───┬──┘
    └────┬─────┘
    ┌────┴─────┐
    │ HQ-Access│  VLANs 10, 20, 30, 40, 99
    └──────────┘
```

**Three tiers (HQ):** Access (HQ-Access) → Distribution/Core (HQ-Core1/2, L3, HSRP) → Edge/WAN (HQ-Edge). Branches use a collapsed-core design (router-on-a-stick + access switch).

Virtual tunnels (GRE over IPsec) run HQ-Edge ↔ each branch; the branch LANs sit in OSPF areas 1–3 and HQ sits in Area 0, so HQ-Edge is the ABR.

## 3. IP Addressing

| Link / Network | Subnet | Notes |
|---|---|---|
| HQ-Edge ↔ ISP | 203.0.113.0/30 | ISP .1, HQ .2 |
| BR1 ↔ ISP | 198.51.100.0/30 | ISP .1, BR1 .2 |
| BR2 ↔ ISP | 198.51.100.4/30 | ISP .5, BR2 .6 |
| BR3 ↔ ISP | 198.51.100.8/30 | ISP .9, BR3 .10 |
| HQ-Edge ↔ Core1 | 10.255.0.0/30 | Edge .1, Core1 .2 (Area 0) |
| HQ-Edge ↔ Core2 | 10.255.0.4/30 | Edge .5, Core2 .6 (Area 0) |
| Tunnel HQ ↔ BR1 | 172.16.1.0/30 | HQ .1, BR1 .2 (Area 1) |
| Tunnel HQ ↔ BR2 | 172.16.2.0/30 | HQ .1, BR2 .2 (Area 2) |
| Tunnel HQ ↔ BR3 | 172.16.3.0/30 | HQ .1, BR3 .2 (Area 3) |

### VLAN plan (X = site: HQ=0, BR1=1, BR2=2, BR3=3)

| VLAN | Name | Subnet | Present at |
|---|---|---|---|
| 10 | FINANCE | 10.X.10.0/24 | all sites |
| 20 | IT | 10.0.20.0/24 | HQ only |
| 30 | SALES | 10.X.30.0/24 | all sites |
| 40 | GUEST | 10.X.40.0/24 | all sites |
| 99 | MGMT | 10.0.99.0/24 | HQ only |
| 999 | NATIVE (unused) | – | all trunks |

HQ SVIs: HSRP VIP = .1, Core1 = .2, Core2 = .3. Core1 is HSRP active for VLANs 10 & 20, Core2 for 30, 40 & 99 (load sharing).

### Router IDs
HQ-Edge 1.1.1.1, Core1 2.2.2.2, Core2 3.3.3.3, BR1 11.11.11.11, BR2 12.12.12.12, BR3 13.13.13.13.

## 4. Design Decisions

- **OSPF multi-area:** Area 0 = HQ backbone (edge + cores + HQ VLANs). Areas 1/2/3 = one per branch, which keeps LSA flooding and SPF recalculation local to each site. HQ-Edge acts as the ABR.
- **GRE over IPsec (transport mode):** plain crypto-map IPsec can't carry OSPF multicast. GRE tunnels carry OSPF; the IPsec profile (AES-256, SHA-256, DH group 14, PSK) encrypts the GRE traffic.
- **HSRP:** Core1/Core2 share a virtual gateway per VLAN with preemption and interface tracking on the uplink to the edge, so a lost uplink moves the active role to the peer.
- **ACL policy:**
  - *Finance (VLAN 10)* is reachable only from Finance and IT (extended ACL, outbound on the SVI/sub-interface; return traffic allowed via `established`/echo-reply).
  - *Guest (VLAN 40)* may reach only external/internet addresses; all RFC1918 ranges are denied (extended ACL inbound).
  - *Management:* VTY access restricted to the MGMT subnet (standard ACL 10).
- **L2 hardening:** unused native VLAN 999, BPDU guard + PortFast on access ports, port-security (max 2), unused ports shut down.

## 5. Files

| File | Device |
|---|---|
| `isp.cfg` | ISP router |
| `hq-edge.cfg` | HQ edge router / VPN hub / ABR |
| `hq-core1.cfg`, `hq-core2.cfg` | HQ L3 switches (HSRP pair) |
| `hq-access.cfg` | HQ access switch |
| `branch1.cfg`, `branch2.cfg`, `branch3.cfg` | Branch routers |
| `branch-switch.cfg` | Branch access switch (same for all, change hostname) |

## 6. Build Steps (GNS3)

1. Images: Cisco IOSv (routers) and IOSv-L2 (switches, needed for SVIs/HSRP).
2. Build the topology above (ISP in the middle; HQ-Edge, BR1, BR2, BR3 each connect to the ISP). Interfaces used are in each config's header comment.
3. Paste configs in this order: ISP → HQ-Edge → Core1/Core2 → access switches → branches.
4. Attach a lightweight host (VPCS) to each VLAN port to generate test traffic.

**Packet Tracer note:** PT doesn't support GRE-over-IPsec/`tunnel protection` or HSRP tracking on all devices. For PT, use a crypto-map site-to-site IPsec VPN (interesting-traffic ACL between site subnets) and static routes between sites; OSPF multi-area stays inside HQ.

## 7. Verification Checklist

| Test | Command | Expected |
|---|---|---|
| OSPF neighbors | `show ip ospf neighbor` (HQ-Edge) | 2 cores (Area 0) + 3 tunnels FULL |
| Multi-area | `show ip ospf` / `show ip route ospf` | ABR with Areas 0–3; `O IA` routes at branches |
| IPsec SA | `show crypto isakmp sa`, `show crypto ipsec sa` | QM_IDLE, encaps/decaps incrementing |
| HSRP | `show standby brief` | Core1 Active for 10/20, Core2 Active for 30/40/99 |
| Failover | Shut Core1 uplinks during `ping -t` | ≤ 1–3 lost pings, Core2 takes over |
| VLANs/trunks | `show vlan brief`, `show interfaces trunk` | VLANs 10/20/30/40/99 on trunks |
| Finance ACL | Sales → Finance ping | Fails; IT → Finance succeeds |
| Guest ACL | Guest → any 10.x host | Fails; Guest → 198.51.100.x/203.0.113.x OK |
| ACL hits | `show access-lists` | Match counters increment |
| Cross-site | BR1 Sales → BR3 Sales | Succeeds through HQ, encrypted |

## 8. Note on the 99.9% uptime figure
99.9% is the **design target** (no single point of failure at the HQ distribution layer, sub-3-second gateway failover). To back it up in an interview, show the failover test above: record packets lost during a Core1 shutdown and compute availability from that. The ISP link and HQ-Edge remain single points; the next step would be dual edge routers/ISPs.

## 9. Resume-Ready Summary
- Designed a three-tier enterprise topology spanning 3 branches and an HQ data center, simulated in GNS3/Packet Tracer.
- Implemented OSPF multi-area routing with the HQ edge as ABR, and GRE-over-IPsec tunnels (AES-256/SHA-256/DH14) carrying OSPF between sites.
- Deployed HSRP with preemption and uplink tracking, load-sharing VLAN gateways across a core pair.
- Enforced segmentation using 802.1Q VLANs plus standard/extended ACLs to isolate Finance and Guest traffic.
