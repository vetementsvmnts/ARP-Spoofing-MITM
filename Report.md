# ARP Spoofing & Credential Interception -- Technical Report

**Author:** Kitsana Thuekoh 
**Date:** May 2026  
**Classification:** Educational / Lab Environment Only  
**Environment:** Isolated VirtualBox Internal Network

---

## 1. Executive Summary

This report documents a successful Man-in-the-Middle (MitM) attack executed via ARP cache poisoning in a controlled laboratory environment. The attack achieved complete traffic interception between a victim workstation and the network gateway, resulting in the extraction of plaintext HTTP login credentials.

**Key Findings:**
- ARP protocol lacks authentication mechanisms -- devices unconditionally trust ARP replies
- Unencrypted HTTP traffic exposes sensitive credentials when intercepted in transit
- Bidirectional ARP poisoning maintains network connectivity while enabling silent surveillance
- Modern mitigations (DAI, HTTPS, port security) effectively neutralize this attack vector when properly implemented

---

## 2. Lab Environment Architecture

### 2.1 Network Topology

![Lab Topology](setup/lab-diagram.png)

### 2.2 System Specifications

| Component | Operating System | IP Address | MAC Address | Role |
|-----------|-----------------|------------|-------------|------|
| Attacker | Kali Linux 2026.1 | 192.168.10.27 | 08:00:27:8a:35:d2 | ARP spoofer, traffic interceptor, packet analyzer |
| Victim | Ubuntu 16.04 LTS | 192.168.10.221 | 08:00:27:91:b9:31 | Target user browsing HTTP login portals |
| Gateway | VirtualBox NAT/Internal Router | 192.168.10.1 | [Redacted] | Default route, internet access relay |

### 2.3 Isolation Measures

- **Network Mode:** VirtualBox Internal Network (no host access, no internet bridging)
- **Scope:** Single /24 subnet with controlled traffic only
- **Authorization:** Full ownership of all systems; no external infrastructure involved

---

## 3. Theoretical Background

### 3.1 Address Resolution Protocol (ARP)

ARP operates at Layer 2 (Data Link) of the OSI model, mapping 32-bit IPv4 addresses to 48-bit MAC addresses on local networks. When a device needs to communicate with another device on the same subnet, it broadcasts an ARP request: "Who has IP X? Tell MAC Y."

The critical vulnerability lies in ARP's **stateless, unauthenticated design**:
- No verification of reply legitimacy
- No challenge-response mechanism
- Devices update cache entries upon receiving *any* ARP reply, even unsolicited ones
- No central authority to validate IP-to-MAC bindings

### 3.2 ARP Cache Poisoning

An attacker on the same broadcast domain can send forged ARP replies containing:
1. **Target IP address** (e.g., gateway at 192.168.10.1)
2. **Attacker's MAC address** (e.g., 08:00:27:8a:35:d2)

The victim updates its ARP cache, believing the attacker is the gateway. All traffic destined for the gateway now flows through the attacker's machine. Bidirectional poisoning (targeting both victim and gateway) ensures full duplex interception.

### 3.3 Man-in-the-Middle Position

Once ARP tables are poisoned:
- **Victim -&gt; Gateway traffic** passes through attacker
- **Gateway -&gt; Victim traffic** passes through attacker
- **IP forwarding** on the attacker maintains connectivity -- victims experience no service disruption
- **Silent interception** occurs without user awareness

---

## 4. Attack Methodology

### Phase 1: Reconnaissance (T1590 -- Gather Victim Network Information)

**Objective:** Identify network topology, active hosts, and gateway location.

```bash
# Attacker network interface enumeration
$ ip addr show
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt;
    link/ether 08:00:27:8a:35:d2
    inet 192.168.10.27/24 brd 192.168.10.255 scope global dynamic noprefixroute eth0

# Default gateway identification
$ ip route | grep default
default via 192.168.10.1 dev eth0 proto dhcp
