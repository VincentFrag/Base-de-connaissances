---
icon: material/clipboard-text-outline
title: Mémo commandes réseau Windows & Linux
tags: [réseau, mémo, commandes]
---

# 📋 Mémo commandes réseau Windows & Linux

!!! abstract "Usage"
    Retrouver la bonne commande en 10 secondes. Pour la méthode complète : 🛠️ [Diagnostiquer une connexion](tuto-diagnostic-connexion.md).

## Configuration IP

| Besoin | Windows | Linux |
|---|---|---|
| Voir l'IP, le masque, la passerelle, le DNS | `ipconfig /all` | `ip a` · `ip r` · `resolvectl status` |
| Renouveler l'IP (DHCP) | `ipconfig /release` puis `/renew` | `sudo dhclient -r && sudo dhclient` |
| Voir les interfaces | `Get-NetAdapter` | `ip link` |
| Voir la table de routage | `route print` | `ip r` |
| Voir l'adresse MAC | `getmac` | `ip link` |

## Tests de connectivité

| Besoin | Windows | Linux |
|---|---|---|
| Tester qu'une machine répond | `ping 192.168.1.1` | `ping -c 4 192.168.1.1` |
| Ping continu | `ping -t 1.1.1.1` | `ping 1.1.1.1` |
| Suivre le chemin des paquets | `tracert 1.1.1.1` | `traceroute 1.1.1.1` |
| Tester un port TCP | `Test-NetConnection srv -Port 443` | `nc -zv srv 443` |
| Tester une requête HTTP | `Invoke-WebRequest https://site` | `curl -I https://site` |

## DNS

| Besoin | Windows | Linux |
|---|---|---|
| Résoudre un nom | `nslookup site.fr` | `dig site.fr` · `nslookup site.fr` |
| Interroger un DNS précis | `nslookup site.fr 1.1.1.1` | `dig @1.1.1.1 site.fr` |
| Vider le cache DNS | `ipconfig /flushdns` | `resolvectl flush-caches` |
| Voir le cache DNS | `ipconfig /displaydns` | `resolvectl statistics` |

## Connexions et ports

| Besoin | Windows | Linux |
|---|---|---|
| Ports en écoute | `netstat -ano \| findstr LISTEN` | `ss -tulpn` |
| Connexions actives | `netstat -ano` | `ss -tan` |
| Qui utilise un port | `Get-NetTCPConnection -LocalPort 443` | `sudo ss -tlpn 'sport = :443'` |
| Table ARP (IP ↔ MAC) | `arp -a` | `ip neigh` |

## Équivalences historiques

!!! info "Anciennes commandes Linux"
    `ifconfig` → `ip a` · `route` → `ip r` · `netstat` → `ss` · `arp` → `ip neigh`.
    Les anciennes commandes (paquet `net-tools`) ne sont plus installées par défaut sur les distributions récentes.
