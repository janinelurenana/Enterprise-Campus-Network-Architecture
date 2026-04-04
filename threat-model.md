# Threat Model & Security Considerations

## Scope

This document covers the attack surface of the campus network as designed. It does not account for endpoint security, identity management, or application-layer threats. The focus is network-layer controls: what paths exist, who can use them, and where the enforcement points are.

---

## Trust Zones

| Zone     | Trust Level | Description                                      |
|----------|-------------|--------------------------------------------------|
| WAN      | Untrusted   | External internet; treated as hostile by default |
| DMZ      | Low         | Public-facing; isolated from internal VLANs      |
| Guest    | Untrusted   | Plug-and-play endpoints; no assumed identity      |
| HR       | Medium      | Sensitive data; restricted cross-VLAN access     |
| IT       | High        | Administrative access; broadest internal reach   |
| Mgmt     | High        | Operational access; access to HR only            |

---

## Attack Surface Analysis

### External (WAN → Network)

**Exposed surface:**
- DMZ Nginx server (HTTP/HTTPS via firewall policy)
- FortiGate management interfaces (limited by allowaccess configuration)

**Mitigations:**
- WAN-to-internal traffic is denied at the firewall (implicit deny)
- DMZ is not routed to the core — no lateral path exists even if the DMZ host is compromised
- FortiGate management access is restricted to specific protocols per interface (port1: ping, https, ssh)

**Residual risk:**
- The DMZ web server itself is an attack target. If exploited, the attacker is contained to VLAN 50 with no routed path inward. Firewall logging captures all DMZ-to-internal attempts.

---

### Guest VLAN (Internal Untrusted)

**Threat:** An untrusted device on the Guest VLAN attempts to access internal resources — either through misconfigured policies or by crafting traffic to bypass controls.

**Mitigations:**
- ACL_GUEST_IN denies all traffic from 10.10.40.0/24 to 10.10.0.0/16 with logging
- The same denial is enforced at the FortiGate (GUEST_to_INTERNAL_DENY policy)
- ICMP unreachables are suppressed on the Guest SVI — internal host existence is not advertised
- Guest DHCP is served by the FortiGate, placing dynamic endpoint management at the security boundary

**Residual risk:**
- A VLAN hopping attack (e.g., 802.1Q double-tagging) could bypass access-layer enforcement if native VLAN is not hardened. Native VLAN should be set to an unused VLAN on all trunks.

---

### East-West Lateral Movement

**Threat:** A compromised host in one VLAN attempts to pivot to another VLAN (e.g., HR → IT).

**Mitigations:**
- Per-VLAN inbound ACLs on all SVIs enforce least-privilege policy
- Default deny stance: traffic must be explicitly permitted
- ACL logging captures denied attempts with source and destination
- ACL enforcement occurs at the SVI ingress point, before any routing decision

**Residual risk:**
- ACLs are stateless. A sophisticated attacker who can manipulate TCP flags (e.g., crafting an `established` flag packet) could potentially bypass the return-traffic permit rules. Stateful inspection at the firewall is the correct long-term mitigation.

---

### DMZ Compromise and Pivoting

**Threat:** The DMZ web server is exploited. The attacker attempts to use it as a pivot point into the internal network.

**Mitigations:**
- The DMZ switch has no uplink to the core switching fabric — physical isolation
- All DMZ-to-internal traffic traverses the FortiGate, where ZONE_DMZ → ZONE_INTERNAL is denied
- Core ACLs provide a second denial layer for any traffic that somehow reaches the core

**Residual risk:**
- If the FortiGate itself were compromised, the zone policy would be bypassed. Hardening the FG management plane (no public management access, restricted admin IPs) reduces this risk.

---

### Core Switch Compromise

**Threat:** An attacker gains access to a core switch and modifies ACLs or routing tables.

**Mitigations:**
- SSH is the only remote access method (Telnet not configured)
- SSH encryption restricted to AES CTR variants (no legacy ciphers)
- Management access should be restricted to VLAN 20 (Management) — this is an operational control, not currently enforced by the configuration shown

**Residual risk:**
- No AAA server is configured in this design. Local credentials are a single factor. In production, RADIUS/TACACS+ with centralized authentication is required.

---

### FortiGate HA Manipulation

**Threat:** An attacker triggers a failover to the secondary unit and exploits a configuration gap during the transition window.

**Mitigations:**
- Session synchronization is enabled — in-flight sessions are preserved across failover
- Configuration is synchronized between primary and secondary (checksum matched)
- The 1-packet loss during failover represents a minimal exposure window

**Residual risk:**
- The heartbeat link (port8) is a single point for HA communication. Compromise or failure of this link could cause split-brain behavior. In production, dual heartbeat links on separate physical paths are recommended.

---

## Security Controls Summary

| Control                    | Layer         | Enforcement Point         |
|---------------------------|---------------|---------------------------|
| VLAN segmentation          | Layer 2       | Access switch, trunk pruning |
| East-west ACLs             | Layer 3       | Core switch SVI ingress   |
| Guest internet restriction | Layer 3       | Core ACL + FortiGate policy |
| Perimeter stateful inspection | Layer 4-7  | FortiGate firewall        |
| NAT (internal masking)     | Layer 3       | FortiGate outbound NAT    |
| DMZ isolation              | Layer 2/3     | Physical + FG zone policy |
| SSH-only management        | Management    | Switch configuration      |
| ICMP suppression (Guest)   | Layer 3       | Core SVI                  |

---

## Known Gaps (Production Hardening Required)

- No AAA / centralized authentication
- No 802.1X port authentication on access layer
- Native VLAN not explicitly hardened on trunks
- No IDS/IPS on the FortiGate (policies are permit/deny only)
- No dual heartbeat path for FG HA
- No rate limiting or DHCP snooping on access switches
- Management access not restricted to Management VLAN at the switch CLI level
