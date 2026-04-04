# IP Addressing Plan

## Addressing Philosophy

The address plan follows a consistent schema: `10.10.<VLAN-ID>.0/24`. VLAN 10 maps to 10.10.10.0/24, VLAN 20 to 10.10.20.0/24, and so on. This makes the relationship between VLAN and subnet immediately readable in routing tables, ACLs, and firewall logs without consulting a reference document.

Gateway addresses use `.1` consistently across all VLANs. This is the HSRP virtual IP — the address configured on every host as its default gateway. Core switch SVIs use `.2` (CORE-1) and `.3` (CORE-2) so that gateway, primary, and secondary addresses follow a predictable pattern.

Transit links between the core switches and the FortiGate cluster use /30 subnets from the 10.10.255.0/24 range. Using /30s (rather than /31s) maintains compatibility with older IOS versions that do not support /31 point-to-point addressing. The 255.x range is visually distinct in logs and routing tables, making transit links easy to identify.

All internal space fits within a single 10.10.0.0/16 summary. This summary is used in firewall routing, Guest ACL denies, and anywhere a "all internal" shorthand is needed. No subnets are allocated outside this block.

---

## Internal VLANs

| VLAN | Name       | Subnet          | HSRP VIP    | CORE-1 SVI  | CORE-2 SVI  | Trust    |
|-----:|------------|-----------------|-------------|-------------|-------------|----------|
| 10   | IT         | 10.10.10.0/24   | 10.10.10.1  | 10.10.10.2  | 10.10.10.3  | High     |
| 20   | Management | 10.10.20.0/24   | 10.10.20.1  | 10.10.20.2  | 10.10.20.3  | High     |
| 30   | HR         | 10.10.30.0/24   | 10.10.30.1  | 10.10.30.2  | 10.10.30.3  | Medium   |
| 40   | Guest      | 10.10.40.0/24   | 10.10.40.1  | 10.10.40.2  | 10.10.40.3  | None     |

**Usable host range per VLAN:** .2 through .254 (255 total — .1 is the HSRP VIP, .255 is broadcast).

**Host address convention:**
- `.1` — HSRP virtual IP (configured on hosts as default gateway)
- `.2` — CORE-1 SVI
- `.3` — CORE-2 SVI
- `.4` through `.20` — Reserved for infrastructure (servers, management interfaces)
- `.21` through `.254` — Available for end hosts (DHCP pool or static assignment)

---

## DMZ

| VLAN | Name | Subnet        | Gateway     | Notes                     |
|-----:|------|---------------|-------------|---------------------------|
| 50   | DMZ  | 10.10.50.0/24 | 10.10.50.1  | FortiGate DMZ interface   |

The DMZ gateway (.1) is the FortiGate's DMZ-facing interface address. No HSRP is used in the DMZ — the FortiGate HA cluster presents a single virtual MAC on the DMZ interface, so failover is transparent without a separate gateway redundancy protocol.

The Nginx web server is statically assigned an address within 10.10.50.0/24. DHCP is not used in the DMZ; all hosts should have known, fixed addresses for firewall policy and logging purposes.

---

## WAN Segment

| VLAN | Name | Purpose                                   |
|-----:|------|-------------------------------------------|
| 901  | WAN  | Connects ISP uplink, FG-1 WAN, FG-2 WAN  |

The WAN segment uses DHCP addressing assigned by the ISP on port1 of each FortiGate. The specific WAN IP is not documented here as it is dynamically assigned and subject to change. In a production deployment, a static WAN IP or a DHCP reservation from the ISP would be used to ensure consistent inbound NAT behavior.

---

## Core ↔ Firewall Transit Links

Transit links use /30 subnets from 10.10.255.0/24. Each link is a dedicated point-to-point routed segment between one core switch interface and one FortiGate interface.

| Link ID | Subnet           | Core Switch | Core IP       | FortiGate | FG IP         | Interface Pair             |
|---------|------------------|-------------|---------------|-----------|---------------|----------------------------|
| P2P-1   | 10.10.255.0/30   | CORE-1      | 10.10.255.1   | FG-1      | 10.10.255.2   | C1 G0/0 ↔ FG1 port2       |
| P2P-2   | 10.10.255.4/30   | CORE-2      | 10.10.255.5   | FG-2      | 10.10.255.6   | C2 G0/0 ↔ FG2 port4       |
| P2P-3   | 10.10.255.8/30   | CORE-1      | 10.10.255.9   | FG-1      | 10.10.255.10  | C1 G2/0 ↔ FG1 port3       |
| P2P-4   | 10.10.255.12/30  | CORE-2      | 10.10.255.13  | FG-1      | 10.10.255.14  | C2 G2/0 ↔ FG1 port5       |

**Link roles:**

- **P2P-1** (CORE-1 ↔ FG-1): Primary path. CORE-1 default route and FG-1 primary internal route both use this link. IP SLA on CORE-1 probes 10.10.255.2 via this interface.
- **P2P-2** (CORE-2 ↔ FG-2): Primary path for CORE-2. FG-2's primary internal route uses this link.
- **P2P-3** (CORE-1 ↔ FG-1 cross): Cross-link. Provides CORE-1 an alternate path if FG-2's port4 is the active firewall interface. Backup route on CORE-1 (AD 20).
- **P2P-4** (CORE-2 ↔ FG-1 cross): Cross-link. Provides CORE-2 connectivity to FG-1 even when FG-2 is not active. Used by CORE-2's IP SLA probe during HA testing.

The cross-links (P2P-3, P2P-4) exist specifically to handle the FortiGate HA failover scenario. When FG-2 assumes the active role, CORE-1 needs an alternate path to reach the now-active unit. Without cross-links, CORE-1 would only have a path to FG-1 (which is standby) and traffic would black-hole until routing reconverged.

---

## Routing Summary

### CORE-1 Static Routes

| Destination      | Next-Hop      | AD | Condition          | Purpose                                     |
|------------------|---------------|----|--------------------|---------------------------------------------|
| 0.0.0.0/0        | 10.10.255.2   | 1  | Tracked (IP SLA)   | Primary default via FG-1                    |
| 0.0.0.0/0        | 10.10.255.2   | 1  | Always             | Untracked fallback (same next-hop)           |
| 0.0.0.0/0        | 10.10.255.10  | 20 | Always             | Backup default via FG-1 cross-link           |
| 10.10.0.0/16     | 10.10.255.2   | 1  | Always             | Internal summary via FG-1 (for return paths) |
| 10.10.0.0/16     | 10.10.255.10  | 1  | Always             | Internal summary via FG-1 cross-link         |
| 10.10.255.0/30   | 10.10.255.2   | 1  | Always             | Host route: P2P-1 reachability               |
| 10.10.255.4/30   | 10.10.255.6   | 1  | Always             | Host route: P2P-2 reachability               |
| 10.10.255.8/30   | 10.10.255.10  | 1  | Always             | Host route: P2P-3 reachability               |
| 10.10.255.12/30  | 10.10.255.14  | 1  | Always             | Host route: P2P-4 reachability               |

### CORE-2 Static Routes

| Destination      | Next-Hop      | AD | Condition          | Purpose                                      |
|------------------|---------------|----|--------------------|----------------------------------------------|
| 0.0.0.0/0        | 10.10.255.14  | 1  | Tracked (IP SLA)   | Primary default via FG-1 cross-link          |
| 0.0.0.0/0        | 10.10.255.6   | 20 | Always             | Backup default via FG-2                      |
| 0.0.0.0/0        | 10.10.255.10  | 20 | Always             | Backup default via FG-1 cross-link (untracked)|
| 10.10.0.0/16     | 10.10.255.6   | 1  | Always             | Internal summary via FG-2                    |
| 10.10.0.0/16     | 10.10.255.14  | 1  | Always             | Internal summary via FG-1 cross-link         |
| 10.10.255.0/30   | 10.10.255.2   | 1  | Always             | Host route: P2P-1 reachability               |
| 10.10.255.4/30   | 10.10.255.6   | 1  | Always             | Host route: P2P-2 reachability               |
| 10.10.255.8/30   | 10.10.255.10  | 1  | Always             | Host route: P2P-3 reachability               |
| 10.10.255.12/30  | 10.10.255.14  | 1  | Always             | Host route: P2P-4 reachability               |

### FortiGate Static Routes (both units, synced via HA)

| Destination  | Next-Hop      | Device  | AD | Purpose                                 |
|--------------|---------------|---------|----|-----------------------------------------|
| 10.10.0.0/16 | 10.10.255.1   | port2   | 1  | Primary internal path via CORE-1        |
| 10.10.0.0/16 | 10.10.255.13  | port5   | 20 | Backup internal path via CORE-2 cross   |
| 10.10.0.0/16 | 10.10.255.5   | port4   | 1  | Internal path via CORE-2 direct         |
| 10.10.0.0/16 | 10.10.255.9   | port3   | 20 | Backup internal path via CORE-1 cross   |

The FortiGate has no default route — it has a WAN interface in DHCP mode, which installs a default route automatically. The static routes above cover only the internal summary.

---

## Address Allocation Guidelines

**Infrastructure reservations (.1–.20 in each VLAN):**

| Address Range      | Purpose                             |
|--------------------|-------------------------------------|
| x.x.x.1            | HSRP virtual IP (default gateway)   |
| x.x.x.2            | CORE-1 SVI                          |
| x.x.x.3            | CORE-2 SVI                          |
| x.x.x.4–x.x.x.10  | Reserved (servers, management IPs)  |
| x.x.x.11–x.x.x.20 | Reserved (future infrastructure)    |

**End host range (.21–.254):**

Static assignment is preferred for all internal VLANs (10, 20, 30) to ensure auditability. DHCP scopes may be configured on the core switch for temporary or unmanaged devices, with the pool starting at .21 and excluding the reserved range. Guest VLAN (40) uses FortiGate-hosted DHCP; the full .21–.254 range may be used for dynamic assignment.

---

## Subnetting Reference

All /30 transit subnets:

| Subnet           | Network     | Host 1        | Host 2        | Broadcast     |
|------------------|-------------|---------------|---------------|---------------|
| 10.10.255.0/30   | 10.10.255.0 | 10.10.255.1   | 10.10.255.2   | 10.10.255.3   |
| 10.10.255.4/30   | 10.10.255.4 | 10.10.255.5   | 10.10.255.6   | 10.10.255.7   |
| 10.10.255.8/30   | 10.10.255.8 | 10.10.255.9   | 10.10.255.10  | 10.10.255.11  |
| 10.10.255.12/30  | 10.10.255.12| 10.10.255.13  | 10.10.255.14  | 10.10.255.15  |

Remaining /30 subnets in 10.10.255.16/28 through 10.10.255.252/30 are available for future transit links (e.g., second ISP uplink, out-of-band management network, additional firewall segments).
