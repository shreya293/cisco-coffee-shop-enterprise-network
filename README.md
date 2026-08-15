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

## 🗺 Network Architecture & Topology

The topology is structured around a central 24-port switch connected via an 802.1Q trunk uplink to an edge router terminating an ISP uplink.
