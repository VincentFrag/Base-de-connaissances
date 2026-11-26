---
icon: material/clipboard-text-outline
title: Mémo références réseau
tags: [référence, mémo, réseau, adressage, DNS, HTTP]
---

# 📋 Mémo références réseau

!!! abstract "Usage"
    Les tableaux qu'on finit toujours par rechercher : plages d'adresses, masques, enregistrements DNS, codes HTTP. Pour les ports : 📋 [Mémo ports et protocoles](memo-ports-protocoles.md).

## Plages IP privées et spéciales

| Plage | Nom | À quoi ça sert |
|---|---|---|
| `10.0.0.0/8` | Privée (RFC 1918) | Réseaux d'entreprise (le lab utilise `10.0.0.0/24`) |
| `172.16.0.0/12` | Privée (RFC 1918) | De `172.16.0.0` à `172.31.255.255` (Docker utilise `172.17.0.0/16`) |
| `192.168.0.0/16` | Privée (RFC 1918) | Box et réseaux domestiques |
| `127.0.0.0/8` | Loopback | La machine elle-même (`127.0.0.1` = `localhost`) |
| `169.254.0.0/16` | APIPA / lien local | Auto-attribuée quand le DHCP ne répond pas |
| `100.64.0.0/10` | CGNAT | NAT des opérateurs (et Tailscale) |
| `0.0.0.0` | Toutes / aucune | « Toutes les interfaces » pour un service, « pas encore d'IP » pour un client DHCP |
| `255.255.255.255` | Broadcast limité | Tout le réseau local (utilisé par le DHCP Discover) |
| `224.0.0.0/4` | Multicast | Envoi à un groupe de machines |

:material-arrow-right: Détails : [Adressage IP — adresses privées vs publiques](../reseau/concept-adressage-ip.md)

**En IPv6**

| Préfixe | Équivalent IPv4 | Usage |
|---|---|---|
| `::1` | `127.0.0.1` | Loopback |
| `fe80::/10` | `169.254.0.0/16` | Lien local (toujours présent sur une interface) |
| `fc00::/7` | Plages RFC 1918 | Adresses privées (ULA) |
| `2000::/3` | IP publiques | Adresses publiques routables |

## Masques CIDR

| CIDR | Masque | Adresses | Machines utilisables | Exemple |
|---|---|---|---|---|
| `/8` | `255.0.0.0` | 16 777 216 | 16 777 214 | `10.0.0.0/8` |
| `/16` | `255.255.0.0` | 65 536 | 65 534 | `172.16.0.0/16` |
| `/22` | `255.255.252.0` | 1 024 | 1 022 | Gros site |
| `/23` | `255.255.254.0` | 512 | 510 | |
| `/24` | `255.255.255.0` | 256 | 254 | `10.0.0.0/24`, le plus courant |
| `/25` | `255.255.255.128` | 128 | 126 | Moitié d'un /24 |
| `/26` | `255.255.255.192` | 64 | 62 | |
| `/27` | `255.255.255.224` | 32 | 30 | |
| `/28` | `255.255.255.240` | 16 | 14 | Petite DMZ |
| `/29` | `255.255.255.248` | 8 | 6 | |
| `/30` | `255.255.255.252` | 4 | 2 | Lien entre deux routeurs |
| `/32` | `255.255.255.255` | 1 | 1 | Une seule machine (règle de pare-feu) |

!!! tip "Le calcul rapide"
    Machines utilisables = 2^(32 − CIDR) − 2 (on retire l'adresse réseau et le broadcast). :material-arrow-right: [Découper en sous-réseaux](../reseau/tuto-sous-reseaux.md)

## Types d'enregistrements DNS

| Type | Contient | Exemple |
|---|---|---|
| **A** | Nom → IPv4 | `srv-web01.lab.local → 10.0.0.31` |
| **AAAA** | Nom → IPv6 | `srv-web01.lab.local → fd00::31` |
| **CNAME** | Alias → autre nom | `www.lab.local → srv-lb01.lab.local` |
| **PTR** | IP → nom (zone inverse) | `10.0.0.10 → srv-dc01.lab.local` |
| **MX** | Serveur de mail du domaine | `lab.local → mail.lab.local` (priorité 10) |
| **NS** | Serveurs DNS qui font autorité sur la zone | `lab.local → srv-dc01.lab.local` |
| **SOA** | Infos de la zone (serveur principal, numéro de série) | Un seul par zone |
| **SRV** | Emplacement d'un service | `_ldap._tcp.lab.local → srv-dc01:389` (indispensable à l'AD) |
| **TXT** | Texte libre (SPF, vérifications) | `v=spf1 mx -all` |

:material-arrow-right: Détails : [DHCP et DNS](../reseau/concept-dhcp-dns.md)

## Codes HTTP

| Code | Sens | Ce que ça veut dire en pratique |
|---|---|---|
| **1xx** | Information | Rare à voir |
| `101` | Switching Protocols | Passage en WebSocket |
| **2xx** | Succès | |
| `200` | OK | Tout va bien |
| `201` | Created | Ressource créée (API) |
| `204` | No Content | Succès, rien à renvoyer |
| **3xx** | Redirection | |
| `301` | Moved Permanently | Redirection définitive (ex. HTTP → HTTPS) |
| `302` | Found | Redirection temporaire |
| `304` | Not Modified | Le navigateur utilise sa copie en cache |
| **4xx** | Erreur du client | La demande pose problème |
| `400` | Bad Request | Requête mal formée |
| `401` | Unauthorized | Il faut s'authentifier |
| `403` | Forbidden | Authentifié mais pas autorisé (droits sur les fichiers, règle) |
| `404` | Not Found | La page n'existe pas |
| `405` | Method Not Allowed | Mauvaise méthode (GET au lieu de POST…) |
| `408` | Request Timeout | Le client a mis trop de temps |
| `429` | Too Many Requests | Trop de requêtes, limitation de débit |
| **5xx** | Erreur du serveur | Le problème est côté serveur |
| `500` | Internal Server Error | L'application a planté : lire ses journaux |
| `502` | Bad Gateway | Le proxy ou le répartiteur n'arrive pas à joindre le serveur derrière |
| `503` | Service Unavailable | Service arrêté ou surchargé (ex. aucun serveur sain derrière HAProxy) |
| `504` | Gateway Timeout | Le serveur derrière le proxy répond trop lentement |

```bash
# Voir le code renvoyé par un site
curl -I http://srv-lb01.lab.local
```

## Couches OSI en un coup d'œil

| N° | Couche | Unité | Exemples | Équipement |
|---|---|---|---|---|
| 7 | Application | Données | HTTP, DNS, SSH, SMB | Pare-feu applicatif, proxy |
| 6 | Présentation | Données | TLS, encodage | |
| 5 | Session | Données | RPC, NetBIOS | |
| 4 | Transport | Segment | TCP, UDP | Pare-feu (ports) |
| 3 | Réseau | Paquet | IP, ICMP | Routeur |
| 2 | Liaison | Trame | Ethernet, ARP, 802.1Q | Switch |
| 1 | Physique | Bit | Câble, Wi-Fi, fibre | Hub, répéteur |

:material-arrow-right: Détails : [Le modèle OSI et TCP/IP](../reseau/concept-modele-osi-tcpip.md)

## Unités

| Unité | Valeur | Piège |
|---|---|---|
| 1 octet (o, B) | 8 bits (b) | Les débits sont en **bits**, les fichiers en **octets** : 1 Gbit/s ≈ 125 Mo/s |
| 1 Ko / 1 Kio | 1 000 / 1 024 octets | Un disque de 1 To affiche ≈ 931 Gio dans l'OS |
| 1 ms | Latence | < 1 ms en local, 10 à 40 ms vers Internet |
