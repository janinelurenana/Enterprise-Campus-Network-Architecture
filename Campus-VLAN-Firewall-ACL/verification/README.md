Visual evidence confirming the operational status and security enforcement of the build.

## Layer 2 & Connectivity

* **l2-core-show-vlan-brief.png:** Confirms VLAN database consistency on the Core.

* **l2-access-sw1-show-interfaces-trunk.png:** Validates 802.1Q trunking and allowed VLANs.

* **topology.png:** The logical and physical diagram of the environment.

## Security & Routing Enforcement

* **acl-core-show-access-lists.png:** Displays hit counters for ACLs, proving traffic is being actively filtered at the Core.

* **fw-policy-table.png:** Shows the FortiGate security policy sequence, ensuring the "Guest" network is isolated.

* **fw-routing-table.png:** Confirms the default route to the WAN and internal routes back to the Core.

## End-to-End (E2E) Testing

* **e2e-internal-to-internet-ping.png:** Success shot proving NAT and routing are functional for internal users.

* **e2e-guest-to-internal-deny.png:** Confirms that the security posture is successfully blocking Guest access to internal resources.