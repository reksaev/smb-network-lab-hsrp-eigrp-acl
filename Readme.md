# SMB Network Lab (Cisco Packet Tracer) — HSRP + EIGRP + DHCP Relay + ACL

Small/Medium Business network lab built in Cisco Packet Tracer 8.2.2 (Windows 11).
Scenario: “WOW Decor” office + IT/admin segment. The lab demonstrates VLAN segmentation, gateway redundancy,
dynamic routing, centralized DHCP with relay, basic server services, and ACL-based access control.

## Tech stack / features
- Cisco Packet Tracer 8.2.2 (Windows 11)
- VLANs: 10 (Sales), 20 (Finance), 30 (Server), 99 (Admin)
- Inter-VLAN routing via router-on-a-stick (802.1Q subinterfaces)
- HSRP gateway redundancy for VLAN 10 & 20 (EDGE-01 active by default)
- EIGRP (AS 1) between routers
- DHCP server on R-CORE-01 + DHCP relay (ip helper-address) on EDGE routers
- DNS + Email (SMTP/POP3) + FTP on SRV-CORE-01
- LACP EtherChannel trunk between SW-CORE-01 and SW-ADMIN-01
- Extended ACLs applied inbound on user VLANs (10/20)

## Topology overview
Routers:
- R-EDGE-01, R-EDGE-02 (HSRP pair for user VLANs)
- R-CORE-01 (core side, DHCP + server/admin VLANs)

Switches:
- SW-OFFICE-01 (user access switch)
- SW-CORE-01 (core switch)
- SW-ADMIN-01 (admin switch)

Hosts:
- PC-SALES-01 (VLAN10), PC-FIN-01 (VLAN20), PC-ADMIN-01 (VLAN99)
- SRV-CORE-01 (VLAN30)

Domain:
- wow.com

## Addressing (subnets)
- VLAN10 (Sales): 192.168.10.0/24  | HSRP VIP: 192.168.10.254
- VLAN20 (Finance): 192.168.20.0/24 | HSRP VIP: 192.168.20.254
- VLAN30 (Server): 192.168.30.0/24
- VLAN99 (Admin): 192.168.99.0/24
- Transit links (EIGRP): 10.10.0.0/8 and 20.20.1.0/8 (as configured in the lab)

Server:
- SRV-CORE-01: 192.168.30.4/24, GW 192.168.30.1, DNS 192.168.30.4

Clients via DHCP:
- Default gateway for VLAN10: 192.168.10.254
- Default gateway for VLAN20: 192.168.20.254
- Default gateway for VLAN99: 192.168.99.1
- DNS for all: 192.168.30.4

## Routing
Dynamic routing is done with EIGRP (AS 1).
EIGRP networks per router are included in the configs (see /configs).

## DHCP design
- DHCP server runs on R-CORE-01 for VLAN10/VLAN20/VLAN30/VLAN99 (no excluded addresses in this lab).
- DHCP relay is configured on EDGE subinterfaces for VLAN10 and VLAN20 using ip helper-address toward R-CORE-01.

## ACL policy (what is allowed/blocked)
Goal:
- Sales (VLAN10) and Finance (VLAN20) can communicate with each other and access the Server VLAN (SRV-CORE-01).
- Sales/Finance must NOT access the Admin VLAN (VLAN99).
- Admin VLAN has full access everywhere (no restriction applied on VLAN99 in this lab).

Implementation:
- Extended ACL applied inbound on user VLAN subinterfaces on both EDGE routers:
  - ACL_VLAN10_IN on the VLAN10 subinterface
  - ACL_VLAN20_IN on the VLAN20 subinterface

Notes:
- DHCP (bootpc/bootps) is permitted first so clients can obtain IPs through relay.
- DNS (UDP/TCP 53) is permitted.
- Established TCP and ICMP echo-reply are included to avoid breaking return traffic where relevant.

ACLs used:

ACL_VLAN10_IN:
- permit udp any eq bootpc any eq bootps
- permit udp any eq bootps any eq bootpc
- permit udp any eq domain any
- permit tcp any eq domain any
- permit icmp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 echo-reply
- permit tcp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 established
- deny ip 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255
- permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
- permit ip 192.168.10.0 0.0.0.255 host 192.168.30.4
- permit ip 192.168.10.0 0.0.0.255 any

ACL_VLAN20_IN:
- permit udp any eq bootpc any eq bootps
- permit udp any eq bootps any eq bootpc
- permit udp any eq domain any
- permit tcp any eq domain any
- permit icmp 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255 echo-reply
- permit tcp 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255 established
- deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
- permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
- permit ip 192.168.20.0 0.0.0.255 host 192.168.30.4
- permit ip 192.168.20.0 0.0.0.255 any

## Services on SRV-CORE-01 (Packet Tracer server)
Domain: wow.com
Enabled services:
- DNS
- Email (SMTP/POP3)
- FTP

Mailboxes tested:
- admin@wow.com
- sale@wow.com
- fin@wow.com
(Tested sending from each PC to the others.)

FTP:
- FTP service is enabled with a user account.
- Tested login from a PC in another VLAN (works).

## Files in this repository
- .pkt — Packet Tracer project file (open with Cisco Packet Tracer 8.2.2)
- /configs — device configs (routers/switches)
- /screenshots — verification screenshots (HSRP, EIGRP neighbors, routes, VLANs, trunks, DHCP, ACL hits, services)
- Routing_Tables.xlsx — addressing/routing table for the lab

## How to run
1) Install Cisco Packet Tracer 8.2.2.
2) Open the .pkt file.
3) Wait ~10–20 seconds for routing/HSRP/EIGRP to converge.
4) From PCs, verify:
   - DHCP address obtained
   - Ping between VLAN10 <-> VLAN20 (allowed)
   - Ping/access to SRV-CORE-01 (allowed)
   - Attempt access to VLAN99 from VLAN10/VLAN20 (blocked)
   - Email send/receive and DNS resolution
   - FTP login to 192.168.30.4
   - Clients use HSRP VIP as default gateway (.10.254 / .20.254)	

## Notes / limitations
- This is a learning lab built in Packet Tracer (simulation), not a production network.
- Some real-world behaviors (timers, hardware-specific features, etc.) may differ on physical devices.

Author: Roman Saratovskii
