
# Secure Small-Business Enterprise Network (Coffee Shop Branch)

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

                 +-----------------------+
                 |       Cloud ISP       |
                 +-----------+-----------+
                             | (Gig0/0 - WAN)
                 +-----------+-----------+
                 |    Coffee-Shop RTR    | (Cisco 2911)
                 |   192.168.x.1 Gateways|
                 +-----------+-----------+
                             | (Gig0/1 - 802.1Q Trunk)
                 +-----------+-----------+
                 |       Coffee-SW       | (Cisco 2960)
                 |   192.168.99.2 (SVI)  |
                 +---+-------+-------+---+
                     |       |       |
     +---------------+       |       +---------------+
     | (Fa0/1 - Fa0/5)       | (Fa0/6 - Fa0/10)      | (Fa0/11)
     v                       v                       v
+------------------+    +------------------+    +------------------+
|  VLAN 10: Office |    |   VLAN 20: POS   |    | VLAN 30: Guest   |
| - Manager PC     |    | - POS Terminal   |    | - AccessPoint-PT |
| - Office Printer |    | - Receipt Printer|    | - Guest Laptops  |
+------------------+    +------------------+    +------------------+


### Topology Diagram (Packet Tracer)
![Network Topology Diagram](images/topology_diagram.jpeg)

---

## 📊 Addressing Scheme & VLAN Segmentation

| VLAN ID | VLAN Name | Subnet | Gateway | Static Pool (Reserved) | DHCP Pool Range | Assigned Ports |
|:---|:---|:---|:---|:---|:---|:---|
| **10** | `managment_office` | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.1` - `.20` | `192.168.10.21` - `.254` | `Fa0/1` - `Fa0/5` |
| **20** | `POS` | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.1` - `.20` | `192.168.20.21` - `.254` | `Fa0/6` - `Fa0/10` |
| **30** | `GUEST_WIFI` | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.1` - `.20` | `192.168.30.21` - `.254` | `Fa0/11` |
| **99** | `Network_Management`| `192.168.99.0/24` | `192.168.99.1` | `192.168.99.1` - `.20` | N/A (Static SVI: `.2`) | Internal / Trunk |

---

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
2. Router Configuration (Coffee-Shop RTR)
Cisco CLI
! --- Hostname & System Security ---
Router> enable
Router# configure terminal
Router(config)# hostname RTR
RTR(config)# no ip domain-lookup
RTR(config)# service password-encryption
RTR(config)# enable secret Cisco123!
RTR(config)# banner motd ^C Unautorized access banned ^C

! --- Line Hardening ---
RTR(config)# line con 0
RTR(config-line)# password ConsolePass123!
RTR(config-line)# logging synchronous
RTR(config-line)# exit
RTR(config)# line vty 0 4
RTR(config-line)# login
RTR(config-line)# exit

! --- WAN Interface to ISP (Pre-staged) ---
RTR(config)# interface gig0/0
RTR(config-if)# description ## to_ISP##
RTR(config-if)# no shutdown
RTR(config-if)# exit

! --- Router-on-a-Stick Sub-interfaces ---
RTR(config)# interface gig0/1
RTR(config-if)# description ## to_SW1 ##
RTR(config-if)# no ip address
RTR(config-if)# no shutdown
RTR(config-if)# exit

RTR(config)# interface gig0/1.10
RTR(config-subif)# description ##Management_Office##
RTR(config-subif)# encapsulation dot1Q 10
RTR(config-subif)# ip address 192.168.10.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.20
RTR(config-subif)# description ## POS_Gateway##
RTR(config-subif)# encapsulation dot1Q 20
RTR(config-subif)# ip address 192.168.20.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.30
RTR(config-subif)# description ##GUEST_WIFI##
RTR(config-subif)# encapsulation dot1Q 30
RTR(config-subif)# ip address 192.168.30.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.99
RTR(config-subif)# description ##Network_Managment_gateway##
RTR(config-subif)# encapsulation dot1Q 99
RTR(config-subif)# ip address 192.168.99.1 255.255.255.0
RTR(config-subif)# exit

! --- DHCP Server Pools Configuration ---
RTR(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.20
RTR(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.20
RTR(config)# ip dhcp excluded-address 192.168.30.1 192.168.30.20

RTR(config)# ip dhcp pool managment_office
RTR(dhcp-config)# network 192.168.10.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.10.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool POS
RTR(dhcp-config)# network 192.168.20.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.20.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool GUEST_WIFI
RTR(dhcp-config)# network 192.168.30.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.30.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

! --- Extended Access Control List (Guest Network Isolation) ---
RTR(config)# ip access-list extended GUEST
RTR(config-ext-nacl)# permit udp any eq bootpc any eq bootps
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
RTR(config-ext-nacl)# permit ip 192.168.30.0 0.0.0.255 any
RTR(config-ext-nacl)# exit

! --- Apply ACL Inbound on Guest Sub-interface ---
RTR(config)# interface gig0/1.30
RTR(config-subif)# ip access-group GUEST in
RTR(config-subif)# exit
RTR(config)# do copy run start
🖨 End-Device & Wireless Setup
Static IP Device Assignments
1. Office Printer (VLAN 10)
IP Address: 192.168.10.10

Subnet Mask: 255.255.255.0

Default Gateway: 192.168.10.1

DNS Server: 8.8.8.8

Office Printer Global Settings	Office Printer FastEthernet0 Config
2. Receipt Printer (VLAN 20)
IP Address: 192.168.20.10

Subnet Mask: 255.255.255.0

Default Gateway: 192.168.20.1

DNS Server: 8.8.8.8

Receipt Printer Global Settings	Receipt Printer FastEthernet0 Config
Wireless Access Point & Client Associations
SSID: Coffee-SHop-Guest

Channel: Channel 6 (2.4 GHz)

Security Protocol: WPA2-PSK (AES Encryption)

Pre-shared Key (PSK): CofShop!!!

Access Point Port 1 Setup	Client WPA2-Personal Authentication
Guest Client Association Status	Guest Laptop 0 DHCP Leasing (192.168.30.21)
📜 Configuration Verification (Running-Configs)
Router Running-Config Proofs (RTR# do sh run)
Router Services & DHCP Exclusions	Router DHCP Pools & Hardware UDI
Router Sub-Interfaces (Gi0/1.10, Gi0/1.20, Gi0/1.30, Gi0/1.99)	Router Extended ACL (GUEST) & Line Configs
Switch Running-Config Proofs (Coffee-SW# sh run)
Switch Hostname, Secrets & SSH Config	Switch Access Ports (Fa0/1 – Fa0/5 in VLAN 10)
Switch Access Ports (Fa0/6 – Fa0/11 in VLAN 20 & 30)	Switch Trunk Uplink (Gi0/1) & Management SVI (Vlan99)	Switch Line VTY & Console Security
🧪 Testing & Connectivity Validation
1. Inter-VLAN Routing & Office LAN Validation
From Manager PC (192.168.10.x), traffic successfully reaches the local printer (192.168.10.10) and default gateway (192.168.10.1) with <1ms round-trip latency.

2. POS Terminal Connectivity
The POS Terminal verifies instant communication with its default gateway (192.168.20.1) and receipt printer (192.168.20.10) with 0% packet loss.

3. Guest ACL Isolation & Quarantine Test
From Laptop0 (192.168.30.21):

Ping to POS Subnet (192.168.20.1): Destination host unreachable (Blocked by ACL GUEST).

Ping within Guest Subnet (192.168.30.22): 4/4 packets successful (Intra-subnet communication working).

💡 Key Engineering Takeaways
DHCP Bootstrapping inside ACLs: When applying an inbound ACL on a DHCP client subnet, standard deny rules will drop broadcast DHCP requests unless permit udp any eq bootpc any eq bootps is placed at top-of-stack.

Deterministic Port Mapping: Grouping switch ports sequentially (1-5 Office, 6-10 POS, 11 AP) reduces administrative overhead and makes on-site physical troubleshooting straightforward.

Defense-in-Depth for Management: Disabling telnet across VTY lines via transport input ssh and routing management through an isolated SVI (VLAN 99) protects infrastructure from rogue LAN discovery.
"""

with open("README.md", "w") as f:
f.write(readme_content)

print("README.md generated successfully.")


```text?code_stdout&code_event_index=1
README.md generated successfully.

Your Markdown file is ready:
[file-tag: code-generated-file-abeec21a-7867-45b3-8a4e-e6bcfdd93de6]

Suggested Image File Mapping for Your Repository
To display your screenshots seamlessly in your GitHub repo, create an images/ directory and rename your uploaded screenshots to the corresponding filenames used in the README.md:

Image / Screenshot Content	Recommended File Name in images/ folder
Topology diagram	images/topology_diagram.jpeg
Router show run (Part 1 - Services/DHCP Exclusions)	images/rtr_sh_run_1.jpeg
Router show run (Part 2 - DHCP Pools/UDI)	images/rtr_sh_run_2.jpeg
Router show run (Part 3 - Sub-interfaces)	images/rtr_sh_run_3.jpeg
Router show run (Part 4 - ACL & Lines)	images/rtr_sh_run_4.jpeg
Switch show run (Part 1 - Secrets & SSH)	images/sw_sh_run_1.jpeg
Switch show run (Part 2 - VLAN 10 Ports)	images/sw_sh_run_2.jpeg
Switch show run (Part 3 - VLAN 20 & 30 Ports)	images/sw_sh_run_3.jpeg
Switch show run (Part 4 - Trunk & SVI)	images/sw_sh_run_4.jpeg
Switch show run (Part 5 - VTY & Console)	images/sw_sh_run_5.jpeg
Office Printer Global Config	images/printer_office_global.jpeg
Office Printer FastEthernet IP	images/printer_office_ip.jpeg
Receipt Printer Global Config	images/printer_receipt_global.jpeg
Receipt Printer FastEthernet IP	images/printer_receipt_ip.jpeg
Access Point Port Config	images/ap_port_config.jpeg
Laptop WPA2 Connect Screen	images/laptop_wpa2_connect.jpeg
Laptop Wireless Monitor Active	images/laptop_wireless_monitor.jpeg
Laptop 0 DHCP Verification	images/laptop0_dhcp.jpeg
Laptop 1 DHCP Verification	images/laptop1_dhcp.jpeg
Manager PC Ping Verification	images/manager_pc_ping_test.jpeg
POS Terminal Ping Verification	images/pos_terminal_ping_test.jpeg
Guest Laptop ACL Blocking Ping	images/guest_laptop_ping_acl_test.jpeg
Ready-to-Post LinkedIn Caption
🚀 Designing & Building a Multi-VLAN Secure Small-Business Network in Cisco Packet Tracer ☕🌐

Small business environments—like cafés, retail branches, and clinics—often handle three competing networking needs: secure internal operations, PCI-compliant Point-of-Sale (POS) transactions, and open guest Wi-Fi.

I recently built and simulated an end-to-end small enterprise network architecture designed to solve this with clear network segmentation and zero-trust isolation principles.

🔹 Architecture & Implementation Highlights:
• VLAN Segmentation: Split the LAN into Management Office (VLAN 10), POS Systems (VLAN 20), Guest Wi-Fi (VLAN 30), and Network Infrastructure Management (VLAN 99).
• Inter-VLAN Routing: Implemented Router-on-a-Stick (ROAS) with 802.1Q sub-interfaces on a Cisco 2911 router to centralize routing over a single trunk.
• Dynamic & Static IP Allocation: Configured Cisco IOS DHCP pools with reserved static ranges (.1–.20) for gateways and printers.
• Traffic Isolation & Security: Designed an inbound Extended Access Control List (ACL) permitting DHCP bootstrap traffic while dropping any guest attempts to access internal office subnets or management interfaces.
• Device Hardening: Secured management access with SSHv2, dedicated privilege levels, encrypted credentials, STP PortFast on edge ports, and custom MOTD banners.

📂 GitHub Repository: [Insert your GitHub Repo Link]
📺 Reference Video Inspiration: Build a Coffee Shop Network in Cisco Packet Tracer

#Networking #Cisco #PacketTracer #NetworkEngineering #VLAN #Routing #CyberSecurity #CCNA #HandsOnLab


Build a Coffee Shop Network in Cisco Packet Tracer (Step-by-Step)
ITTechStudies · 37K views

 can you provide me the beginning details, like what should be my repository name, description, and everything related to IT

Here is the setup profile for your GitHub repository to make it look clean, professional, and well-structured.

1. Repository Name Options
Choose a concise, industry-standard naming format:

Recommended: cisco-coffee-shop-enterprise-network

Alternative 1: secure-smb-network-packet-tracer

Alternative 2: cisco-packet-tracer-multi-vlan-lab

2. Repository Description & Metadata
Paste these directly into the repository settings header on GitHub:

Short Description:

Multi-VLAN secure small business enterprise network built in Cisco Packet Tracer featuring Router-on-a-Stick, DHCP pools, Guest ACL isolation, and SSHv2 hardening.

Website / Reference URL (Optional):

[https://youtu.be/pKnmaU1zhUI](https://youtu.be/pKnmaU1zhUI) (or your LinkedIn post link once published)

Repository Topics / Tags:

cisco packet-tracer ccna networking network-engineering vlan router-on-a-stick dhcp access-control-lists network-security layer2-switching ssh

3. Recommended Repository File Structure
Organize your GitHub repository like this so all the image links in your README.md render immediately:

Plaintext
cisco-coffee-shop-enterprise-network/
│
├── README.md                          <-- (The complete markdown documentation file)
├── LICENSE                            <-- (MIT License recommended)
│
├── configs/                           <-- (Optional: plain text backups of your CLI configs)
│   ├── Router_running_config.txt
│   └── Switch_running_config.txt
│
├── topologies/                        <-- (Packet Tracer save file)
│   └── Coffee_Shop_Topology.pkt
│
└── images/                            <-- (All your exported screenshots)
    ├── topology_diagram.jpeg
    ├── rtr_sh_run_1.jpeg
    ├── rtr_sh_run_2.jpeg
    ├── rtr_sh_run_3.jpeg
    ├── rtr_sh_run_4.jpeg
    ├── sw_sh_run_1.jpeg
    ├── sw_sh_run_2.jpeg
    ├── sw_sh_run_3.jpeg
    ├── sw_sh_run_4.jpeg
    ├── sw_sh_run_5.jpeg
    ├── printer_office_global.jpeg
    ├── printer_office_ip.jpeg
    ├── printer_receipt_global.jpeg
    ├── printer_receipt_ip.jpeg
    ├── ap_port_config.jpeg
    ├── laptop_wpa2_connect.jpeg
    ├── laptop_wireless_monitor.jpeg
    ├── laptop0_dhcp.jpeg
    ├── laptop1_dhcp.jpeg
    ├── manager_pc_ping_test.jpeg
    ├── pos_terminal_ping_test.jpeg
    └── guest_laptop_ping_acl_test.jpeg
4. Git Commit & Push Quick Commands
When uploading from your terminal:

Bash
# Initialize git in your project directory
git init

# Add remote repo
git remote add origin https://github.com/<your-username>/cisco-coffee-shop-enterprise-network.git

# Stage files
git add .

# Create initial commit
git commit -m "feat: complete multi-VLAN coffee shop network topology with full CLI proofs & README"

# Push to main branch
git branch -M main
git push -u origin main
99 actually created fall because when we just see the switch carefully we understand that we never used VLAN 99 for the interfaces and also we have assigned the IP addresses for the VLAN 99. In order to make the layer to switch as layer three, we can easily use IP routing, but instead of that, we have choose the way of default gateway, assigning the IP address like 192.168.99.1. Can you explain me why it is done?

Here is the breakdown of why VLAN 99 is designed this way and why ip default-gateway is used instead of ip routing.

1. Why VLAN 99 Has No Physical Access Ports
VLAN 99 is dedicated purely as an Infrastructure Management VLAN, not a user access network.

Separation of Management and Data Planes:

End-user devices and printers live in VLAN 10 (Office), VLAN 20 (POS), and VLAN 30 (Guest).

If the switch’s management interface (SVI) shared an IP range with users (such as VLAN 10), any compromised office PC or guest could scan and attempt brute-force attacks on the switch CLI.

No Access Ports Needed:

No PC or printer should ever be plugged into a switch port assigned to VLAN 99.

The only endpoints inside VLAN 99 are:

interface Vlan99 (SVI on the Switch): 192.168.99.2

GigabitEthernet0/1.99 (Sub-interface on the Router): 192.168.99.1

This traffic travels exclusively across the 802.1Q trunk link (Gi0/1), keeping network management completely isolated from user data.

2. Why Use ip default-gateway Instead of ip routing?
There are two primary architectural reasons for using ip default-gateway here:

A. Hardware Constraints of Layer 2 Switches (e.g., Cisco Catalyst 2960)
A standard Layer 2 switch cannot route packets between different IP subnets. It operates at Layer 2 (MAC address tables) and does not support the ip routing command.

Because the switch only needs an IP address for its own management (so administrators can SSH into it), it behaves like an endpoint/host on the network rather than a router.

B. How the Switch Sends Return Traffic
When you connect via SSH to 192.168.99.2 from a workstation in a different subnet (e.g., Manager PC on 192.168.10.x):

The packet arrives at the switch through the router's sub-interface (Gi0/1.99).

The switch accepts the packet, but to reply back to 192.168.10.x, it needs to know where to send traffic that is outside of the 192.168.99.0/24 subnet.

Since a Layer 2 switch has no dynamic routing table, it relies on ip default-gateway 192.168.99.1 to forward all return packets back to the router.

3. Centralized Security Control (Router-on-a-Stick)
By terminating all VLAN gateways (including VLAN 99) on the router (Coffee-Shop RTR):

Centralized Policy Enforcement: The router acts as the single security inspection point.

ACL Enforcement: The Extended Access Control List (GUEST) applied on Gi0/1.30 can explicitly block 192.168.30.0/24 from ever reaching 192.168.99.0/24 (Network Management), preventing rogue Wi-Fi clients from communicating with the switch or router management interfaces.

Summary Comparison
Feature	Layer 2 Switch Setup (This Project)	Layer 3 Switch Setup (ip routing)
Inter-VLAN Routing	Handled by Router (Coffee-Shop RTR)	Handled directly by Switch
Gateway Command	ip default-gateway 192.168.99.1	ip routing + Static/Dynamic routes
Switch Role	Pure switching + Management Host	Multi-layer Routing & Switching
Cost / Real-world Use	Standard budget SMB branch deployment	High-throughput core/distribution layers
what my readme.md file constains??

The generated README.md file contains a complete, production-style technical documentation structured into the following sections:

Executive Summary & Design Objectives: The business rationale for building a segmented network for "Mike's Coffee Shop," outlining objectives like Zero-Trust isolation, centralized routing, and device hardening.

Network Architecture & Topology: An ASCII flow diagram of the network path (ISP → Router → Switch → End Devices) and an embedded image placeholder for the Packet Tracer topology diagram.

Addressing Scheme & VLAN Segmentation Table: A clean reference table detailing:

VLAN 10 (managment_office): 192.168.10.0/24, Gateway: .1, Reserved: .1–.20, Ports Fa0/1–Fa0/5.

VLAN 20 (POS): 192.168.20.0/24, Gateway: .1, Reserved: .1–.20, Ports Fa0/6–Fa0/10.

VLAN 30 (GUEST_WIFI): 192.168.30.0/24, Gateway: .1, Reserved: .1–.20, Port Fa0/11.

VLAN 99 (Network_Management): 192.168.99.0/24, Gateway: .1, Switch SVI: .2.

Step-by-Step Configuration Commands (Extracted CLI Scripts):

Switch (Coffee-SW): Hostname, password encryption, secret, MOTD banner, SSHv2 keys/domain, line hardening (Console & VTY 0 15), VLAN creation, Access Port range assignment with spanning-tree portfast, 802.1Q trunk setup on Gi0/1, SVI on interface Vlan99, and ip default-gateway.

Router (Coffee-Shop RTR): System hardening, pre-staged ISP WAN interface, Router-on-a-Stick sub-interfaces (Gi0/1.10, .20, .30, .99) with 802.1Q encapsulation, DHCP pools with exclusion ranges, and the Extended Access Control List (GUEST) applied inbound.

End-Device & Wireless Setup: Static IP configurations for both printers (Office and POS receipt printers), along with wireless AP settings (SSID Coffee-SHop-Guest, WPA2-PSK passphrase, Channel 6) and screenshots of client associations.

Configuration Verification (Running-Configs): Grid tables displaying all the do sh run screenshots captured from the switch and router to serve as proof of work.

Testing & Connectivity Validation: Screenshot proofs showing ICMP ping tests for:

Office PC to local printer and gateway.

POS Terminal to receipt printer and gateway.

Guest laptop successfully pinging within its subnet while being explicitly blocked from reaching the POS/Office subnets via the ACL.

Key Engineering Takeaways: Technical insights highlighting ACL DHCP bootstrapping rules, port reservation strategies, and out-of-band/SVI management best practices.

let the table of content be i toggle make changes in readme.md

                 +-----------------------+
                 |       Cloud ISP       |
                 +-----------+-----------+
                             | (Gig0/0 - WAN)
                 +-----------+-----------+
                 |    Coffee-Shop RTR    | (Cisco 2911)
                 |   192.168.x.1 Gateways|
                 +-----------+-----------+
                             | (Gig0/1 - 802.1Q Trunk)
                 +-----------+-----------+
                 |       Coffee-SW       | (Cisco 2960)
                 |   192.168.99.2 (SVI)  |
                 +---+-------+-------+---+
                     |       |       |
     +---------------+       |       +---------------+
     | (Fa0/1 - Fa0/5)       | (Fa0/6 - Fa0/10)      | (Fa0/11)
     v                       v                       v
+------------------+    +------------------+    +------------------+
|  VLAN 10: Office |    |   VLAN 20: POS   |    | VLAN 30: Guest   |
| - Manager PC     |    | - POS Terminal   |    | - AccessPoint-PT |
| - Office Printer |    | - Receipt Printer|    | - Guest Laptops  |
+------------------+    +------------------+    +------------------+


### Topology Diagram (Packet Tracer)
![Network Topology Diagram](images/topology_diagram.jpeg)

---

## 📊 Addressing Scheme & VLAN Segmentation

| VLAN ID | VLAN Name | Subnet | Gateway | Static Pool (Reserved) | DHCP Pool Range | Assigned Ports |
|:---|:---|:---|:---|:---|:---|:---|
| **10** | `managment_office` | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.1` - `.20` | `192.168.10.21` - `.254` | `Fa0/1` - `Fa0/5` |
| **20** | `POS` | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.1` - `.20` | `192.168.20.21` - `.254` | `Fa0/6` - `Fa0/10` |
| **30** | `GUEST_WIFI` | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.1` - `.20` | `192.168.30.21` - `.254` | `Fa0/11` |
| **99** | `Network_Management`| `192.168.99.0/24` | `192.168.99.1` | `192.168.99.1` - `.20` | N/A (Static SVI: `.2`) | Internal / Trunk |

---

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
2. Router Configuration (Coffee-Shop RTR)
Cisco CLI
! --- Hostname & System Security ---
Router> enable
Router# configure terminal
Router(config)# hostname RTR
RTR(config)# no ip domain-lookup
RTR(config)# service password-encryption
RTR(config)# enable secret Cisco123!
RTR(config)# banner motd ^C Unautorized access banned ^C

! --- Line Hardening ---
RTR(config)# line con 0
RTR(config-line)# password ConsolePass123!
RTR(config-line)# logging synchronous
RTR(config-line)# exit
RTR(config)# line vty 0 4
RTR(config-line)# login
RTR(config-line)# exit

! --- WAN Interface to ISP (Pre-staged) ---
RTR(config)# interface gig0/0
RTR(config-if)# description ## to_ISP##
RTR(config-if)# no shutdown
RTR(config-if)# exit

! --- Router-on-a-Stick Sub-interfaces ---
RTR(config)# interface gig0/1
RTR(config-if)# description ## to_SW1 ##
RTR(config-if)# no ip address
RTR(config-if)# no shutdown
RTR(config-if)# exit

RTR(config)# interface gig0/1.10
RTR(config-subif)# description ##Management_Office##
RTR(config-subif)# encapsulation dot1Q 10
RTR(config-subif)# ip address 192.168.10.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.20
RTR(config-subif)# description ## POS_Gateway##
RTR(config-subif)# encapsulation dot1Q 20
RTR(config-subif)# ip address 192.168.20.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.30
RTR(config-subif)# description ##GUEST_WIFI##
RTR(config-subif)# encapsulation dot1Q 30
RTR(config-subif)# ip address 192.168.30.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.99
RTR(config-subif)# description ##Network_Managment_gateway##
RTR(config-subif)# encapsulation dot1Q 99
RTR(config-subif)# ip address 192.168.99.1 255.255.255.0
RTR(config-subif)# exit

! --- DHCP Server Pools Configuration ---
RTR(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.20
RTR(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.20
RTR(config)# ip dhcp excluded-address 192.168.30.1 192.168.30.20

RTR(config)# ip dhcp pool managment_office
RTR(dhcp-config)# network 192.168.10.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.10.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool POS
RTR(dhcp-config)# network 192.168.20.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.20.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool GUEST_WIFI
RTR(dhcp-config)# network 192.168.30.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.30.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

! --- Extended Access Control List (Guest Network Isolation) ---
RTR(config)# ip access-list extended GUEST
RTR(config-ext-nacl)# permit udp any eq bootpc any eq bootps
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
RTR(config-ext-nacl)# permit ip 192.168.30.0 0.0.0.255 any
RTR(config-ext-nacl)# exit

! --- Apply ACL Inbound on Guest Sub-interface ---
RTR(config)# interface gig0/1.30
RTR(config-subif)# ip access-group GUEST in
RTR(config-subif)# exit
RTR(config)# do copy run start
🖨 End-Device & Wireless Setup
Static IP Device Assignments
1. Office Printer (VLAN 10)
IP Address: 192.168.10.10

Subnet Mask: 255.255.255.0

Default Gateway: 192.168.10.1

DNS Server: 8.8.8.8

Office Printer Global Settings	Office Printer FastEthernet0 Config
2. Receipt Printer (VLAN 20)
IP Address: 192.168.20.10

Subnet Mask: 255.255.255.0

Default Gateway: 192.168.20.1

DNS Server: 8.8.8.8

Receipt Printer Global Settings	Receipt Printer FastEthernet0 Config
Wireless Access Point & Client Associations
SSID: Coffee-SHop-Guest

Channel: Channel 6 (2.4 GHz)

Security Protocol: WPA2-PSK (AES Encryption)

Pre-shared Key (PSK): CofShop!!!

Access Point Port 1 Setup	Client WPA2-Personal Authentication
Guest Client Association Status	Guest Laptop 0 DHCP Leasing (192.168.30.21)
📜 Configuration Verification (Running-Configs)
Router Running-Config Proofs (RTR# do sh run)
Router Services & DHCP Exclusions	Router DHCP Pools & Hardware UDI
Router Sub-Interfaces (Gi0/1.10, Gi0/1.20, Gi0/1.30, Gi0/1.99)	Router Extended ACL (GUEST) & Line Configs
Switch Running-Config Proofs (Coffee-SW# sh run)
Switch Hostname, Secrets & SSH Config	Switch Access Ports (Fa0/1 – Fa0/5 in VLAN 10)
Switch Access Ports (Fa0/6 – Fa0/11 in VLAN 20 & 30)	Switch Trunk Uplink (Gi0/1) & Management SVI (Vlan99)	Switch Line VTY & Console Security
🧪 Testing & Connectivity Validation
1. Inter-VLAN Routing & Office LAN Validation
From Manager PC (192.168.10.x), traffic successfully reaches the local printer (192.168.10.10) and default gateway (192.168.10.1) with <1ms round-trip latency.

2. POS Terminal Connectivity
The POS Terminal verifies instant communication with its default gateway (192.168.20.1) and receipt printer (192.168.20.10) with 0% packet loss.

3. Guest ACL Isolation & Quarantine Test
From Laptop0 (192.168.30.21):

Ping to POS Subnet (192.168.20.1): Destination host unreachable (Blocked by ACL GUEST).

Ping within Guest Subnet (192.168.30.22): 4/4 packets successful (Intra-subnet communication working).

💡 Key Engineering Takeaways
DHCP Bootstrapping inside ACLs: When applying an inbound ACL on a DHCP client subnet, standard deny rules will drop broadcast DHCP requests unless permit udp any eq bootpc any eq bootps is placed at top-of-stack.

Deterministic Port Mapping: Grouping switch ports sequentially (1-5 Office, 6-10 POS, 11 AP) reduces administrative overhead and makes on-site physical troubleshooting straightforward.

Defense-in-Depth for Management: Disabling telnet across VTY lines via transport input ssh and routing management through an isolated SVI (VLAN 99) protects infrastructure from rogue LAN discovery.
"""

with open("README.md", "w") as f:
f.write(readme_content)

print("Updated README.md with toggle Table of Contents.")


```text?code_stdout&code_event_index=1
Updated README.md with toggle Table of Contents.

I have updated the README.md file so that the Table of Contents uses an interactive HTML <details> and <summary> dropdown toggle.

The updated file is available for download here:

MD icon
README
 MD 
Open
How the Table of Contents Render in GitHub:
HTML
<details>
<summary>📌 <b>Table of Contents (Click to Expand / Collapse)</b></summary>

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

</details>
This keeps the top of your GitHub repository clean, compact, and scannable while allowing viewers to toggle the full navigation outline on demand.

 I'm unable to access the topology diagram. Could you provide me the code so that I can just directly copy and paste in my readme, and the diagram will automatically appear?  And I want to make a toggle button for every table of contents, as you have shown. Executor SummaryToggle Button

Addressing

Toggle Button

Switch Configuration

Toggle Button

Show Run Images





That is another Toggle Button as a proof. I want this kind of setting so that my readme looks very professional, very clean, and it will not look very lengthy.  I want to create one README itself. I want to create a separate folder for router configuration and switch configuration. Inside that, everything should be present in one README. I want to set everything right based on the toggle buttons. Just by switching on the toggle button, every content should appear, and every screenshot should appear according to what we have decided. You can refer to my screenshot as shown. There are links. Just by clicking on that particular link, my page is going to redirect to that particular toggle button so that I can turn on the toggle button and receive all the content which is an example shown in a screenshot? 

Topology Snapshot (Packet Tracer)
📊 Section 3: IP Addressing & VLAN Segmentation Scheme
This section outlines the IP allocation matrix, subnet masks, default gateways, and physical switch port reservations.

VLAN ID	VLAN Name	Subnet	Gateway	Static Pool (Reserved)	DHCP Pool Range	Assigned Ports
10	managment_office	192.168.10.0/24	192.168.10.1	192.168.10.1 - .20	192.168.10.21 - .254	Fa0/1 - Fa0/5
20	POS	192.168.20.0/24	192.168.20.1	192.168.20.1 - .20	192.168.20.21 - .254	Fa0/6 - Fa0/10
30	GUEST_WIFI	192.168.30.0/24	192.168.30.1	192.168.30.1 - .20	192.168.30.21 - .254	Fa0/11
99	Network_Management	192.168.99.0/24	192.168.99.1	192.168.99.1 - .20	N/A (Static SVI: .2)	Internal / Trunk
🛠 Section 4: Switch Layer 2 & Hardening Configuration
This section contains all CLI commands deployed on Coffee-SW (Cisco Catalyst 2960) including VLAN segmentation, Spanning-Tree PortFast, 802.1Q trunking, SVI configuration, and SSHv2 line hardening.

Cisco CLI
! ==========================================
! SWITCH CONFIGURATION: Coffee-SW (Cisco 2960)
! ==========================================

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
🌐 Section 5: Router-on-a-Stick, DHCP & ACL Security Configuration
This section provides the full configuration script for Coffee-Shop RTR (Cisco 2911) including 802.1Q sub-interfaces, dynamic DHCP scopes, and inbound Extended ACLs.

Cisco CLI
! ==========================================
! ROUTER CONFIGURATION: Coffee-Shop RTR (Cisco 2911)
! ==========================================

! --- Hostname & System Security ---
Router> enable
Router# configure terminal
Router(config)# hostname RTR
RTR(config)# no ip domain-lookup
RTR(config)# service password-encryption
RTR(config)# enable secret Cisco123!
RTR(config)# banner motd ^C Unautorized access banned ^C

! --- Line Hardening ---
RTR(config)# line con 0
RTR(config-line)# password ConsolePass123!
RTR(config-line)# logging synchronous
RTR(config-line)# exit
RTR(config)# line vty 0 4
RTR(config-line)# login
RTR(config-line)# exit

! --- WAN Interface to ISP (Pre-staged) ---
RTR(config)# interface gig0/0
RTR(config-if)# description ## to_ISP##
RTR(config-if)# no shutdown
RTR(config-if)# exit

! --- Router-on-a-Stick Sub-interfaces ---
RTR(config)# interface gig0/1
RTR(config-if)# description ## to_SW1 ##
RTR(config-if)# no ip address
RTR(config-if)# no shutdown
RTR(config-if)# exit

RTR(config)# interface gig0/1.10
RTR(config-subif)# description ##Management_Office##
RTR(config-subif)# encapsulation dot1Q 10
RTR(config-subif)# ip address 192.168.10.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.20
RTR(config-subif)# description ## POS_Gateway##
RTR(config-subif)# encapsulation dot1Q 20
RTR(config-subif)# ip address 192.168.20.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.30
RTR(config-subif)# description ##GUEST_WIFI##
RTR(config-subif)# encapsulation dot1Q 30
RTR(config-subif)# ip address 192.168.30.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.99
RTR(config-subif)# description ##Network_Managment_gateway##
RTR(config-subif)# encapsulation dot1Q 99
RTR(config-subif)# ip address 192.168.99.1 255.255.255.0
RTR(config-subif)# exit

! --- DHCP Server Pools Configuration ---
RTR(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.20
RTR(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.20
RTR(config)# ip dhcp excluded-address 192.168.30.1 192.168.30.20

RTR(config)# ip dhcp pool managment_office
RTR(dhcp-config)# network 192.168.10.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.10.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool POS
RTR(dhcp-config)# network 192.168.20.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.20.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool GUEST_WIFI
RTR(dhcp-config)# network 192.168.30.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.30.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

! --- Extended Access Control List (Guest Network Isolation) ---
RTR(config)# ip access-list extended GUEST
RTR(config-ext-nacl)# permit udp any eq bootpc any eq bootps
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
RTR(config-ext-nacl)# permit ip 192.168.30.0 0.0.0.255 any
RTR(config-ext-nacl)# exit

! --- Apply ACL Inbound on Guest Sub-interface ---
RTR(config)# interface gig0/1.30
RTR(config-subif)# ip access-group GUEST in
RTR(config-subif)# exit
RTR(config)# do copy run start
🖨 Section 6: End-Device & Wireless Infrastructure Setup
This section documents static printer provisioning, access point wireless parameters, and client WPA2 associations.

1. Static IP Device Assignments
Office Printer (VLAN 10)
IP Address: 192.168.10.10

Subnet Mask: 255.255.255.0

Default Gateway: 192.168.10.1

DNS Server: 8.8.8.8

Office Printer Global Settings	Office Printer FastEthernet0 Config
Receipt Printer (VLAN 20)
IP Address: 192.168.20.10

Subnet Mask: 255.255.255.0

Default Gateway: 192.168.20.1

DNS Server: 8.8.8.8

Receipt Printer Global Settings	Receipt Printer FastEthernet0 Config
2. Wireless Access Point & Client Associations
SSID: Coffee-SHop-Guest

Channel: Channel 6 (2.4 GHz)

Security Protocol: WPA2-PSK (AES Encryption)

Pre-shared Key (PSK): CofShop!!!

Access Point Port 1 Setup	Client WPA2-Personal Authentication
Guest Client Association Status	Guest Laptop 0 DHCP Lease (192.168.30.21)
📜 Section 7: Running-Configuration Proofs (show run)
This section provides visual audit proof of all running-configurations across the router and switch.

Router Running-Config Proofs (RTR# do sh run)
Router Services & DHCP Exclusions	Router DHCP Pools & Hardware UDI
Router Sub-Interfaces (Gi0/1.10, .20, .30, .99)	Router Extended ACL (GUEST) & Line Configs
Switch Running-Config Proofs (Coffee-SW# sh run)
Switch Hostname, Secrets & SSH Config	Switch Access Ports (Fa0/1 – Fa0/5 in VLAN 10)
Switch Access Ports (Fa0/6 – Fa0/11 in VLAN 20 & 30)	Switch Trunk Uplink (Gi0/1) & Management SVI (Vlan99)	Switch Line VTY & Console Security
🧪 Section 8: Connectivity Validation & Security Isolation Tests
This section contains verification ping tests demonstrating successful Intra-VLAN communication, gateway reachability, and zero-trust ACL blocking.

1. Inter-VLAN Routing & Office LAN Validation
From Manager PC (192.168.10.x), traffic successfully reaches the local printer (192.168.10.10) and default gateway (192.168.10.1) with <1ms round-trip latency.

2. POS Terminal Connectivity
The POS Terminal verifies instant communication with its default gateway (192.168.20.1) and receipt printer (192.168.20.10) with 0% packet loss.

3. Guest ACL Isolation & Quarantine Test
From Laptop0 (192.168.30.21):

Ping to POS Subnet (192.168.20.1): Destination host unreachable (Explicitly blocked by ACL GUEST).

Ping within Guest Subnet (192.168.30.22): 4/4 packets successful (Intra-subnet communication working).

💡 Section 9: Key Engineering Takeaways
This section synthesizes the core technical insights gained throughout this implementation.

DHCP Bootstrapping inside ACLs: When applying an inbound ACL on a DHCP client subnet, standard deny rules will drop broadcast DHCP requests unless permit udp any eq bootpc any eq bootps is placed at top-of-stack.

Deterministic Port Mapping: Grouping switch ports sequentially (1-5 Office, 6-10 POS, 11 AP) reduces administrative overhead and makes on-site physical troubleshooting straightforward.

Defense-in-Depth for Management: Disabling telnet across VTY lines via transport input ssh and routing management through an isolated SVI (VLAN 99) protects infrastructure from rogue LAN discovery.

with open("README.md", "w") as f:
f.write(readme_structured)

print("Modular toggle README.md created.")


```text?code_stdout&code_event_index=1
Modular toggle README.md created.

Your updated README.md file matching the modular layout in your reference screenshot has been generated:
  
MD
MD icon
README
 MD 
Open

Key Additions & Structural Enhancements
Auto-Rendering Native Topology Diagram (Mermaid.js):
You do not need to rely solely on image hosting for the diagram. GitHub natively parses and renders the Mermaid diagram code below directly in dark/light mode:

Code snippet
flowchart TD
    ISP["☁️ Cloud ISP"] ---|Gig0/0 WAN| RTR["🔲 Cisco 2911 Router<br><b>Coffee-Shop RTR</b><br>192.168.x.1 Gateways"]
    
    RTR ---|Gig0/1 (802.1Q Trunk)<br>VLANs 10, 20, 30, 99| SW["🔲 Cisco 2960 Switch<br><b>Coffee-SW</b><br>192.168.99.2 (SVI)"]
    
    subgraph VLAN10 ["🏢 VLAN 10: Management Office (192.168.10.0/24)"]
        SW ---|Fa0/1| PC_MGR["💻 Manager PC<br>192.168.10.21 (DHCP)"]
        SW ---|Fa0/2| PRN_OFF["🖨️ Office Printer<br>192.168.10.10 (Static)"]
    end

    subgraph VLAN20 ["💳 VLAN 20: Point of Sale (192.168.20.0/24)"]
        SW ---|Fa0/6| POS_TERM["💻 POS Terminal<br>192.168.20.21 (DHCP)"]
        SW ---|Fa0/7| PRN_RCPT["🖨️ Receipt Printer<br>192.168.20.10 (Static)"]
    end

    subgraph VLAN30 ["☕ VLAN 30: Guest Wi-Fi (192.168.30.0/24)"]
        SW ---|Fa0/11| AP["📡 AccessPoint-PT<br>SSID: Coffee-SHop-Guest"]
        AP -.-|802.11n Wireless| LP0["💻 Guest Laptop 0<br>192.168.30.21 (DHCP)"]
        AP -.-|802.11n Wireless| LP1["💻 Guest Laptop 1<br>192.168.30.22 (DHCP)"]
    end

    classDef rtr fill:#003366,stroke:#fff,stroke-width:2px,color:#fff;
    classDef sw fill:#004080,stroke:#fff,stroke-width:2px,color:#fff;
    classDef dev fill:#1a1a1a,stroke:#33cc33,stroke-width:1px,color:#fff;
    class RTR rtr;
    class SW sw;
    class PC_MGR,PRN_OFF,POS_TERM,PRN_RCPT,AP,LP0,LP1 dev;
Anchor Jump Links:
Clicking any of the top section links (e.g., View Section 4: Switch Layer 2 Configuration) scrolls the page directly to that specific module header.

Individual Modular Toggle Buttons (<details> & <summary>):
Every single section is wrapped in its own expandable container (▶ Click to view...), keeping your document clean, organized, and compact on first load.




Gemini is AI and can make mistakes.

Analysing
# Secure Small-Business Enterprise Network (Coffee Shop Branch)

A production-ready small-business enterprise local area network (LAN) designed and validated in **Cisco Packet Tracer**. This project simulates an end-to-end network implementation for a multi-zone commercial environment ("Mike's Coffee Shop"), demonstrating network segmentation, Inter-VLAN routing (Router-on-a-Stick), dynamic addressing via Cisco IOS DHCP pools, hardened device administration, and security boundaries using extended Access Control Lists (ACLs).

---

## 📁 Network Implementation Modules

* 🏢 [View Section 1: Executive Summary & Design Objectives](#section-1-executive-summary--design-objectives) — *Business use case, physical layout, and zero-trust zone isolation.*
* 🗺 [View Section 2: Network Topology Architecture](#section-2-network-topology-architecture) — *Interactive Mermaid render & packet flow breakdown.*
* 📊 [View Section 3: IP Addressing & VLAN Segmentation Scheme](#section-3-ip-addressing--vlan-segmentation-scheme) — *Subnet planning, gateways, static exclusions, and port mapping.*
* 🛠 [View Section 4: Switch Layer 2 & Hardening Configuration](#section-4-switch-layer-2--hardening-configuration) — *VLAN database, PortFast, 802.1Q trunk, SVI management, and SSHv2.*
* 🌐 [View Section 5: Router-on-a-Stick, DHCP & ACL Security Configuration](#section-5-router-on-a-stick-dhcp--acl-security-configuration) — *Sub-interfaces, DHCP scopes, and inbound Extended ACLs.*
* 🖨 [View Section 6: End-Device & Wireless Infrastructure Setup](#section-6-end-device--wireless-infrastructure-setup) — *Printers static provisioning, AP WPA2-PSK, and wireless client association.*
* 📜 [View Section 7: Running-Configuration Proofs (show run)](#section-7-running-configuration-proofs-show-run) — *CLI audit proofs captured across router and switch.*
* 🧪 [View Section 8: Connectivity Validation & Security Isolation Tests](#section-8-connectivity-validation--security-isolation-tests) — *Ping tests, gateway reachability, and ACL quarantine verification.*
* 💡 [View Section 9: Key Engineering Takeaways](#section-9-key-engineering-takeaways) — *Best practices regarding DHCP bootstrapping, port safety, and management planes.*

---

<a name="section-1-executive-summary--design-objectives"></a>
## 🏢 Section 1: Executive Summary & Design Objectives

This section details the business operational requirements, multi-zone isolation needs, and primary engineering objectives for Mike's Coffee Shop.

<details>
<summary>▶ <b>Click to view Executive Summary & Architecture Objectives</b></summary>
<br>

### Business Overview
In a commercial establishment like a modern café or retail branch, business operations require three core capabilities:
1. **Administrative Operations:** Secure workstation access and network printing for store management.
2. **Point-of-Sale (POS) Transactions:** Reliable, low-latency transaction processing and receipt generation isolated from general traffic.
3. **Public Wireless (Guest Wi-Fi):** Frictionless customer internet access that is strictly quarantined from internal business systems.

### Core Engineering Objectives
* **Zero-Trust Zone Isolation:** Separate management, financial/POS, and guest traffic into distinct Layer 2 broadcast domains (VLANs).
* **Router-on-a-Stick (ROAS):** Enable centralized Layer 3 inter-VLAN routing across a single 802.1Q trunk link.
* **Granular Traffic Quarantining (Extended ACL):** Ensure guest clients can reach DNS/DHCP and external WAN routes while completely dropping traffic headed towards internal subnets.
* **Deterministic IP Planning:** Reserve static pools (`.1`–`.20`) for critical default gateways and network printers, while dynamically leasing addresses (`.21`+) to PCs and wireless endpoints.
* **Management Hardening:** Enforce SSHv2 remote administrative access, MD5/Scrypt privileged secrets, MOTD banners, and console line security.

</details>

---

<a name="section-2-network-topology-architecture"></a>
## 🗺 Section 2: Network Topology Architecture

This section renders the physical and logical layout of the Coffee Shop deployment directly inside GitHub using Native Mermaid markdown.

<details>
<summary>▶ <b>Click to view Network Topology Diagram & Packet Flow</b></summary>
<br>

### Native Interactive Topology Diagram

```mermaid
flowchart TD
    ISP["☁️ Cloud ISP"] ---|Gig0/0 WAN| RTR["🔲 Cisco 2911 Router<br><b>Coffee-Shop RTR</b><br>192.168.x.1 Gateways"]
    
    RTR ---|Gig0/1 (802.1Q Trunk)<br>VLANs 10, 20, 30, 99| SW["🔲 Cisco 2960 Switch<br><b>Coffee-SW</b><br>192.168.99.2 (SVI)"]
    
    subgraph VLAN10 ["🏢 VLAN 10: Management Office (192.168.10.0/24)"]
        SW ---|Fa0/1| PC_MGR["💻 Manager PC<br>192.168.10.21 (DHCP)"]
        SW ---|Fa0/2| PRN_OFF["🖨️ Office Printer<br>192.168.10.10 (Static)"]
    end

    subgraph VLAN20 ["💳 VLAN 20: Point of Sale (192.168.20.0/24)"]
        SW ---|Fa0/6| POS_TERM["💻 POS Terminal<br>192.168.20.21 (DHCP)"]
        SW ---|Fa0/7| PRN_RCPT["🖨️ Receipt Printer<br>192.168.20.10 (Static)"]
    end

    subgraph VLAN30 ["☕ VLAN 30: Guest Wi-Fi (192.168.30.0/24)"]
        SW ---|Fa0/11| AP["📡 AccessPoint-PT<br>SSID: Coffee-SHop-Guest"]
        AP -.-|802.11n Wireless| LP0["💻 Guest Laptop 0<br>192.168.30.21 (DHCP)"]
        AP -.-|802.11n Wireless| LP1["💻 Guest Laptop 1<br>192.168.30.22 (DHCP)"]
    end

    classDef rtr fill:#003366,stroke:#fff,stroke-width:2px,color:#fff;
    classDef sw fill:#004080,stroke:#fff,stroke-width:2px,color:#fff;
    classDef dev fill:#1a1a1a,stroke:#33cc33,stroke-width:1px,color:#fff;
    class RTR rtr;
    class SW sw;
    class PC_MGR,PRN_OFF,POS_TERM,PRN_RCPT,AP,LP0,LP1 dev;
```

### Topology Snapshot (Packet Tracer)
![Network Topology Diagram](images/topology_diagram.jpeg)

</details>

---

<a name="section-3-ip-addressing--vlan-segmentation-scheme"></a>
## 📊 Section 3: IP Addressing & VLAN Segmentation Scheme

This section outlines the IP allocation matrix, subnet masks, default gateways, and physical switch port reservations.

<details>
<summary>▶ <b>Click to view Detailed IP Addressing & Subnet Matrix</b></summary>
<br>

| VLAN ID | VLAN Name | Subnet | Gateway | Static Pool (Reserved) | DHCP Pool Range | Assigned Ports |
|:---|:---|:---|:---|:---|:---|:---|
| **10** | `managment_office` | `192.168.10.0/24` | `192.168.10.1` | `192.168.10.1` - `.20` | `192.168.10.21` - `.254` | `Fa0/1` - `Fa0/5` |
| **20** | `POS` | `192.168.20.0/24` | `192.168.20.1` | `192.168.20.1` - `.20` | `192.168.20.21` - `.254` | `Fa0/6` - `Fa0/10` |
| **30** | `GUEST_WIFI` | `192.168.30.0/24` | `192.168.30.1` | `192.168.30.1` - `.20` | `192.168.30.21` - `.254` | `Fa0/11` |
| **99** | `Network_Management`| `192.168.99.0/24` | `192.168.99.1` | `192.168.99.1` - `.20` | N/A (Static SVI: `.2`) | Internal / Trunk |

</details>

---

<a name="section-4-switch-layer-2--hardening-configuration"></a>
## 🛠 Section 4: Switch Layer 2 & Hardening Configuration

This section contains all CLI commands deployed on `Coffee-SW` (Cisco Catalyst 2960) including VLAN segmentation, Spanning-Tree PortFast, 802.1Q trunking, SVI configuration, and SSHv2 line hardening.

<details>
<summary>▶ <b>Click to view Switch CLI Configuration Script</b></summary>
<br>

```cisco
! ==========================================
! SWITCH CONFIGURATION: Coffee-SW (Cisco 2960)
! ==========================================

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

</details>

---

<a name="section-5-router-on-a-stick-dhcp--acl-security-configuration"></a>
## 🌐 Section 5: Router-on-a-Stick, DHCP & ACL Security Configuration

This section provides the full configuration script for `Coffee-Shop RTR` (Cisco 2911) including 802.1Q sub-interfaces, dynamic DHCP scopes, and inbound Extended ACLs.

<details>
<summary>▶ <b>Click to view Router CLI Configuration Script</b></summary>
<br>

```cisco
! ==========================================
! ROUTER CONFIGURATION: Coffee-Shop RTR (Cisco 2911)
! ==========================================

! --- Hostname & System Security ---
Router> enable
Router# configure terminal
Router(config)# hostname RTR
RTR(config)# no ip domain-lookup
RTR(config)# service password-encryption
RTR(config)# enable secret Cisco123!
RTR(config)# banner motd ^C Unautorized access banned ^C

! --- Line Hardening ---
RTR(config)# line con 0
RTR(config-line)# password ConsolePass123!
RTR(config-line)# logging synchronous
RTR(config-line)# exit
RTR(config)# line vty 0 4
RTR(config-line)# login
RTR(config-line)# exit

! --- WAN Interface to ISP (Pre-staged) ---
RTR(config)# interface gig0/0
RTR(config-if)# description ## to_ISP##
RTR(config-if)# no shutdown
RTR(config-if)# exit

! --- Router-on-a-Stick Sub-interfaces ---
RTR(config)# interface gig0/1
RTR(config-if)# description ## to_SW1 ##
RTR(config-if)# no ip address
RTR(config-if)# no shutdown
RTR(config-if)# exit

RTR(config)# interface gig0/1.10
RTR(config-subif)# description ##Management_Office##
RTR(config-subif)# encapsulation dot1Q 10
RTR(config-subif)# ip address 192.168.10.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.20
RTR(config-subif)# description ## POS_Gateway##
RTR(config-subif)# encapsulation dot1Q 20
RTR(config-subif)# ip address 192.168.20.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.30
RTR(config-subif)# description ##GUEST_WIFI##
RTR(config-subif)# encapsulation dot1Q 30
RTR(config-subif)# ip address 192.168.30.1 255.255.255.0
RTR(config-subif)# exit

RTR(config)# interface gig0/1.99
RTR(config-subif)# description ##Network_Managment_gateway##
RTR(config-subif)# encapsulation dot1Q 99
RTR(config-subif)# ip address 192.168.99.1 255.255.255.0
RTR(config-subif)# exit

! --- DHCP Server Pools Configuration ---
RTR(config)# ip dhcp excluded-address 192.168.10.1 192.168.10.20
RTR(config)# ip dhcp excluded-address 192.168.20.1 192.168.20.20
RTR(config)# ip dhcp excluded-address 192.168.30.1 192.168.30.20

RTR(config)# ip dhcp pool managment_office
RTR(dhcp-config)# network 192.168.10.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.10.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool POS
RTR(dhcp-config)# network 192.168.20.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.20.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

RTR(config)# ip dhcp pool GUEST_WIFI
RTR(dhcp-config)# network 192.168.30.0 255.255.255.0
RTR(dhcp-config)# default-router 192.168.30.1
RTR(dhcp-config)# dns-server 8.8.8.8
RTR(dhcp-config)# exit

! --- Extended Access Control List (Guest Network Isolation) ---
RTR(config)# ip access-list extended GUEST
RTR(config-ext-nacl)# permit udp any eq bootpc any eq bootps
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
RTR(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.99.0 0.0.0.255
RTR(config-ext-nacl)# permit ip 192.168.30.0 0.0.0.255 any
RTR(config-ext-nacl)# exit

! --- Apply ACL Inbound on Guest Sub-interface ---
RTR(config)# interface gig0/1.30
RTR(config-subif)# ip access-group GUEST in
RTR(config-subif)# exit
RTR(config)# do copy run start
```

</details>

---

<a name="section-6-end-device--wireless-infrastructure-setup"></a>
## 🖨 Section 6: End-Device & Wireless Infrastructure Setup

This section documents static printer provisioning, access point wireless parameters, and client WPA2 associations.

<details>
<summary>▶ <b>Click to view End-Device & Wireless Setup Screenshots</b></summary>
<br>

### 1. Static IP Device Assignments

#### Office Printer (VLAN 10)
* **IP Address:** `192.168.10.10`
* **Subnet Mask:** `255.255.255.0`
* **Default Gateway:** `192.168.10.1`
* **DNS Server:** `8.8.8.8`

| Office Printer Global Settings | Office Printer FastEthernet0 Config |
|:---:|:---:|
| ![Office Printer Global](images/printer_office_global.jpeg) | ![Office Printer IP](images/printer_office_ip.jpeg) |

#### Receipt Printer (VLAN 20)
* **IP Address:** `192.168.20.10`
* **Subnet Mask:** `255.255.255.0`
* **Default Gateway:** `192.168.20.1`
* **DNS Server:** `8.8.8.8`

| Receipt Printer Global Settings | Receipt Printer FastEthernet0 Config |
|:---:|:---:|
| ![Receipt Printer Global](images/printer_receipt_global.jpeg) | ![Receipt Printer IP](images/printer_receipt_ip.jpeg) |

---

### 2. Wireless Access Point & Client Associations

* **SSID:** `Coffee-SHop-Guest`
* **Channel:** Channel 6 (2.4 GHz)
* **Security Protocol:** WPA2-PSK (AES Encryption)
* **Pre-shared Key (PSK):** `CofShop!!!`

| Access Point Port 1 Setup | Client WPA2-Personal Authentication |
|:---:|:---:|
| ![AP Port Config](images/ap_port_config.jpeg) | ![Client Association](images/laptop_wpa2_connect.jpeg) |

| Guest Client Association Status | Guest Laptop 0 DHCP Lease (`192.168.30.21`) |
|:---:|:---:|
| ![Wireless Monitor](images/laptop_wireless_monitor.jpeg) | ![Laptop0 DHCP Lease](images/laptop0_dhcp.jpeg) |

</details>

---

<a name="section-7-running-configuration-proofs-show-run"></a>
## 📜 Section 7: Running-Configuration Proofs (`show run`)

This section provides visual audit proof of all running-configurations across the router and switch.

<details>
<summary>▶ <b>Click to view Router & Switch running-config proofs</b></summary>
<br>

### Router Running-Config Proofs (`RTR# do sh run`)

| Router Services & DHCP Exclusions | Router DHCP Pools & Hardware UDI |
|:---:|:---:|
| ![RTR Show Run 1](images/rtr_sh_run_1.jpeg) | ![RTR Show Run 2](images/rtr_sh_run_2.jpeg) |

| Router Sub-Interfaces (`Gi0/1.10`, `.20`, `.30`, `.99`) | Router Extended ACL (`GUEST`) & Line Configs |
|:---:|:---:|
| ![RTR Show Run 3](images/rtr_sh_run_3.jpeg) | ![RTR Show Run 4](images/rtr_sh_run_4.jpeg) |

---

### Switch Running-Config Proofs (`Coffee-SW# sh run`)

| Switch Hostname, Secrets & SSH Config | Switch Access Ports (`Fa0/1` – `Fa0/5` in VLAN 10) |
|:---:|:---:|
| ![SW Show Run 1](images/sw_sh_run_1.jpeg) | ![SW Show Run 2](images/sw_sh_run_2.jpeg) |

| Switch Access Ports (`Fa0/6` – `Fa0/11` in VLAN 20 & 30) | Switch Trunk Uplink (`Gi0/1`) & Management SVI (`Vlan99`) | Switch Line VTY & Console Security |
|:---:|:---:|:---:|
| ![SW Show Run 3](images/sw_sh_run_3.jpeg) | ![SW Show Run 4](images/sw_sh_run_4.jpeg) | ![SW Show Run 5](images/sw_sh_run_5.jpeg) |

</details>

---

<a name="section-8-connectivity-validation--security-isolation-tests"></a>
## 🧪 Section 8: Connectivity Validation & Security Isolation Tests

This section contains verification ping tests demonstrating successful Intra-VLAN communication, gateway reachability, and zero-trust ACL blocking.

<details>
<summary>▶ <b>Click to view Connectivity Verification & ACL Tests</b></summary>
<br>

### 1. Inter-VLAN Routing & Office LAN Validation
From **Manager PC** (`192.168.10.x`), traffic successfully reaches the local printer (`192.168.10.10`) and default gateway (`192.168.10.1`) with `<1ms` round-trip latency.

![Manager PC Connectivity Test](images/manager_pc_ping_test.jpeg)

---

### 2. POS Terminal Connectivity
The **POS Terminal** verifies instant communication with its default gateway (`192.168.20.1`) and receipt printer (`192.168.20.10`) with 0% packet loss.

![POS Terminal Ping Test](images/pos_terminal_ping_test.jpeg)

---

### 3. Guest ACL Isolation & Quarantine Test
From **Laptop0** (`192.168.30.21`):
* **Ping to POS Subnet (`192.168.20.1`):** `Destination host unreachable` (Explicitly blocked by ACL `GUEST`).
* **Ping within Guest Subnet (`192.168.30.22`):** `4/4 packets successful` (Intra-subnet communication working).

![Guest Isolation Ping Test](images/guest_laptop_ping_acl_test.jpeg)

</details>

---

<a name="section-9-key-engineering-takeaways"></a>
## 💡 Section 9: Key Engineering Takeaways

This section synthesizes the core technical insights gained throughout this implementation.

<details>
<summary>▶ <b>Click to view Key Engineering Insights</b></summary>
<br>

1. **DHCP Bootstrapping inside ACLs:** When applying an inbound ACL on a DHCP client subnet, standard `deny` rules will drop broadcast DHCP requests unless `permit udp any eq bootpc any eq bootps` is placed at top-of-stack.
2. **Deterministic Port Mapping:** Grouping switch ports sequentially (`1-5` Office, `6-10` POS, `11` AP) reduces administrative overhead and makes on-site physical troubleshooting straightforward.
3. **Defense-in-Depth for Management:** Disabling telnet across VTY lines via `transport input ssh` and routing management through an isolated SVI (`VLAN 99`) protects infrastructure from rogue LAN discovery.

</details>
