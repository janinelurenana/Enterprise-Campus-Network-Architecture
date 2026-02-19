# HR File Server Verification

**Objective:** Verify HR file server accessibility for authorized internal users, and restriction from unauthorized users (Guest VLAN, DMZ, WAN).

**Server Details:**

* IP: 10.10.30.20
* Share: `Public`
* Client used: `wbitt-network-multitool-1`

### Verification Steps

1. List available shares:

```bash
smbclient -L //10.10.30.20 -U labuser
```

*Output shows `Public` share.*

2. Connect to the share:

```bash
smbclient //10.10.30.20/Public -U labuser
```

3. List files inside the share:

```
ls
```

*Shows `connection_test.txt`.*

4. Download test file:

```bash
get connection_test.txt
```

5. Verify file content:

```bash
cat connection_test.txt
```

*Confirms authorized access and proper file retrieval.*

**Screenshots:** All verification evidence is stored in the `screenshots/` folder.

> **Outcome:** HR file server is accessible only to authorized internal users; Guest, DMZ, and WAN access is denied. Firewall policies and ACLs enforce segmentation as intended.
