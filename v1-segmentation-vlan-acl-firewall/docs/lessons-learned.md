# Lessons Learned

## Technical Lessons

* An **L3 switch can effectively replace a traditional router** for inter-VLAN routing in a campus or small enterprise design, reducing cost and operational complexity.

* **Stateful inspection on the FortiGate** significantly simplified the policy set; unlike the Core ACLs, firewall policies only require a "one-way" definition (source to destination), as the device automatically tracks and permits return traffic via the **session table**. 

* Firewall operation reinforced that **routing correctness is a prerequisite for security policy enforcement**; NAT and policies are irrelevant if the firewall lacks a "map" (static route) back to internal subnets.

* Using a **summarized internal route (10.10.0.0/16)** on the firewall simplified the routing table and made the design easier to scale as VLANs grow.

## Design & Process Lessons

* Separating **east–west traffic control (core ACLs)** from **north–south control (firewall policies)** resulted in clearer responsibility boundaries and easier troubleshooting. This prevents 'hairpinning' traffic to the firewall, preserving WAN-link bandwidth and reducing firewall CPU overhead, while the FortiGate maintains stateful control over North-South (Internet) traffic.

* End-to-end validation **(internal → internet success, guest → internal failure)** was essential to confirm that the design intent matched real network behavior.
  
* **Policy Intent & The Implicit Deny:** The design relies on the **Implicit Deny** to enforce a **zero-trust posture.** While most traffic is dropped silently by the default policy, an **Explicit Deny** was configured for Guest-to-Internal traffic to ensure that unauthorized access attempts from untrusted segments are logged and auditable.
