# Cisco-Packet-Tracer-Project-ARP-Spoofing-DHCP-Starvation-Simulation
Cisco Packet Tracer Lab: ARP Spoofing & DHCP Starvation Simulation
Tool: Cisco Packet Tracer
Topic: Network Attack Simulation & Defense
Difficulty: Intermediate

 Overview
This lab demonstrates two common Layer 2 network attacks — DHCP Starvation and ARP Spoofing — using Cisco Packet Tracer. After simulating both attacks, I applied industry-standard defenses (DHCP Snooping and Dynamic ARP Inspection) to harden the network against them.

🖧 Network Topology
PC0  ──Fa0/1──┐
PC1  ──Fa0/2──┤
PC2  ──Fa0/3──┤── Switch0 (2960-24TT) ──Gi0/1── Router0 (2911)
Server─Fa0/4──┘
<img width="2494" height="1257" alt="Screenshot 2026-05-12 at 9 20 16 PM" src="https://github.com/user-attachments/assets/c1cf7163-1d11-482e-b140-72da47a713a5" />

DeviceRoleIP AddressPC0Legitimate client192.168.1.2 (DHCP)PC1Legitimate client192.168.1.4 (DHCP)PC2Attacker machine192.168.1.5 (DHCP)Server1DHCP Server192.168.1.100 (Static)Router0Default Gateway192.168.1.1 (Static)Switch0Layer 2 SwitchN/A

Configuration
Router (GigabitEthernet0/0)
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 ip helper-address 192.168.1.100
 no shutdown
The ip helper-address command forwards DHCP broadcast requests from clients to the DHCP server, enabling DHCP to function across the network.
DHCP Server
Pool Name:    labpool
Start IP:     192.168.1.10
Subnet Mask:  255.255.255.0
Default GW:   192.168.1.1
Max Users:    50
IP Assignment Verification
All three PCs successfully obtained IP addresses via DHCP:
PC0 → 192.168.1.2
PC1 → 192.168.1.4
PC2 → 192.168.1.5

 Attack Simulation
Attack 1 — DHCP Starvation
How it works:
An attacker floods the DHCP server with requests using spoofed MAC addresses. Each request consumes one IP from the pool. Once the pool is exhausted, legitimate users can't obtain an IP address and are effectively denied network access.
Observed in Packet Tracer:
Using Simulation Mode with DHCP filtering enabled, I observed the full DHCP DORA process (Discover → Offer → Request → Acknowledge) as packets traveled from PC2 through the switch to the server. In a real attack, a script would repeat this process thousands of times per second until the pool is exhausted.
Impact:
Legitimate users receive an APIPA address (169.254.x.x) instead of a valid network IP, losing connectivity entirely.

Attack 2 — ARP Spoofing
How it works:
ARP (Address Resolution Protocol) maps IP addresses to MAC addresses. It has no authentication — any device can send an ARP reply claiming any IP belongs to its MAC. An attacker exploits this by sending fake ARP replies to victims, poisoning their ARP cache and redirecting traffic through the attacker's machine.
Observed in Packet Tracer:
After pinging the router from PC0, I inspected the ARP table:
C:\>arp -a
Internet Address    Physical Address    Type
192.168.1.1         00e0.f71a.6e01      dynamic
PC2's real MAC address was identified:
Physical Address: 0005.5E43.784C
IP Address:       192.168.1.5
Simulated attack scenario:
In a real attack, PC2 would broadcast a fake ARP reply:

"192.168.1.1 is at 0005.5E43.784C"

PC0 would update its ARP cache to associate the router's IP with PC2's MAC, causing all of PC0's traffic to flow through the attacker first — a classic Man-in-the-Middle (MitM) attack.

 Defense Implementation
Defense 1 — DHCP Snooping
DHCP Snooping prevents starvation attacks by rate-limiting DHCP requests on untrusted ports and only allowing DHCP offers from trusted ports (the server uplink).
Configuration applied on Switch0:
ip dhcp snooping
ip dhcp snooping vlan 1

interface FastEthernet0/1
 ip dhcp snooping limit rate 10

interface FastEthernet0/2
 ip dhcp snooping limit rate 10

interface FastEthernet0/3
 ip dhcp snooping limit rate 10

interface FastEthernet0/5
 ip dhcp snooping trust
Verification:
Switch#show ip dhcp snooping

Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs: 1

Interface         Trusted    Rate limit (pps)
-----------       -------    ----------------
FastEthernet0/2   no         10
FastEthernet0/3   no         10
FastEthernet0/1   no         10
FastEthernet0/4   yes        unlimited
Show Image
Any port attempting more than 10 DHCP requests per second will have excess packets dropped — stopping starvation floods cold.

Defense 2 — Dynamic ARP Inspection (DAI)
DAI prevents ARP spoofing by validating ARP packets against the DHCP snooping binding table. If a device claims an IP/MAC combination that doesn't match what was assigned by DHCP, the packet is dropped.
Configuration applied on Switch0:
ip arp inspection vlan 1

interface FastEthernet0/5
 ip arp inspection trust
Verification:
Switch#show ip arp inspection

Vlan    Configuration    Operation
----    -------------    ---------
1       Enabled          Active

Vlan    Forwarded    Dropped    DHCP Drops    ACL Drops
----    ---------    -------    ----------    ---------
1       0            0          0             0
<img width="331" height="127" alt="Screenshot 2026-05-12 at 9 22 13 PM" src="https://github.com/user-attachments/assets/de3d8193-e9d1-4157-b26d-7c9fa0d3e2f9" />

DAI is active on VLAN 1. All ARP packets from untrusted ports are validated — fake ARP replies like those used in spoofing attacks are automatically dropped.

Key Takeaways

ARP has no built-in authentication — any device can poison another's ARP cache without any credentials
DHCP pools are finite — without rate limiting, a single attacker can exhaust all available IPs in seconds
Layer 2 attacks are often overlooked — most firewall rules operate at Layer 3+, leaving switches as a critical but underprotected attack surface
DHCP Snooping and DAI work together — DAI depends on the snooping binding table, so both must be enabled for full protection
Trusted ports must be configured carefully — only uplinks to routers/servers should be trusted; all end-user ports should be untrusted


 Defensive Recommendations
ThreatDefenseCommandDHCP StarvationDHCP Snooping + rate limitingip dhcp snooping limit rate 10ARP Spoofing / MitMDynamic ARP Inspectionip arp inspection vlan 1Rogue DHCP ServerDHCP Snooping trusted portsip dhcp snooping trustMAC FloodingPort Securityswitchport port-security

Disclaimer
This lab was conducted entirely in a controlled, isolated Cisco Packet Tracer environment. All techniques are for educational purposes only. Performing these attacks on real networks without explicit authorization is illegal and unethical.

 References

Cisco DHCP Snooping Configuration Guide
Cisco Dynamic ARP Inspection Guide
OWASP ARP Spoofing
NIST Network Security Guidelines
