# Lessons Learned

This document records what broke, what was misunderstood, and what required iteration during the build. It is written honestly — not as a polished retrospective, but as a reference for the next engineer who builds something similar and hits the same walls.

---

## 1. ACL Direction Is Everything

**What happened:** Early in the build, ACLs were applied outbound on SVIs instead of inbound. The rules were syntactically correct and the policy intent was right, but traffic that should have been blocked was passing freely.

**Why it matters:** An outbound ACL on a VLAN SVI evaluates traffic *leaving* the routed interface toward the VLAN — that is, traffic arriving at hosts, not traffic sourced by them. Applying an outbound ACL to block HR→IT traffic does nothing, because by the time the packet reaches the outbound check on the HR SVI, it has already been routed. The correct placement is inbound on the *source* VLAN's SVI, where the packet is intercepted as it enters the routed domain.

**The fix:** Moved all ACLs to inbound. Verified with `show ip access-lists` and confirmed hit counters incremented on deny rules during test traffic.

**Takeaway:** Before writing a single ACL entry, decide the enforcement point. Inbound on the source SVI is the correct default for source-based policy. Outbound is for destination-based policy, which is rarely what you want on an internal SVI.

---

## 2. Both Core Switches Need Identical ACLs

**What happened:** ACLs were initially only configured on CORE-1, with the assumption that CORE-1 was the active HSRP gateway and therefore the only device that needed enforcement. During a failover test, traffic that should have been denied started passing — CORE-2 had no ACLs and was now the active gateway.

**Why it matters:** In an HSRP deployment, either switch can be the active gateway for any VLAN at any time. Policy that lives only on CORE-1 disappears from the data path the moment CORE-2 takes over. This is not a theoretical gap — it was observable during the first HSRP failover test.

**The fix:** Duplicated all ACLs to CORE-2. In production, a configuration management tool or templating system would enforce this automatically and flag drift.

**Takeaway:** Any stateless enforcement mechanism deployed on redundant devices must exist on all devices that can become active. HSRP failover is not just a routing event — it is an enforcement handoff.

---

## 3. Firewall NAT Success Does Not Mean Routing Works

**What happened:** During early connectivity testing, pings from internal hosts to the internet were succeeding. This was interpreted as confirmation that routing, ACLs, and firewall policy were all working correctly. They were not. The ACLs had a misconfiguration that was masked by the fact that internet traffic was working — the misconfigured rules affected east-west traffic, which was not being tested at that point.

**Why it matters:** NAT at the firewall creates an asymmetry in validation. The firewall rewrites source addresses on outbound traffic and uses its session table to handle return traffic. A host can reach the internet with a broken internal routing table, wrong ACL order, or misconfigured SVI, as long as the path to the firewall and back is intact. Internet connectivity passing does not confirm that east-west policy, inter-VLAN routing, or return-path ACLs are correct.

**The fix:** Adopted a layer-by-layer validation sequence: L2 first (VLAN membership, trunk integrity), then L3 (SVI reachability, routing table), then ACL (permit/deny per policy matrix), then firewall (policy table, NAT sessions), then end-to-end. Each layer was confirmed independently before moving to the next.

**Takeaway:** Validate each layer in isolation. A working internet connection is evidence of one narrow path, not the overall network. Always test the cases that should fail, not just the ones that should succeed.

---

## 4. The Implicit Deny Does Not Log

**What happened:** During ACL validation, denied traffic was not appearing in logs. The ACL was working — traffic was being dropped — but there was no log evidence, making it impossible to confirm *which rule* was matching.

**Why it matters:** IOS's implicit `deny ip any any` at the end of every ACL silently drops traffic. It produces no log output. In a lab environment this is an inconvenience; in production it means you have no visibility into what is hitting the default deny.

**The fix:** Added explicit `deny ip any any log` as the final entry in every ACL. This produces syslog output for every packet hitting the default deny, including source IP, destination IP, and protocol. Combined with `show ip access-lists` to inspect per-rule match counts, this makes ACL behavior fully observable.

**Takeaway:** Always make the final deny explicit and always include `log`. The extra line is negligible operationally and the logging visibility is significant. This also documents intent — a reviewer reading the config knows the deny is deliberate.

---

## 5. HSRP Preempt Delay Is Not Optional

**What happened:** After adding HSRP preempt to CORE-1 without a delay, CORE-1 would reclaim the active role immediately on restart — before its routing table, STP topology, and SVI states had fully converged. This caused a brief period where CORE-1 was the HSRP active gateway but was forwarding traffic with an incomplete routing table, producing intermittent drops that were difficult to diagnose.

**Why it matters:** HSRP preempt without delay means the switch asserts active the moment it comes online and its priority is highest. But "online" in HSRP terms is faster than "fully converged" in routing terms. STP can take several seconds to complete, SVIs may come up before their trunk links are fully established, and IP SLA probes need time to determine gateway reachability. Traffic arriving at a preempting switch during this window may be dropped or misrouted.

**The fix:** Added `standby X preempt delay minimum 60` on CORE-1 for all HSRP groups. The 60-second delay is conservative — empirically, full convergence took under 30 seconds — but the extra margin prevents edge cases during simultaneous multi-interface restarts.

**Takeaway:** Always configure a preempt delay. Sixty seconds is a reasonable default. If exact convergence timing matters, measure it: bring up the switch, observe when the routing table stabilizes and STP completes, and set the delay to that value plus a safety margin.

---

## 6. STP Topology Changes Affect the Whole VLAN

**What happened:** During access uplink failure testing, hosts on VLANs other than the one being tested experienced brief connectivity interruptions. The STP topology change notification (TCN) triggered MAC table flushes across all ports in the affected VLAN, not just on the failing link.

**Why it matters:** Rapid-PVST+ is per-VLAN, so a topology change on one VLAN should not affect others. But on an access switch carrying multiple VLANs (ACC-1 carries VLAN 20 and 40), a TCN on either VLAN can cause temporary flooding on the other while MAC tables rebuild. This is expected behavior, but it was not anticipated and caused confusion when connectivity on VLAN 20 was briefly disrupted during a test targeting VLAN 40.

**The fix:** No configuration change was required — the behavior is correct. The insight is that PortFast (now `spanning-tree portfast edge`) must be consistently configured on all host-facing ports to prevent hosts from generating TCNs when they come up or go down. A host port without PortFast will trigger a TCN on link-up, which flushes MAC tables unnecessarily. Audited all access ports and confirmed PortFast was applied uniformly.

**Takeaway:** PortFast on host-facing ports is not just a convenience to avoid the 30-second STP delay — it prevents unnecessary TCN generation. Missing PortFast on even one high-churn port (like a printer or device that frequently power-cycles) can cause periodic MAC table flushes that affect all hosts in the VLAN.

---

## 7. IP SLA Probe Interface Matters

**What happened:** An IP SLA probe was configured to monitor FortiGate reachability but was not bound to a specific source interface. The probe was reaching its target via an unexpected path — a cross-link rather than the primary uplink — which meant the tracking object did not reflect primary link health. The primary uplink could go down and the probe would still succeed, leaving the tracked default route installed and pointing at a dead path.

**Why it matters:** IP SLA probes without a `source-interface` binding will use the routing table to determine their egress path. If multiple paths exist to the probe target, the probe may exit on a different interface than intended. The purpose of the probe — to verify that a *specific* path is up — is undermined if the probe can reach the target via an alternate path.

**The fix:** Added `source-interface GigabitEthernet0/0` (the primary uplink) to each IP SLA probe. This forces the probe out the specific interface being monitored, ensuring that probe failure means that interface's path to the gateway is down.

**Takeaway:** Always bind IP SLA probes to a source interface. Without this, you are testing reachability to a destination, not health of a specific path. These are different things and the distinction matters for failover design.

---

## 8. FortiGate Zone Policy Requires Action to Permit

**What happened:** A firewall policy was configured with `set srcaddr`, `set dstaddr`, and `set service` — but `set action accept` was omitted. The omission was not obvious because the FortiGate CLI accepted the configuration without error. Traffic that should have been permitted was silently denied.

**Why it matters:** In FortiOS, the default action for a policy is deny. `set action accept` must be explicitly stated for permit rules. Unlike IOS ACLs where `permit` is the verb, FortiGate policy entries are objects with an action field that defaults to deny if unset. A policy entry without an explicit action is effectively a deny rule that looks like a permit rule in the config.

**The fix:** Audited all firewall policies and added explicit `set action accept` to every permit rule. Deny rules were left without an action field to make the distinction clear — a missing action now visually signals "this is a deny."

**Takeaway:** In FortiOS, always explicitly set `action accept` on permit policies. Never assume a policy is permitting traffic without confirming it in the policy table (`get firewall policy` or the GUI policy view, which shows the action column clearly). This caught one other subtle issue: a policy that had been edited and accidentally had its action reset to default.

---

## 9. Cross-Links Are Necessary for Firewall HA, Not Just Core Redundancy

**What happened:** The initial topology connected CORE-1 only to FG-1, and CORE-2 only to FG-2, with no cross-links. When FG-2 became the active firewall unit during an HA failover test, CORE-1 had no path to the active firewall. Its default route pointed at FG-1 (now standby), and traffic from CORE-1's VLANs black-holed.

**Why it matters:** FortiGate HA failover does not move IP addresses — FG-2 takes over using a shared virtual MAC, but the routed adjacencies remain tied to physical interface IPs. CORE-1's next-hop (10.10.255.2, FG-1's port2) does not move to FG-2 during failover. Without a cross-link from CORE-1 to FG-2, CORE-1 is isolated from the active firewall.

**The fix:** Added cross-links between CORE-1 and FG-2 (P2P-3, P2P-4), and added backup static routes via those links at AD 20. Tested FG HA failover again — CORE-1 traffic rerouted to the active unit via the cross-link within the IP SLA detection window.

**Takeaway:** In a dual-core, dual-firewall topology, full mesh connectivity between cores and firewalls is required for failover to be seamless. A topology that only connects Core-1→FW-1 and Core-2→FW-2 has a hidden dependency: whichever core's primary firewall is the one that fails, that core loses its upstream path. Cross-links eliminate this dependency.

---

## 10. Validation Requires Testing the Failure Cases, Not Just the Happy Path

**What happened:** Initial testing confirmed that permitted traffic flowed correctly — IT could reach HR, Management could reach the internet, Guest could ping external IPs. The project was nearly considered complete before realizing that none of the denied traffic had been explicitly tested. When it was, two of the deny rules were found to be ineffective due to rule ordering issues caught from lesson 1.

**Why it matters:** A network that passes all its permitted traffic tests may still have security gaps. ACL rules are ordered, and a permit above a deny will shadow the deny for matching traffic. Firewall policies have similar ordering behavior. The only way to confirm that a deny is working is to generate traffic that should be denied and confirm it is dropped.

**The fix:** Built a structured test matrix covering every cell in the east-west policy table — both permitted and denied flows — and ran each case explicitly, checking ACL hit counters before and after to confirm the correct rule matched. Extended this to firewall policies using the FortiGate traffic log.

**Takeaway:** Test every deny explicitly. A security control that has never been tested under adversarial conditions is a security control whose effectiveness is unknown. The test matrix in `/tests/connectivity-baseline/` documents both the permitted and denied cases, and both were verified before the network was considered validated.
