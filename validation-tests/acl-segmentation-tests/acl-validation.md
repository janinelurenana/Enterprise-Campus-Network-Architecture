## ACL Validation

ACL enforcement was validated by testing each deny rule explicitly:

| Traffic Scenario                           | ACL Applied        | Expected | Result |
|--------------------------------------------|--------------------|----------|--------|
| Guest (40) → IT subnet (10)                | ACL_GUEST_IN       | Deny     | ✅      |
| Guest (40) → HR subnet (30)                | ACL_GUEST_IN       | Deny     | ✅      |
| HR (30) → IT subnet (10)                   | ACL_HR_IN          | Deny     | ✅      |
| HR (30) → Management subnet (20)           | ACL_HR_IN          | Deny     | ✅      |
| Management (20) → IT subnet (10)           | ACL_MGMT_IN        | Deny     | ✅      |
| IT (10) → Guest subnet (40)                | ACL_IT_IN          | Deny     | ✅      |
| IT (10) → HR subnet (30)                   | ACL_IT_IN          | Permit   | ✅      |
| IT (10) → Management subnet (20)           | ACL_IT_IN          | Permit   | ✅      |
| Internal host → internet (HTTP)            | FW policy          | Permit   | ✅      |
| Return traffic (established TCP)           | ACL (all VLANs)    | Permit   | ✅      |

ACL hit counters (`show ip access-lists`) were inspected after each test to confirm the correct rule matched.

---