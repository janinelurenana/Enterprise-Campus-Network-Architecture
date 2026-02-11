# Verification & Validation

Visual evidence confirming the operational status, traffic flow, and security enforcement of the V1 foundation.

## Layer 2 & Connectivity
*Proving the underlying switching fabric is stable and correctly segmented.*

* **[l2-core-show-vlan-brief.png](./l2-switching/l2-core-show-vlan-brief.png):** Confirms VLAN database consistency on the Core and ensures SVIs are associated with the correct subnets.
* **[l2-access-sw1-show-interfaces-trunk.png](./l2-switching/l2-access-sw1-show-interfaces-trunk.png):** Validates 802.1Q trunking and the list of allowed VLANs across the access layer.
* **[topology.png](./topology/topology.png):** High-level logical and physical diagram of the Phase 1 environment.

## Security & Routing Enforcement
*Validating that the "brain" of the network is correctly directing and filtering traffic.*

* **[acl-core-show-access-lists.png](./security-testing/acl-core-show-access-lists.png):** Displays incremental hit counters for ACLs, providing real-time proof that traffic is being actively filtered at the Layer 3 Core.
* **[fw-policy-table.png](./security-testing/fw-policy-table.png):** Demonstrates the FortiGate security policy sequence, specifically focusing on the isolation of the Guest segment.
* **[fw-routing-table.png](./l3-routing/fw-routing-table.png):** Confirms the default route (0.0.0.0/0) to the WAN and the recursive internal routes back to the Core switch.

## End-to-End (E2E) Testing
*Simulated user scenarios to confirm the final "Definition of Done."*

* **[e2e-internal-to-internet-ping.png](./security-testing/e2e-internal-to-internet-ping.png):** Success shot proving that Source NAT (SNAT) and routing are fully functional for internal users.
* **[e2e-guest-to-internal-deny.png](./security-testing/e2e-guest-to-internal-deny.png):** Security validation confirming that the posture is successfully dropping unauthorized traffic from the Guest VLAN to internal resources.