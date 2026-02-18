# Verification.md — What I tested and how to reproduce

This file is a quick “proof” that the lab is working end-to-end.  
All checks below were done in Cisco Packet Tracer 8.2.2 on Windows 11.

## 1) Layer 2 (VLANs / trunks / EtherChannel)

### VLANs on Office switch (Sales/Finance)
Command:
- show vlan brief
Expected:
- VLAN10 and VLAN20 exist
- PC-SALES-01 port is in VLAN10
- PC-FIN-01 port is in VLAN20

### Trunks (Office switch uplinks)
Command:
- show interfaces trunk
Expected:
- Uplink ports are trunking
- VLANs 10 and 20 are allowed and active

### LACP EtherChannel (Core <-> Admin)
Command (on both SW-CORE-01 and SW-ADMIN-01):
- show etherchannel summary
Expected:
- Po1 is up
- Member ports are (P) bundled in the port-channel
- Po1 is trunk

## 2) Routing (EIGRP)

Command (on all routers):
- show ip eigrp neighbors
Expected:
- All EIGRP adjacencies are up

Command (on all routers):
- show ip route
Expected:
- Remote VLAN networks are present as EIGRP-learned routes
- Local subnets show as connected

## 3) Gateway redundancy (HSRP)

Command (on R-EDGE-01 and R-EDGE-02):
- show standby brief
Expected:
- VLAN10 group shows a single Active and a single Standby router
- VLAN20 group shows a single Active and a single Standby router
- Virtual IPs:
  - VLAN10: 192.168.10.254
  - VLAN20: 192.168.20.254
Note:
- In my lab, Active is R-EDGE-01 by default.

## 4) DHCP (Core router + DHCP relay)

### DHCP pools on R-CORE-01
Command:
- show ip dhcp pool
Expected:
- Pools exist for VLAN10 / VLAN20 / VLAN30 / VLAN99
- Leases increase after PCs request addresses

### DHCP relay on EDGE subinterfaces
Command (examples):
- show ip interface g0/1.10
- show ip interface g0/1.20
(or on R-EDGE-02: g0/2.10 and g0/2.20)

Expected:
- “Helper address” is set (points to the DHCP server on the Core side)

### Client verification (PCs)
On each PC:
- Desktop -> IP Configuration -> DHCP
Expected:
- PC gets an IP in the correct subnet
- Default gateway is the HSRP virtual IP (…10.254 or …20.254)
- DNS server is 192.168.30.4

## 5) DNS

Server:
- SRV-CORE-01 has DNS enabled and configured for the internal zone wow.com

Client test:
- From any PC: open Command Prompt and test name resolution (or use browser/mail by name)
Expected:
- wow.com records resolve correctly using SRV-CORE-01 (192.168.30.4)

## 6) Email

Server:
- SRV-CORE-01 Email service is enabled for wow.com
- Mailboxes exist (admin@wow.com, sale@wow.com, fin@wow.com)

Client test (on each PC):
- Desktop -> Email -> configure account -> send test message to another mailbox
Expected:
- Messages are delivered between users across VLANs (Sales <-> Finance and to Admin when allowed)

## 7) FTP

Server:
- SRV-CORE-01 FTP service is enabled
- At least one user exists (example: sale / password set)

Client test:
- From a PC: open Command Prompt -> ftp 192.168.30.4 -> login -> dir / put / get
Expected:
- Login succeeds
- Directory listing works
- Upload/download works based on FTP permissions and server folder configuration

## 8) ACL segmentation (VLAN10/VLAN20 -> block VLAN99)

Command (on EDGE routers):
- show access-lists
- show ip interface g0/1.10 / g0/1.20 (or g0/2.10 / g0/2.20)

Expected policy:
- VLAN10 and VLAN20 can reach each other and the server (192.168.30.4)
- VLAN10 and VLAN20 cannot initiate access to VLAN99
- Admin PC (VLAN99) can reach all VLANs (no ACL applied to VLAN99 in this lab)

Quick test matrix:
- PC-SALES-01 -> PC-FIN-01: PASS
- PC-FIN-01 -> PC-SALES-01: PASS
- PC-SALES-01 -> 192.168.30.4 (server): PASS
- PC-FIN-01 -> 192.168.30.4 (server): PASS
- PC-SALES-01 -> VLAN99 host: FAIL (blocked)
- PC-FIN-01 -> VLAN99 host: FAIL (blocked)
- PC-ADMIN-01 -> VLAN10/VLAN20/VLAN30: PASS
