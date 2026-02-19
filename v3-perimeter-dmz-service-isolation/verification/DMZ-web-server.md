# DMZ Web Server Verification

**Objective:** Validate that the DMZ web server is operational while remaining isolated from internal networks.

### Evidence

| Verification Item                | Purpose                                                   | Screenshot                |
| -------------------------------- | --------------------------------------------------------- | ------------------------- |
| DMZ Topology                     | Confirms DMZ segment is physically and logically isolated | `topology.png`        |
| DMZ Server IP                    | Confirms correct IP/subnet assignment                     | [dmz_server_ip.png](./screenshots/dmz_server_ip.png)       |
| Firewall Policy (WAN → DMZ HTTP) | Validates explicit allowance of HTTP/HTTPS traffic        | `dmz_firewall_policy.png` |
| Web Service Response             | Confirms HTTP requests succeed                            | [dmz_web_response.png](./screenshots/dmz_web_response.png)    |

> **Outcome:** DMZ web server is reachable externally via HTTP/HTTPS, internal networks remain protected, and inter-zone policies are enforced.
