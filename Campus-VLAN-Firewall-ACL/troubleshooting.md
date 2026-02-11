# Troubleshooting Notes

This document captures real issues encountered during implementation and the reasoning used to resolve them.  
Each case follows a **symptom → cause → fix → verification** structure.

---

## Case 1: ACL Top-Down Rule Processing Issue

### Symptom
Hosts in **VLAN 10 (10.10.10.0/24)** were able to reach **VLAN 40 (10.10.40.0/24)** despite an explicit deny policy intended to block that traffic.

### Root Cause
Extended ACLs are evaluated **top-down**, and processing stops at the **first matching rule**.

The original ACL contained a broad permit statement placed above a more specific deny:
```
permit ip 10.10.10.0 0.0.0.255 any
deny ip 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255 log
```

Traffic destined for VLAN 40 matched the general permit and was allowed before reaching the deny rule.

### Resolution
Reordered the ACL so that **specific denies precede general permits**, following least-privilege principles:
```
ip access-list extended ACL_IT_IN
permit icmp any any echo-reply
permit ip 10.10.10.0 0.0.0.255 10.10.20.0 0.0.0.255
permit ip 10.10.10.0 0.0.0.255 10.10.30.0 0.0.0.255
deny ip 10.10.10.0 0.0.0.255 10.10.40.0 0.0.0.255 log
permit ip 10.10.10.0 0.0.0.255 any
deny ip any any
```

### Verification
- Ping attempts from VLAN 10 to VLAN 40 fail with **“Administratively prohibited”**
- Allowed inter-VLAN traffic functions as intended
- `show access-lists ACL_IT_IN` confirms hit counters incrementing on the deny rule

### Key Takeaway
ACL behavior is **order-dependent**. Broad permits placed too early can silently bypass security intent. This is a common real-world failure mode when policies grow organically.

---

## Case 2: Firewall Policy Correct but Traffic Still Failing

### Symptom
Internal VLANs could not reach the internet even though:
- Firewall policies were correctly defined
- NAT was enabled
- Policy order appeared correct

Ping attempts to external IPs resulted in timeouts.

### Root Cause
Firewall **routing was incomplete**.

While outbound policies existed, the FortiGate lacked a clear return path for internal subnets.  
Without an explicit route pointing internal networks back to the core L3 switch, return traffic could not be forwarded correctly.

Firewall policy evaluation occurs **after routing**, not before.

### Resolution
Added a summarized static route directing all internal VLANs back to the core switch:

```
config router static
edit 1
set dst 10.10.0.0 255.255.0.0
set gateway 10.10.255.1
set device "port2"
next
end
```

This informs the firewall that all internal VLANs reside behind the core L3 device.

### Verification
- Internal VLANs successfully reach the internet
- `show router info routing-table all` confirms:
  - Default route via WAN interface
  - 10.10.0.0/16 route via core-facing interface
- Firewall logs show sessions correctly established and NAT applied

### Key Takeaway
Firewalls are **routers first, policy engines second**.  
If routing is incorrect or incomplete, perfectly valid security policies will never be evaluated.

---

## Overall Lesson
Most network failures are not caused by missing commands, but by **misunderstanding processing order**:
- Routing before policy
- ACL order before intent
- Specific rules before general ones

This project reinforced the importance of validating control-plane logic before assuming a security misconfiguration.


