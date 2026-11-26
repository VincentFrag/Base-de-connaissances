---
icon: material/book-open-variant
title: VPN
tags: [réseau, intermédiaire, concept, vpn, wireguard, accès distant]
---

# 🧠 VPN

!!! abstract "En une phrase"
    Un **VPN** (*Virtual Private Network*) crée un **tunnel chiffré** à travers Internet pour qu'une machine (ou tout un site) se retrouve **comme branchée sur un réseau privé distant**.

## L'analogie

Un VPN, c'est un **couloir privé** entre deux bâtiments, construit au milieu de la rue (Internet). Les passants voient qu'il y a un couloir, mais pas qui marche dedans ni ce qu'il transporte. Une fois au bout, tu es **à l'intérieur** du bâtiment, avec les mêmes accès que les gens sur place.

**SSH** ouvre une porte vers **une machine**. Un **VPN** te fait entrer dans **tout le réseau** : serveurs, interface web Proxmox, partages de fichiers, imprimante…

## Les deux grandes familles

| | **Site-à-site** | **Nomade** (*client-to-site*, accès distant) |
|---|---|---|
| Qui est relié | Deux **réseaux** entiers (siège ↔ agence) | Un **appareil** (PC, smartphone) ↔ un réseau |
| Qui porte le tunnel | Les **routeurs / pare-feux** des deux sites | Un **logiciel client** sur l'appareil |
| Quand il est actif | En permanence | Quand l'utilisateur l'allume |
| Les postes savent-ils qu'il y a un VPN ? | Non, c'est transparent | Oui, ils lancent le client |
| Exemple | L'agence de Lyon voit les serveurs du siège à Paris | Un technicien en télétravail, ou toi qui joins ton homelab depuis l'école |

```
Site-à-site :   [LAN Paris]──[Routeur]════tunnel════[Routeur]──[LAN Lyon]

Nomade :        [PC portable]════tunnel════[Serveur VPN]──[LAN du lab]
                 (client VPN)                (dans le lab)
```

## Les protocoles courants

| Protocole | Points forts | Limites | Où on le croise |
|---|---|---|---|
| **WireGuard** | Très simple (un fichier de config), rapide, moderne | Peu d'options (pas de login/mot de passe, uniquement des clés) | Homelabs, Linux, pfSense, Tailscale |
| **OpenVPN** | Très répandu, passe presque partout (TCP 443 possible) | Config plus lourde (certificats), plus lent | Entreprises, pfSense, fournisseurs VPN |
| **IPsec** (IKEv2) | Standard, intégré aux routeurs et à Windows | Complexe à configurer et à dépanner | **Site-à-site** entre équipements (Cisco, Fortinet, pfSense) |

## Comment fonctionne WireGuard

| Élément | Rôle |
|---|---|
| **Paire de clés** | Chaque appareil a une clé **privée** (secrète) et une clé **publique** (qu'il donne aux autres), comme pour SSH |
| **Interface `wg0`** | Une carte réseau virtuelle avec sa propre IP dans le **réseau du tunnel** (ex. `10.8.0.0/24`) |
| **Pair** (*peer*) | Un autre appareil autorisé, identifié par sa clé publique |
| **`Endpoint`** | Adresse publique et port où joindre le serveur (UDP **51820** par défaut) |
| **`AllowedIPs`** | Les réseaux qui passent dans le tunnel : c'est à la fois une **route** et un **filtre** |
| **`PersistentKeepalive`** | Un petit paquet régulier pour garder le tunnel ouvert à travers la box |

## Tunnel complet ou tunnel partagé ?

| | **Split tunnel** (partagé) | **Full tunnel** (complet) |
|---|---|---|
| `AllowedIPs` côté client | Seulement le réseau du lab (ex. `10.0.0.0/24`) | `0.0.0.0/0` (tout) |
| Trafic vers le lab | Dans le tunnel | Dans le tunnel |
| Navigation Internet | En direct, par ta connexion locale | Passe par le lab (et sort par ta box) |
| Usage | Administrer son homelab | Wi-Fi public douteux, politique d'entreprise |

## À quoi ça sert concrètement

- Joindre **tout ton homelab** depuis l'extérieur (interface Proxmox, SSH, partages) en n'ouvrant **qu'un seul port UDP** sur la box.
- Relier deux sites d'une entreprise comme s'ils n'en faisaient qu'un.
- Donner aux télétravailleurs l'accès aux serveurs internes sans les exposer sur Internet.
- Se protéger sur un Wi-Fi public (tunnel complet).

!!! warning "Pièges fréquents"
    - **Même plage d'adresses des deux côtés** : si le réseau de l'école ou de l'hôtel est aussi en `192.168.1.0/24`, ton PC ne sait plus où envoyer le trafic. D'où l'intérêt d'un lab dans une plage peu courante (`10.0.0.0/24`).
    - **IP publique qui change** : la plupart des box ont une IP dynamique. Utiliser un nom **DynDNS** (DuckDNS, No-IP…) dans `Endpoint`.
    - **CGNAT** : certains opérateurs (4G/5G, fibre récente) partagent une IP publique entre clients, aucune redirection de port n'est alors possible. Solution : demander une IPv4 dédiée à l'opérateur, ou passer par un service relais (Tailscale).
    - **Routes de retour oubliées** : le tunnel monte, mais les serveurs du lab ne savent pas répondre au réseau du tunnel. Faire du **NAT** sur le serveur VPN, ou ajouter une route.
    - Un VPN n'est **pas une protection en soi** : un appareil volé avec sa config donne accès au lab. Une clé par appareil, et on supprime le pair en cas de perte.

## Pour aller plus loin

- 🛠️ **[Accéder à son homelab avec WireGuard](tuto-wireguard-acces-distant.md)** : la mise en pratique (VPN nomade)
- 🧠 [SSH](concept-ssh.md) et 🛠️ [SSH par rebond](tuto-ssh-rebond.md) (l'autre façon de joindre un réseau interne, machine par machine)
- 🧠 [Pare-feu et NAT](concept-pare-feu-nat.md) (redirection de port, NAT de sortie)
