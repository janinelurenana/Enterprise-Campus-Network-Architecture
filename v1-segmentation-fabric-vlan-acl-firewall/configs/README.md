# Configurations

This folder houses the production-ready configurations for the Phase 1 network infrastructure. The files are organized by functional role to streamline audits and deployment.

## Infrastructure Layer
*Foundational switching and connectivity configurations.*

* **[access-sw1-3.txt](./infrastructure/):** Layer 2 configurations for the access layer, including:
    * VLAN membership and 802.1Q trunking.
    * Port security and STP (Spanning Tree Protocol) optimizations.

## Security Layer
*Perimeter defense and internal traffic filtering.*

* **[core-acls.txt](./security/core-acls.txt):** Standard and extended Access Control Lists implemented on the Core switch to enforce East-West traffic segmentation between departments.
* **[fortigate.txt](./security/fortigate.txt):** Perimeter security configurations, including:
    * **Stateful Inspection:** Security policies for Guest and Internal zones.
    * **Routing:** Static routes back to the Core SVIs.
    * **Translation:** PAT/NAT configurations for outbound Internet access.