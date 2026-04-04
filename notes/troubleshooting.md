# Troubleshooting Log

This document records four failure scenarios encountered during the build and validation of the campus network. Each entry follows the same structure: what was observed, what actually caused it, how it was resolved, and what the underlying principle is. These are not hypothetical — each case represents a real misconfiguration that produced real symptoms, some of which were not immediately obvious from the symptoms alone.

---

## Case 1 — Routing Loop (TTL Exceeded)

### Symptom

End hosts could not reach the internet. `traceroute` showed packets oscillating between the core switch and the FortiGate indefinitely. Ping returned ICMP Type 11, Code 0 — TTL expired in transit.

### Root Cause

A static default route had been manually added to the FortiGate:

```
0.0.0.0/0 → Core Switch (transit IP)
```

The core switches already had their own default routes pointing at the FortiGate:

```
0.0.0.0/0 → FortiGate (transit IP)
```

Both devices were forwarding unknown destinations to each other. Internet-bound traffic from a host entered the core, was sent to the FortiGate, was sent back to the core, and looped until TTL hit zero. This is a textbook bidirectional default route loop — and it can be easy to miss during initial setup because the intent behind each route is individually correct. The core *should* send unknown destinations to the firewall. The problem is that the firewall was doing the same thing in reverse, rather than forwarding to the ISP.

### Resolution

Removed the manual static default route from the FortiGate. The WAN interface (port1) was already configured for DHCP from the ISP, which installs a default route automatically when the interface comes up. The manual route was redundant and conflicting.

After the change, the routing table on the FortiGate showed:

- `0.0.0.0/0` → WAN interface (DHCP-learned)
- `10.10.0.0/16` → core-facing interface (static)

And on the core:

- `0.0.0.0/0` → FortiGate transit IP (static)

The loop resolved immediately. No other configuration changes were required.

Verified with `get router info routing-table all` on the FortiGate and `show ip route` on both core switches.

### Principle

In any edge design with a firewall between the LAN and ISP, exactly one device owns the upstream default route — the firewall, learned from the ISP. Every device behind it points its default at the firewall. The firewall never points its default at the internal network. Any deviation from this creates a loop. This sounds obvious, but it becomes non-obvious when routes are added incrementally during troubleshooting and the full picture is not re-examined.

---

## Case 2 — Asymmetric Path Selection in Full-Mesh Topology

### Symptom

No outage occurred. Traffic was flowing, but path behavior was inconsistent. CORE-1 was forwarding some traffic through FG-2 instead of FG-1, despite the CORE-1 ↔ FG-1 link being fully operational. Packet captures showed asymmetric forwarding: outbound flows leaving CORE-1 via FG-2, return flows arriving via FG-1. This caused intermittent session issues and made failover behavior unpredictable during testing.

### Root Cause

Multiple static routes to the same destination were configured on the FortiGate with equal administrative distance and equal priority. FortiOS treats equal-cost static routes as candidates for ECMP or arbitrary selection — the forwarding behavior is not guaranteed to be deterministic. In a full-mesh topology with four transit links, all pointing at the same 10.10.0.0/16 summary, the FortiGate was distributing traffic across links with no preference ordering.

The issue was not that equal-cost routes are inherently wrong — ECMP is valid in some designs. The problem was that ECMP was not the intended design. The topology was supposed to have a clear primary path with explicit failover order. Equal priorities produced behavior that looked like a misconfiguration even when all links were healthy.

### Resolution

Implemented a cascading priority model on all FortiGate static routes. FortiOS resolves route selection in order: administrative distance first, then priority (lower value preferred). Where distance is equal, priority creates a deterministic tie-breaker without changing convergence behavior.

| Route | Next-Hop | Device | Distance | Priority |
|-------|----------|--------|----------|----------|
| Primary internal path | 10.10.255.1 | port2 | 10 | 10 |
| Secondary internal path | 10.10.255.13 | port5 | 10 | 20 |
| Tertiary internal path | 10.10.255.5 | port4 | 10 | 30 |
| Fallback internal path | 10.10.255.9 | port3 | 10 | 40 |

With this ordering, the primary path is always preferred when healthy. Backup paths activate in sequence only when the preferred path fails. Traffic symmetry was restored and failover behavior became verifiable.

### Principle

Redundancy without explicit ordering is not a controlled failover design — it is non-determinism with extra links. In any multi-path topology, every route must have an unambiguous preference. Equal-cost routes are only appropriate when the intent is genuine load-balancing with symmetric return paths, which requires additional design consideration. When the intent is failover, priorities must be distinct.

---

## Case 3 — ACL Rule Shadowing (Broad Permit Before Specific Deny)

### Symptom

Hosts in VLAN 10 (IT, 10.10.10.0/24) could reach VLAN 40 (Guest, 10.10.40.0/24). The ACL applied to the VLAN 10 SVI contained an explicit deny rule for this traffic. The rule existed, but was never matching — hit counters on the deny rule showed zero.

### Root Cause

The ACL had a broad permit statement placed above the specific deny:

```
permit ip 10.10.10.0 0.0.0.255 any
deny   ip 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255 log
```

IOS ACLs are evaluated top-down and stop at the first match. Traffic from IT to Guest matched `permit ip ... any` before reaching the deny. The deny rule was syntactically correct and intentionally placed — it was simply unreachable. This type of error does not generate a warning or error message. The ACL applies without complaint, and the only indication that something is wrong is that denied traffic passes.

### Resolution

Reordered the ACL so that specific denies precede the broad permit. The correct structure places return-traffic permits first, then specific denies for restricted destinations, then specific permits for allowed destinations, then the broad permit for internet access, then the explicit default deny:

```
ip access-list extended ACL_IT_IN
 permit icmp any any echo-reply
 permit tcp any any established
 deny   ip 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255 log
 permit ip 10.10.10.0 0.0.0.255 10.10.20.0 0.0.0.255
 permit ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255
 permit ip 10.10.10.0 0.0.0.255 any
 deny   ip any any log
```

After the change, `show ip access-lists ACL_IT_IN` confirmed hit counters incrementing on the deny rule during test traffic. Pings from VLAN 10 to VLAN 40 were dropped.

### Principle

ACL rule ordering is the most common source of silent policy failures. A rule that is never hit is a rule that does not exist from a traffic perspective, regardless of what the config looks like. The discipline is simple: specific rules before general rules, always. When diagnosing unexpected permit behavior, the first check is hit counters — a deny rule with zero hits is either unreachable or the traffic is not arriving on that interface. Both are worth investigating.

---

## Case 4 — Firewall Policy Correct, Routing Incomplete

### Symptom

Internal hosts could not reach the internet. Firewall policies were in place, NAT was enabled, and policy order was verified. The FortiGate traffic log showed no matching sessions — traffic was not reaching the policy engine at all. Ping attempts from all internal VLANs timed out.

### Root Cause

The FortiGate had no route for the internal subnets. The firewall knew how to reach the WAN (default route via DHCP) but had no entry for 10.10.0.0/16. When a return packet arrived from the internet destined for an internal host, the FortiGate could not determine where to forward it. Without a routing entry, the packet was dropped before policy evaluation — the firewall never had a chance to consult its session table or apply NAT in reverse.

This is a conceptually important failure mode: firewall policy is evaluated after the routing decision. If the routing table cannot resolve the destination, the packet is dropped regardless of what the policy says. A firewall with perfect policy and no internal routes will drop all return traffic silently.

### Resolution

Added a static summary route covering all internal VLANs, pointing at the core switch:

```
config router static
    edit 1
        set dst 10.10.0.0 255.255.0.0
        set gateway 10.10.255.1
        set device "port2"
    next
end
```

After the change, `get router info routing-table all` on the FortiGate confirmed:

- `0.0.0.0/0` → WAN interface (outbound path)
- `10.10.0.0/16` → port2, next-hop 10.10.255.1 (return path)

Internal hosts could immediately reach the internet. Firewall logs showed sessions being established and NAT being applied correctly.

### Principle

A firewall is a router with a policy engine attached. Routing is evaluated first; policy second. This means a firewall with no route for a destination will drop traffic before policy is ever consulted — and the traffic log may show nothing at all, because the packet was discarded at the routing stage. When firewall policies appear correct but traffic is still failing, verify the routing table before touching the policies. `get router info routing-table all` (FortiGate) or the equivalent on other platforms should be the first diagnostic step, not the last.

---

## Summary

| Case | Symptom | Root Cause | Category |
|------|---------|------------|----------|
| 1 | TTL exceeded, packets looping | Bidirectional default route conflict | Routing design |
| 2 | Asymmetric forwarding, inconsistent paths | Equal-priority static routes, no failover order | Route preference |
| 3 | Denied traffic passing, zero hits on deny rule | Broad permit shadowing specific deny | ACL ordering |
| 4 | No internet despite correct policy, empty traffic log | Missing internal return route on firewall | Routing completeness |

Each of these failures was invisible at the interface level — links were up, devices were reachable, and configurations looked plausible. The failures only became visible through deliberate traffic testing and inspection of routing tables, ACL hit counters, and firewall session logs. The diagnostic method mattered as much as the fix.
