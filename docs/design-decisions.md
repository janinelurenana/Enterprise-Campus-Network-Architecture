# Design Decisions & Rationale

This document explains the key architectural choices made in the campus network build — not just what was configured, but why, and what was considered and rejected.

---

## 1. Collapsed Core/Distribution Layer

**Decision:** Combine the distribution and core layers into two redundant switches, rather than maintaining a traditional three-tier hierarchy (access → distribution → core).

**Rationale:** For a single-building campus with a small number of access switches, a three-tier design adds physical complexity and additional failure domains without meaningful benefit. A collapsed core/distribution model retains all the functional capabilities (per-VLAN routing, HSRP, STP root control) while reducing the number of devices and the number of inter-tier links to manage.

**Tradeoff:** The collapsed model does not scale well beyond approximately 10-15 access switches without introducing latency from cross-core traffic. For a larger campus or multi-building design, a dedicated distribution tier becomes necessary.

---

## 2. Static Routing Over OSPF

**Decision:** Use static default routes and summary routes throughout, rather than deploying OSPF or another dynamic routing protocol.

**Rationale:** Static routing was chosen specifically to isolate redundancy validation from routing convergence. When testing HSRP failover or FG HA, static routes ensure that the traffic path does not change unexpectedly due to routing reconvergence — any packet loss observed is directly attributable to the failover mechanism under test, not to a routing update.

Additionally, the network topology is small and stable. There are fewer than ten routed segments. The operational overhead of maintaining static routes is negligible at this scale.

**Tradeoff:** Static routing does not scale. Any topology change requires manual route updates across multiple devices. IP SLA tracking partially mitigates this by withdrawing the primary default route when the upstream gateway is unreachable, but this is not a substitute for dynamic routing in a larger environment.

**Future state:** OSPF with appropriate area design would be the next step, providing sub-second convergence and eliminating manual route management.

---

## 3. ACLs on SVIs vs. Firewall-Based East-West Inspection

**Decision:** Enforce east-west policy using extended ACLs applied inbound on core SVIs, rather than routing all inter-VLAN traffic through the FortiGate.

**Rationale:** SVI-inbound ACLs enforce policy at wire speed on the switching hardware, without adding a hop through the firewall. This keeps inter-VLAN traffic off the firewall data plane, which is a bottleneck compared to switching ASICs. The ACL logic is simple enough to be stateless — the permitted traffic flows are well-defined, and return traffic can be explicitly allowed with minimal rule complexity.

Routing all east-west traffic through the firewall ("hair-pinning") would require the FortiGate to handle significantly more traffic, would introduce the firewall as a point of congestion, and would complicate the routing design.

**Tradeoff:** Stateless ACLs require explicit return-path permits (`permit tcp any any established`, `permit icmp any any echo-reply`). This creates a theoretical gap: a crafted packet with the TCP established bit set could bypass the return-path rule. For a production environment with an active threat model, moving east-west inspection to the firewall (or deploying a dedicated internal inspection device) is the correct long-term answer.

---

## 4. HSRP with Default Timers

**Decision:** Configure HSRP for gateway redundancy but leave hello and hold timers at their defaults (3-second hello, 10-second hold).

**Rationale:** Default timers were retained intentionally to produce measurable, honest convergence data. Tuning timers to subsecond values (e.g., 200ms/700ms) would reduce failover loss to near-zero, but would also increase control-plane overhead and complicate the relationship between HSRP and STP convergence. The 5-packet loss measured at default timers represents a real cost that is documented rather than hidden through tuning.

**Tradeoff:** 5 ICMP packets (~5 seconds) of disruption during gateway failover is acceptable for most batch or background traffic but is noticeable for interactive applications. In an environment with latency-sensitive workloads, timer tuning and BFD integration would be warranted.

---

## 5. X-Mesh Access Uplinks with Rapid-PVST+ (No MEC)

**Decision:** Connect each access switch to both core switches via individual physical links, managed by Rapid-PVST+ rather than Multi-Chassis EtherChannel (MEC).

**Rationale:** MEC (implemented via VSS, vPC, or similar) would allow both uplinks from an access switch to be simultaneously active, eliminating blocked links. However, MEC significantly increases configuration complexity, requires specific platform support, and introduces a tightly-coupled control plane between the core switches. Rapid-PVST+ with X-mesh topology achieves the same physical path diversity with blocking links in steady state and sub-second reconvergence on link failure.

**Tradeoff:** Under normal operation, the cross-links are blocking and represent unused bandwidth. This is an accepted cost at this scale. In a high-density environment where bandwidth utilization is high, MEC would be the correct design choice.

---

## 6. FortiGate Active-Passive HA vs. Active-Active

**Decision:** Deploy FortiGate in Active-Passive mode rather than Active-Active.

**Rationale:** Active-Passive is simpler to validate, produces deterministic failover behavior, and avoids the asymmetric routing complexity that Active-Active introduces. In Active-Active mode, return traffic must arrive at the same firewall unit that processed the original session — which requires careful routing design or session synchronization at both units simultaneously. Active-Passive with session sync provides the same session continuity on failover with significantly less design complexity.

**Tradeoff:** In Active-Passive mode, the secondary unit is idle during normal operation. Its compute capacity is not utilized. For a high-throughput environment, Active-Active would be preferable.

---

## 7. DMZ on a Separate Layer 2 Switch

**Decision:** Connect the DMZ to a dedicated switch attached directly to the FortiGate, with no uplink into the core switching fabric.

**Rationale:** Physical isolation of the DMZ ensures that a misconfigured firewall policy or a routing table error on the core cannot inadvertently create a path from the DMZ to internal VLANs. The only way to reach internal resources from the DMZ is through the FortiGate, where zone policies explicitly deny this traffic. Defense-in-depth requires that the DMZ isolation not depend solely on a software policy.

**Tradeoff:** The DMZ switch is an additional device to manage. In a very small network, a dedicated VLAN on the existing fabric might be acceptable — but the risk model changes, because a VLAN misconfiguration becomes a path to internal resources.

---

## 8. Guest DHCP Delegated to FortiGate

**Decision:** Serve DHCP for VLAN 40 (Guest) from the FortiGate rather than from the core switch.

**Rationale:** Guest endpoints are untrusted by definition. Serving their DHCP from the FortiGate means that the first device a Guest endpoint communicates with is the security boundary — enabling immediate logging, rate limiting, and policy application before the endpoint has any network connectivity. It also creates a clear operational separation: the core switch manages infrastructure and trusted VLANs; the firewall manages untrusted endpoints.

**Tradeoff:** Adds DHCP responsibility to the firewall. In a high-density guest environment (e.g., conference facility), this could become a bottleneck. In that case, a dedicated DHCP server in the DMZ or a separate guest management VLAN would be appropriate.
