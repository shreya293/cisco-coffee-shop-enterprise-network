# cisco-coffee-shop-enterprise-network
Multi-VLAN secure small business enterprise network built in Cisco Packet Tracer featuring Router-on-a-Stick, DHCP pools, Guest ACL isolation, and SSHv2 hardening

readme_content = """# Secure Small-Business Enterprise Network (Coffee Shop Branch)

A production-ready small-business enterprise local area network (LAN) designed and validated in **Cisco Packet Tracer**. This project simulates an end-to-end network implementation for a multi-zone commercial environment ("Mike's Coffee Shop"), demonstrating network segmentation, Inter-VLAN routing (Router-on-a-Stick), dynamic addressing via Cisco IOS DHCP pools, hardened device administration, and security boundaries using extended Access Control Lists (ACLs).

---

## 📌 Table of Contents
1. [Executive Summary & Design Objectives](#-executive-summary--design-objectives)
2. [Network Architecture & Topology](#-network-architecture--topology)
3. [Addressing Scheme & VLAN Segmentation](#-addressing-scheme--vlan-segmentation)
4. [Step-by-Step Configuration Commands](#-step-by-step-configuration-commands)
   - [Switch Hardening & L2 VLAN Configuration](#1-switch-configuration-coffee-sw)
   - [Router-on-a-Stick, DHCP & ACL Security](#2-router-configuration-coffee-shop-rtr)
5. [End-Device & Wireless Setup](#-end-device--wireless-setup)
6. [Configuration Verification (Running-Configs)](#-configuration-verification-running-configs)
7. [Testing & Connectivity Validation](#-testing--connectivity-validation)
8. [Key Engineering Takeaways](#-key-engineering-takeaways)

---

## 🏢 Executive Summary & Design Objectives

In a commercial establishment like a modern café or retail branch, business operations require three core capabilities:
1. **Administrative Operations:** Secure workstation access and network printing for store management.
2. **Point-of-Sale (POS) Transactions:** Reliable, low-latency transaction processing and receipt generation isolated from general traffic.
3. **Public Wireless (Guest Wi-Fi):** Frictionless customer internet access that is strictly quarantined from internal business systems.

### Core Objectives:
* **Zero-Trust Zone Isolation:** Separate management, financial/POS, and guest traffic into distinct Layer 2 broadcast domains (VLANs).
* **Router-on-a-Stick (ROAS):** Enable centralized Layer 3 inter-VLAN routing across a single 802.1Q trunk link.
* **Granular Traffic Quarantining (Extended ACL):** Ensure guest clients can reach DNS/DHCP and external WAN routes while completely dropping traffic headed towards internal subnets.
* **Deterministic IP Planning:** Reserve static pools (`.1`–`.20`) for critical default gateways and network printers, while dynamically leasing addresses (`.21`+) to PCs and wireless endpoints.
* **Management Hardening:** Enforce SSHv2 remote administrative access, MD5/Scrypt privileged secrets, MOTD banners, and console line security.

---

<detail> 
<summary>Network Architecture & Topology</summary>
   
### Topology Diagram (Packet Tracer)
![Network Topology Diagram](images/topology_diagram.jpeg)
</detail>

---
<detail>
<summary> Addressing and VLAN segmentation</summary>
   
---

## 📊 Addressing Scheme & VLAN Segmentation

| VLAN ID | VLAN Name | Subnet | Gateway | Static Pool (Reserved) | DHCP Pool Range | Assigned Ports |
|:---|:---|:---|:---|:---|:---|:---|
| **10** | `managment_office` | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.1` - `.20` | `192.168.10.21` - `.254` | `Fa0/1` - `Fa0/5` |
| **20** | `POS` | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.1` - `.20` | `192.168.20.21` - `.254` | `Fa0/6` - `Fa0/10` |
| **30** | `GUEST_WIFI` | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.1` - `.20` | `192.168.30.21` - `.254` | `Fa0/11` |
| **99** | `Network_Management`| `192.168.99.0/24` | `192.168.99.1` | `192.168.99.1` - `.20` | N/A (Static SVI: `.2`) | Internal / Trunk |
</detail>

---

<detail>
   <summary> Switch Configuration </summary>
   
## ⚙ Step-by-Step Configuration Commands

### 1. Switch Configuration (`Coffee-SW`)

```cisco
! --- Hostname, Passwords & CLI Usability ---
Switch> enable
Switch# configure terminal
Switch(config)# hostname Coffee-SW
Coffee-SW(config)# no ip domain-lookup
Coffee-SW(config)# service password-encryption
Coffee-SW(config)# enable secret Cisco123!
Coffee-SW(config)# banner motd ^C Unauthorized Access Prohibited ^C

! --- SSHv2 Remote Management Setup ---
Coffee-SW(config)# ip domain-name singh.com
Coffee-SW(config)# crypto key generate rsa general-keys modulus 1024
Coffee-SW(config)# ip ssh version 2
Coffee-SW(config)# username admin privilege 15 secret AdminPass123!

! --- Line Hardening ---
Coffee-SW(config)# line con 0
Coffee-SW(config-line)# password ConsolePass123!
Coffee-SW(config-line)# logging synchronous
Coffee-SW(config-line)# exit
Coffee-SW(config)# line vty 0 15
Coffee-SW(config-line)# login local
Coffee-SW(config-line)# transport input ssh
Coffee-SW(config-line)# exit

! --- VLAN Creation & Port Allocation ---
Coffee-SW(config)# vlan 10
Coffee-SW(config-vlan)# name managment_office
Coffee-SW(config)# vlan 20
Coffee-SW(config-vlan)# name POS
Coffee-SW(config)# vlan 30
Coffee-SW(config-vlan)# name GUEST_WIFI
Coffee-SW(config)# vlan 99
Coffee-SW(config-vlan)# name Network_Managment
Coffee-SW(config-vlan)# exit

! --- Assign Access Ports with STP PortFast ---
Coffee-SW(config)# interface range fa0/1 - 5
Coffee-SW(config-if-range)# description #Managment_office#
Coffee-SW(config-if-range)# switchport mode access
Coffee-SW(config-if-range)# switchport access vlan 10
Coffee-SW(config-if-range)# spanning-tree portfast
Coffee-SW(config-if-range)# exit

Coffee-SW(config)# interface range fa0/6 - 10
Coffee-SW(config-if-range)# description #POS#
Coffee-SW(config-if-range)# switchport mode access
Coffee-SW(config-if-range)# switchport access vlan 20
Coffee-SW(config-if-range)# spanning-tree portfast
Coffee-SW(config-if-range)# exit

Coffee-SW(config)# interface fa0/11
Coffee-SW(config-if)# description # Guest_WIFI#
Coffee-SW(config-if)# switchport mode access
Coffee-SW(config-if)# switchport access vlan 30
Coffee-SW(config-if)# spanning-tree portfast
Coffee-SW(config-if)# exit

! --- 802.1Q Trunk Link to Router ---
Coffee-SW(config)# interface gig0/1
Coffee-SW(config-if)# description #TO_Coffee_Shop_RTR#
Coffee-SW(config-if)# switchport trunk encapsulation dot1q
Coffee-SW(config-if)# switchport mode trunk
Coffee-SW(config-if)# switchport trunk allowed vlan 10,20,30,99
Coffee-SW(config-if)# exit

! --- Switch Virtual Interface (SVI) Management IP ---
Coffee-SW(config)# interface vlan 99
Coffee-SW(config-if)# description ##TO_Switch_Managment##
Coffee-SW(config-if)# ip address 192.168.99.2 255.255.255.0
Coffee-SW(config-if)# no shutdown
Coffee-SW(config-if)# exit
Coffee-SW(config)# ip default-gateway 192.168.99.1
Coffee-SW(config)# interface vlan 1
Coffee-SW(config-if)# shutdown
Coffee-SW(config-if)# exit
Coffee-SW(config)# do copy run start
```
</detail> 





