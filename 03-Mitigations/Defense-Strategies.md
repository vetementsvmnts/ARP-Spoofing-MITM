# Defending Against ARP Spoofing & Man-in-the-Middle Attacks

A comprehensive defense guide for network administrators, security engineers, and defensive practitioners.

---

## Table of Contents

1. [Network-Level Defenses](#1-network-level-defenses)
2. [Host-Level Defenses](#2-host-level-defenses)
3. [Encryption & Transport Security](#3-encryption--transport-security)
4. [Detection & Monitoring](#4-detection--monitoring)
5. [Incident Response](#5-incident-response)
6. [Implementation Checklist](#6-implementation-checklist)

---

## 1. Network-Level Defenses

### 1.1 Dynamic ARP Inspection (DAI)

**What it does:** Validates ARP packets on the switch against a trusted database of IP-to-MAC bindings (from DHCP snooping). Drops forged ARP replies before they reach endpoints.

**Implementation (Cisco IOS):**
```cisco
! Enable DHCP snooping first
ip dhcp snooping
ip dhcp snooping vlan 10

! Enable DAI on the VLAN
ip arp inspection vlan 10

! Validate source MAC, destination MAC, and IP
ip arp inspection validate src-mac dst-mac ip

! Configure trusted ports (uplinks to legitimate routers)
interface GigabitEthernet0/1
 ip arp inspection trust
