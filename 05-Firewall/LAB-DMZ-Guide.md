# Firewall DMZ Lab — step-by-step guide (end of `04 Firewall.pdf`, slides 9-13)

## Topology (slide 9)
`PC0 — Switch0 — R0 — ASA0 — Cloud — Server0 (Google)` and `ASA0 — DMZ switch — web-server + DNS`

| Link | Network | Addresses |
|---|---|---|
| PC0 ↔ R0 (g0/1) | 192.168.1.0/24 | PC0 .1 (GW .254) / R0 .254 |
| R0 (g0/0) ↔ ASA (1/1 **inside**) | 10.0.0.0/30 | R0 .1 / ASA .2 |
| ASA (1/2 **dmz**) ↔ DMZ switch | 172.16.1.0/24 | ASA .254 / Web .1 / DNS .2 |
| ASA (1/3 **outside**) ↔ Cloud | 203.0.113.0/29 | ASA .2 / Cloud .1 *(my choice; .3 is taken by the web server's NAT)* |
| Cloud ↔ Server0 | 192.168.1.0/24 | Cloud .254 / Google .1 |

Security levels: inside 100, dmz 50, outside 0. Traffic from high to low is allowed by default; low to high is blocked unless an ACL permits it.

> Assumption: the public IP of the Google DNS server (step 12) is not given in the assignment, so I use **203.0.113.4**.

## Steps 1-9, 13: host IPs and services
- PC0: 192.168.1.1/24, GW 192.168.1.254, DNS 172.16.1.2
- Web server: 172.16.1.1/24, GW 172.16.1.254, DNS 172.16.1.2 → Services: HTTP + HTTPS ON
- DNS server: 172.16.1.2/24, GW 172.16.1.254 → Services > DNS ON, A record `test.com` → `172.16.1.1`
- Server0 (Google): 192.168.1.1/24, GW 192.168.1.254, DNS 192.168.1.1 → DNS ON, A record `test.com` → `203.0.113.3`

## Router R0 (steps 2, 3, 4, 17)
```
enable
conf t
interface g0/1
 ip address 192.168.1.254 255.255.255.0
 ip nat inside
 no shutdown
interface g0/0
 ip address 10.0.0.1 255.255.255.252
 ip nat outside
 no shutdown
access-list 1 permit 192.168.1.0 0.0.0.255
ip nat inside source list 1 interface g0/0 overload
router rip
 version 2
 no auto-summary
 network 10.0.0.0
 network 192.168.1.0
end
```

## ASA0 (steps 5, 6, 10, 14-16, 18-23)
```
enable
conf t
interface g1/1
 nameif inside
 security-level 100
 ip address 10.0.0.2 255.255.255.252
 no shutdown
interface g1/2
 nameif dmz
 security-level 50
 ip address 172.16.1.254 255.255.255.0
 no shutdown
interface g1/3
 nameif outside
 security-level 0
 ip address 203.0.113.2 255.255.255.248
 no shutdown

! 14 - default route to the Cloud router
route outside 0.0.0.0 0.0.0.0 203.0.113.1

! 15-16 - RIPv2
router rip
 version 2
 no auto-summary
 network 10.0.0.0
 network 172.16.0.0
 default-information originate
 passive-interface outside
 passive-interface dmz

! 18 - PAT for inside + dmz towards outside
object network INSIDE-NET
 subnet 10.0.0.0 255.255.255.252
 nat (inside,outside) dynamic interface
object network DMZ-NET
 subnet 172.16.1.0 255.255.255.0
 nat (dmz,outside) dynamic interface

! 19 - Static NAT for the web server
object network WEB-SRV
 host 172.16.1.1
 nat (dmz,outside) static 203.0.113.3

! 20-21 - inside -> Internet: HTTPS + ping only (+ DNS to the DMZ server)
access-list INSIDE-IN extended permit tcp any any eq 443
access-list INSIDE-IN extended permit icmp any any echo
access-list INSIDE-IN extended permit udp any host 172.16.1.2 eq 53
access-group INSIDE-IN in interface inside

! 22-23 - outside -> in: HTTPS to the web server (REAL IP 172.16.1.1), ping replies, block initiated ICMP
access-list OUTSIDE-IN extended permit tcp any host 172.16.1.1 eq 443
access-list OUTSIDE-IN extended permit icmp any 10.0.0.0 255.255.255.252 echo-reply
access-list OUTSIDE-IN extended deny icmp any any
access-group OUTSIDE-IN in interface outside
end
write memory
```
Exam pitfalls:
- Since ASA 8.3, ACLs use the **real IP** (172.16.1.1), not the public IP 203.0.113.3.
- `nat (dmz,outside)` = `(source interface, destination interface)`.
- ICMP is not stateful by default, hence the `echo-reply` permit inbound (like the example on slide 7).
- R0's PAT means the ASA always sees 10.0.0.1 as the inside source.

## Cloud router (steps 11, 12)
```
enable
conf t
interface g0/0
 ip address 203.0.113.1 255.255.255.248
 ip nat outside
 no shutdown
interface g0/1
 ip address 192.168.1.254 255.255.255.0
 ip nat inside
 no shutdown
ip nat inside source static 192.168.1.1 203.0.113.4
end
```

## Verification
| Test | Expected |
|---|---|
| PC0: `ping 172.16.1.2` | probably **fails**: the dmz→inside echo-reply is blocked (ICMP not inspected, not required by the assignment) |
| PC0: browser `https://test.com` | web server page (local DNS → 172.16.1.1) |
| PC0: `ping 203.0.113.4` | OK (step 20) |
| Server0: browser `https://test.com` | web server page via 203.0.113.3 (step 22) |
| Server0: `ping 203.0.113.3` | **fails** (step 23) |
| ASA: `show nat`, `show xlate`, `show access-list`, `show route`, `show run` | translations and ACL hit counts |
| R0: `show ip nat translations`, `show ip route` | PAT + RIP default route |
