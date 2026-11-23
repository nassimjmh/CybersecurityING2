# LAB Firewall DMZ — guide pas à pas (fin de `04 Firewall.pdf`, slides 9-13)

## Topologie (slide 9)
`PC0 — Switch0 — R0 — ASA0 — Cloud — Server0(Google)` et `ASA0 — Switch DMZ — web-server + DNS`

| Lien | Réseau | Adresses |
|---|---|---|
| PC0 ↔ R0 (g0/1) | 192.168.1.0/24 | PC0 .1 (GW .254) / R0 .254 |
| R0 (g0/0) ↔ ASA (1/1 **inside**) | 10.0.0.0/30 | R0 .1 / ASA .2 |
| ASA (1/2 **dmz**) ↔ switch DMZ | 172.16.1.0/24 | ASA .254 / Web .1 / DNS .2 |
| ASA (1/3 **outside**) ↔ Cloud | 203.0.113.0/29 | ASA .2 / Cloud .1 *(choix perso, .3 est pris par le NAT du web)* |
| Cloud ↔ Server0 | 192.168.1.0/24 | Cloud .254 / Google .1 |

Niveaux de sécurité : inside 100, dmz 50, outside 0. Trafic haut→bas autorisé par défaut, bas→haut bloqué sauf ACL.

> Hypothèse : l'IP publique du Google DNS (étape 12) n'est pas donnée dans le sujet, j'utilise **203.0.113.4**.

## Étapes 1-9 : IP des machines
- PC0 : 192.168.1.1/24, GW 192.168.1.254, DNS 172.16.1.2
- Web : 172.16.1.1/24, GW 172.16.1.254, DNS 172.16.1.2 → Services : HTTP + HTTPS ON
- DNS : 172.16.1.2/24, GW 172.16.1.254 → Services > DNS ON, A record `test.com` → `172.16.1.1`
- Server0 (Google) : 192.168.1.1/24, GW 192.168.1.254, DNS 192.168.1.1 → DNS ON, A record `test.com` → `203.0.113.3` (étape 13)

## Router R0 (étapes 2, 3, 4, 17)
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

## ASA0 (étapes 5, 6, 10, 14-16, 18-23)
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

! 14 - route par défaut vers le Cloud
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

! 18 - PAT inside + dmz vers outside
object network INSIDE-NET
 subnet 10.0.0.0 255.255.255.252
 nat (inside,outside) dynamic interface
object network DMZ-NET
 subnet 172.16.1.0 255.255.255.0
 nat (dmz,outside) dynamic interface

! 19 - Static NAT du web server
object network WEB-SRV
 host 172.16.1.1
 nat (dmz,outside) static 203.0.113.3

! 20-21 - inside -> Internet : HTTPS + ping seulement (+ DNS vers le serveur DMZ)
access-list INSIDE-IN extended permit tcp any any eq 443
access-list INSIDE-IN extended permit icmp any any echo
access-list INSIDE-IN extended permit udp any host 172.16.1.2 eq 53
access-group INSIDE-IN in interface inside

! 22-23 - outside -> entrée : HTTPS vers le web (IP RÉELLE 172.16.1.1), retour du ping, ICMP initié bloqué
access-list OUTSIDE-IN extended permit tcp any host 172.16.1.1 eq 443
access-list OUTSIDE-IN extended permit icmp any 10.0.0.0 255.255.255.252 echo-reply
access-list OUTSIDE-IN extended deny icmp any any
access-group OUTSIDE-IN in interface outside
end
write memory
```
Pièges à retenir (type examen) :
- Depuis ASA 8.3, les ACL utilisent l'**IP réelle** (172.16.1.1), pas l'IP publique 203.0.113.3.
- `nat (dmz,outside)` = `(interface source, interface destination)`.
- L'ICMP n'est pas stateful par défaut → d'où le `echo-reply` autorisé en entrée (comme dans l'exemple du slide 7).
- Le PAT de R0 fait que l'ASA voit toujours 10.0.0.1 comme source de l'inside.

## Cloud router (étapes 11, 12)
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

## Vérifications
| Test | Attendu |
|---|---|
| PC0 : `ping 172.16.1.2` | probablement **échec** : le echo-reply dmz→inside est bloqué (ICMP non inspecté, non demandé par le sujet) |
| PC0 : navigateur `https://test.com` | page du web server (DNS local → 172.16.1.1) |
| PC0 : `ping 203.0.113.4` | OK (étape 20) |
| Server0 : navigateur `https://test.com` | page du web server via 203.0.113.3 (étape 22) |
| Server0 : `ping 203.0.113.3` | **échec** (étape 23) |
| ASA : `show nat`, `show xlate`, `show access-list`, `show route`, `show run` | traductions et hits d'ACL |
| R0 : `show ip nat translations`, `show ip route` | PAT + route par défaut RIP |
