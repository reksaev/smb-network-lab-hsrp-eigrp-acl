# ACL.md — VLAN access control (Packet Tracer lab)

This lab uses extended ACLs to enforce simple segmentation between user VLANs and the admin VLAN.

## Goal / policy
- VLAN10 (Sales) and VLAN20 (Finance):
  - Allowed:
    - Get IP via DHCP (through relay)
    - Use DNS
    - Communicate with each other (VLAN10 <-> VLAN20)
    - Access the server SRV-CORE-01 (192.168.30.4) for internal services (DNS/Email/FTP)
  - Denied:
    - Any direct access to VLAN99 (Admin)

- VLAN99 (Admin):
  - Full access to all VLANs (no ACL applied to VLAN99 in this lab)

## Where ACLs are applied
ACLs are applied inbound on the EDGE routers’ user subinterfaces:

- On R-EDGE-01:
  - Gi0/1.10 → `ip access-group ACL_VLAN10_IN in`
  - Gi0/1.20 → `ip access-group ACL_VLAN20_IN in`

- On R-EDGE-02:
  - Gi0/2.10 → `ip access-group ACL_VLAN10_IN in`
  - Gi0/2.20 → `ip access-group ACL_VLAN20_IN in`

Inbound direction was chosen so unwanted traffic is blocked as close to the source as possible.

## Why DHCP/DNS rules are at the top
ACLs are processed top-down, first match wins.
If DHCP and DNS aren’t explicitly permitted before the deny rules, clients may fail to:
- obtain an IP address (DHCP discover/offer/request/ack),
- resolve names (DNS).

DHCP (client/server ports):
- Client → Server: UDP 68 → 67 (bootpc → bootps)
- Server → Client: UDP 67 → 68 (bootps → bootpc)

DNS ports:
- UDP 53 and TCP 53 (Packet Tracer services often work fine with both permitted)

## ACL definitions (as used in the lab)

### ACL_VLAN10_IN
Applied inbound on VLAN10 subinterfaces.

ip access-list extended ACL_VLAN10_IN
 permit udp any eq bootpc any eq bootps
 permit udp any eq bootps any eq bootpc
 permit udp any eq domain any
 permit tcp any eq domain any
 permit icmp 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 echo-reply
 permit tcp  192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255 established
 deny ip 192.168.10.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip 192.168.10.0 0.0.0.255 host 192.168.30.4
 permit ip 192.168.10.0 0.0.0.255 any

Notes:
- `deny` to VLAN99 enforces “no access to Admin VLAN”.
- ICMP echo-reply and TCP established are included to allow return traffic where relevant.
- The final `permit ip ... any` is intentional to avoid over-blocking traffic outside VLAN99 while still enforcing the VLAN99 deny.

### ACL_VLAN20_IN
Applied inbound on VLAN20 subinterfaces.

ip access-list extended ACL_VLAN20_IN
 permit udp any eq bootpc any eq bootps
 permit udp any eq bootps any eq bootpc
 permit udp any eq domain any
 permit tcp any eq domain any
 permit icmp 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255 echo-reply
 permit tcp  192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255 established
 deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip 192.168.20.0 0.0.0.255 host 192.168.30.4
 permit ip 192.168.20.0 0.0.0.255 any

## Verification checklist
Recommended show commands:
- `show access-lists`
- `show ip interface g0/1.10` / `show ip interface g0/1.20`
- `show ip interface g0/2.10` / `show ip interface g0/2.20`

Functional tests:
- From VLAN10 and VLAN20 PCs:
  - DHCP works (client gets IP, GW, DNS)
  - DNS queries resolve (wow.com)
  - Ping VLAN10 <-> VLAN20 works
  - Access to 192.168.30.4 works (server services)
  - Access to VLAN99 subnet is blocked
- From Admin PC (VLAN99):
  - Can reach VLAN10, VLAN20, VLAN30

## Practical note
If traffic suddenly stops after applying an ACL:
1) Confirm the ACL is applied to the correct interface/subinterface.
2) Confirm direction (in vs out).
3) Ensure DHCP and DNS permits are placed above denies.
4) Check hit counters in `show access-lists` to see what line is matching.
