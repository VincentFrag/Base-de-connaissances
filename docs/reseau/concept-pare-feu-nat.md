---
icon: material/book-open-variant
title: Pare-feu et NAT
tags: [réseau, bases, concept, pare-feu, nat, pfsense]
---

# 🧠 Pare-feu et NAT

!!! abstract "En une phrase"
    Le **pare-feu** décide **qui a le droit de parler à qui** (et sur quel port), le **NAT** **réécrit les adresses** au passage pour que des machines en IP privée sortent sur Internet, ou soient joignables depuis l'extérieur.

## L'analogie

Une entreprise avec un **accueil** à l'entrée.

- Le **pare-feu**, c'est l'agent d'accueil avec sa **liste** : « Les commerciaux peuvent monter au 2ᵉ étage, mais pas au service paie. Les visiteurs ne passent pas. » Il lit la liste **de haut en bas** et applique la **première ligne qui correspond**. Si personne n'est sur la liste : on ne passe pas.
- Le **NAT**, c'est le **standard téléphonique** : de l'extérieur, tout le monde appelle **un seul numéro** (l'IP publique). Le standard sait que « l'appel pour le service web » doit être transféré au poste 20.10, et quand un salarié appelle dehors, c'est le numéro de l'entreprise qui s'affiche, pas son poste.

## Le pare-feu : filtrer le trafic

Une **règle** de pare-feu décrit un flux et ce qu'on en fait :

| Champ | Exemple | Question posée |
|---|---|---|
| **Interface** | `UTILISATEURS` (VLAN 10) | Par où le paquet **entre**-t-il ? |
| **Source** | `192.168.10.0/24` | Qui parle ? |
| **Destination** | `192.168.20.10` | À qui ? |
| **Protocole / port** | TCP `443` | Pour quel service ? |
| **Action** | *Pass*, *Block*, *Reject* | On laisse passer, on jette en silence, ou on refuse poliment |

Trois principes à retenir :

1. **Lecture de haut en bas, la première règle qui correspond gagne** : l'ordre compte. Une règle « tout autoriser » placée en haut rend inutiles toutes celles du dessous.
2. **Tout ce qui n'est pas autorisé est interdit** (*default deny*) : on ouvre au cas par cas, jamais l'inverse.
3. **Le filtrage se fait à l'entrée** : sur pfSense, la règle se pose sur l'interface **par laquelle le trafic arrive** (celle du client, pas celle du serveur).

### Sans état ou avec état

| Type | Fonctionnement | Exemple |
|---|---|---|
| **Sans état** (*stateless*) | Chaque paquet est jugé seul : il faut aussi autoriser **la réponse** dans l'autre sens | ACL Cisco classique |
| **Avec état** (*stateful*) | Le pare-feu **se souvient des connexions** ouvertes : la réponse repasse automatiquement | pfSense, nftables/iptables, pare-feu Windows |

Sur un pare-feu avec état, on n'écrit donc qu'**une règle par sens d'initiative** : « les utilisateurs peuvent **ouvrir** une connexion vers le serveur web ». Le retour est géré par la **table d'états**.

!!! tip "Block ou Reject ?"
    **Block** jette le paquet sans répondre : idéal côté Internet (on ne se dévoile pas). **Reject** renvoie un refus immédiat : pratique en interne, l'utilisateur obtient une erreur tout de suite au lieu d'attendre un délai d'expiration.

## Le NAT : traduire les adresses

Les adresses privées (`10.x`, `172.16-31.x`, `192.168.x`) ne circulent **pas sur Internet**. Le NAT les remplace au passage par l'adresse publique du routeur.

| Type de NAT | Ce qui est réécrit | Sens | Usage typique |
|---|---|---|---|
| **NAT sortant** (SNAT, *masquerade*, PAT, *overload*) | L'**adresse source** devient l'IP publique | Intérieur → Internet | Tous les PC sortent avec **une seule** IP publique |
| **Redirection de port** (DNAT, *port forward*) | L'**adresse de destination** devient l'IP d'un serveur interne | Internet → intérieur | Publier un site web interne sur le port 443 |
| **NAT 1:1** (statique) | Une IP publique entière ↔ une IP privée | Les deux | Un serveur qui a sa propre IP publique |

### Ce qui se passe pour un PC qui va sur Internet

```
PC 192.168.10.50:51000 ──▶ pfSense ──▶ Internet
                          │
     réécrit la source :  192.168.10.50:51000  →  203.0.113.5:62001
     et note la correspondance dans sa table pour renvoyer la réponse au bon PC
```

Plusieurs PC partagent la même IP publique : c'est le **port source** qui permet de les distinguer, d'où le nom **PAT** (*Port Address Translation*).

### Ce qui se passe pour une redirection de port

```
Client Internet ──▶ 203.0.113.5:443 ──▶ pfSense ──▶ 192.168.20.10:443 (serveur web)
                                        réécrit la destination
```

!!! info "NAT et pare-feu travaillent ensemble"
    Une redirection de port **traduit** l'adresse, mais il faut aussi une **règle de pare-feu** qui **autorise** le flux. Sur pfSense, la redirection est appliquée **avant** le filtrage : la règle s'écrit donc avec l'**IP interne** du serveur comme destination (pfSense propose de la créer automatiquement).

## À quoi ça sert concrètement

- Empêcher le VLAN **invités** ou **utilisateurs** d'accéder aux serveurs, sauf aux services prévus (web, DNS).
- Donner accès à Internet à tout un réseau privé avec **une seule** IP publique (c'est ce que fait ta box à la maison).
- Publier un service du homelab (site web, VPN) sans exposer le reste.
- Journaliser les tentatives bloquées pour comprendre qui essaie de joindre quoi.

!!! warning "Pièges fréquents"
    - **Règle au mauvais endroit** : sur pfSense, la règle se pose sur l'interface **d'entrée** du trafic (celle du client).
    - **Mauvais ordre** : une règle *Pass any* au-dessus d'une règle *Block* annule le blocage.
    - **Redirection de port sans règle** : le NAT est fait, mais le pare-feu jette le paquet.
    - **Tester une redirection depuis l'intérieur** : ça échoue souvent (il faut le *NAT reflection*). Teste depuis l'extérieur.
    - **Oublier que les états persistent** : après avoir durci une règle, une connexion déjà ouverte peut continuer à passer. Vider les états (*Diagnostics → States → Reset*) pour tester.
    - **WAN en IP privée dans un homelab** (pfSense derrière la box) : pfSense bloque par défaut les réseaux privés sur le WAN. Décocher *Block private networks* sur l'interface WAN.

## Pour aller plus loin

- 🛠️ **[Filtrer et faire du NAT avec pfSense](tuto-pfsense-pare-feu-nat.md)** : la mise en pratique (pfSense, avec équivalents Cisco IOS et Linux)
- 🧠 [Les VLAN](concept-vlan.md) (le pare-feu filtre le trafic entre VLAN)
- 🧠 [Adressage IP et masque de sous-réseau](concept-adressage-ip.md) (adresses privées et publiques)
- 🧠 [Le modèle OSI et TCP/IP](concept-modele-osi-tcpip.md) (filtrage en couches 3 et 4 : IP et ports)
