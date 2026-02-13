# Verification & Validation

Visual evidence confirming the operational status, traffic flow, and security enforcement of the V1 foundation.

## Layer 2 Switching

*Proving the underlying switching fabric is stable and correctly segmented.*

* **[show-vlan-brief.png](./l2-switching/show-vlan-brief.png):** Confirms VLAN database consistency and ensures SVIs are associated with the correct subnets.
* **[show-interfaces-trunk.png](./l2-switching/show-interfaces-trunk.png):** Validates 802.1Q trunking and the list of allowed VLANs across the access and core layers.

## Layer 3 Routing & Firewall

*Validating that the "brain" of the network is correctly directing and filtering traffic.*

* **[show-ip-route.png](./l3-routing/show-ip-route.png):** Displays the routing table, confirming static routes, OSPF/EIGRP neighbors, or default gateways.
* **[acl-core-show-access-lists.png](./l3-routing/acl-core-show-access-lists.png):** Displays incremental hit counters for ACLs, providing real-time proof that traffic is being actively filtered at the Layer 3 Core.
* **[fw-policy-table.png](./firewall/fw-policy-table.png):** Demonstrates the FortiGate security policy sequence and rule enforcement.
* **[fw-routing-table.png](./firewall/fw-routing-table.png):** Confirms the firewall's routing table, including WAN reachability and internal return paths.

## End-to-End (E2E) Testing

*Simulated user scenarios to confirm the final "Definition of Done."*

* **[internal-to-internet-ping.png](./e2e/internal-to-internet-ping.png):** Success shot proving that Source NAT (SNAT) and routing are fully functional for internal users.
* **[guest-to-internal-deny.png](./e2e/guest-to-internal-deny.png):** Security validation confirming that the posture is successfully dropping unauthorized traffic from the Guest segment to internal resources.
