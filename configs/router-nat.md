# Routeur NAT — Configuration

Assure la translation d'adresses (NAT/PAT) entre le WAN public (`200.0.0.0/30`) et le réseau interne.  
Filtre également le trafic entrant depuis Internet et sortant vers le WAN.

---

## Interfaces

| Interface | IP | Masque | Rôle |
|---|---|---|---|
| G1/0 | `200.0.0.2` | `/30` | Vers Routeur Internet (WAN) |
| G2/0 | `10.1.2.2` | `/30` | Vers Firewall |

## Route par défaut

```cisco
ip route 0.0.0.0 0.0.0.0 200.0.0.1
```

---

## Configuration des interfaces

```cisco
interface GigabitEthernet2/0
 ip address 200.0.0.1 255.255.255.252
 ip nat outside
 ip access-group ACL-NAT-OUTSIDE in
 no shutdown
!
interface GigabitEthernet1/0
 ip address 10.1.2.2 255.255.255.252
 ip nat inside
 ip access-group ACL-NAT-INSIDE in
 no shutdown
```

---

## Configuration NAT

```cisco
! NAT dynamique — LAN10 et DMZ vers WAN
ip nat inside source list NAT-TRAFFIC interface GigabitEthernet1/0 overload

! Liste des réseaux autorisés à sortir via NAT
ip access-list standard NAT-TRAFFIC
 permit 192.168.10.0 0.0.0.255
 permit 172.16.0.0 0.0.255.255
```

---

## ACL-NAT-OUTSIDE — G1/0 (in)

Filtre le trafic entrant depuis Internet vers le réseau interne.

```cisco
ip access-list extended ACL-NAT-OUTSIDE
 ! SSH — client WAN autorisé
 permit tcp 10.0.1.0 0.0.0.3 any eq 22
 ! OpenVPN — client WAN autorisé
 permit udp 10.0.2.0 0.0.0.3 any eq 1194
 ! WEB DMZ — HTTPS et HTTP
 permit tcp 10.0.3.0 0.0.0.3 any eq 443
 permit tcp 10.0.3.0 0.0.0.3 any eq 80
 ! Conférence — HTTPS et flux multimédia
 permit tcp 10.0.3.0 0.0.0.3 any eq 8080
 permit udp 10.0.3.0 0.0.0.3 any eq 10000
 ! NTP — retour de synchronisation
 permit udp any any eq 123
 ! ICMP — debug uniquement
 permit icmp any any
 ! Tout le reste bloqué
 deny ip any any
```

---

## ACL-NAT-INSIDE — G2/0 (in)

Filtre le trafic sortant depuis le réseau interne vers le WAN.

```cisco
ip access-list extended ACL-NAT-INSIDE
 ! LAN10 — accès WAN autorisé
 permit ip 192.168.10.0 0.0.0.255 any
 ! DMZ — accès WAN autorisé
 permit ip 172.16.0.0 0.0.255.255 any
 ! Logs → SIEM
 permit tcp any host 192.168.20.100 eq 5044
 ! NTP — requêtes sortantes
 permit udp any host 10.0.4.1 eq 123
 ! LAN20 — pas d'accès WAN
 deny ip 192.168.20.0 0.0.0.255 any
 ! LAN30 — pas d'accès WAN
 deny ip 192.168.30.0 0.0.0.255 any
 ! Tout le reste bloqué
 deny ip any any
```

---

## Notes

- Le NAT overload (PAT) utilise l'IP publique `200.0.0.2` pour masquer les adresses internes
- Seuls LAN10 (`192.168.10.0/24`) et la DMZ (`172.16.0.0/24`) sont autorisés à sortir via NAT
- LAN20 et LAN30 sont explicitement bloqués vers le WAN
- Les clients WAN sont identifiés par sous-réseau `/30` dédié par service (SSH, OpenVPN, WEB/Conf, NTP)
