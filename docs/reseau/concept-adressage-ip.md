---
icon: material/book-open-variant
title: Adressage IP et masque de sous-réseau
tags: [réseau, bases, concept]
---

# 🧠 Adressage IP et masque de sous-réseau

!!! abstract "En une phrase"
    Une **adresse IP** identifie une machine sur un réseau ; le **masque** indique quelle partie de cette adresse désigne *le réseau* et quelle partie désigne *la machine*.

## L'analogie

Une adresse IP, c'est comme une adresse postale : **« 12 rue des Lilas »**.

- **La rue** = la partie *réseau* (tous les voisins la partagent)
- **Le numéro** = la partie *hôte* (unique dans la rue)

Le masque, c'est la règle qui dit *où s'arrête le nom de la rue et où commence le numéro*.

## Comment ça se lit

Une adresse IPv4 = **4 octets** (4 nombres de 0 à 255), soit 32 bits.

```
Adresse :  192.168.1.25
Masque  :  255.255.255.0     ← notation décimale
           /24               ← notation CIDR (24 bits à 1)
```

| | Octet 1 | Octet 2 | Octet 3 | Octet 4 |
|---|---|---|---|---|
| Adresse | 192 | 168 | 1 | **25** |
| Masque | 255 | 255 | 255 | **0** |
| Rôle | réseau | réseau | réseau | **hôte** |

👉 Ici, le réseau est `192.168.1.0/24` et la machine est la n°25.

## Les adresses à connaître dans un réseau

Pour `192.168.1.0/24` :

| Élément | Valeur | Rôle |
|---|---|---|
| Adresse réseau | `192.168.1.0` | Identifie le réseau, non attribuable |
| Première IP utilisable | `192.168.1.1` | Souvent la passerelle (box, routeur) |
| Dernière IP utilisable | `192.168.1.254` | |
| Broadcast | `192.168.1.255` | « Message à tout le monde », non attribuable |
| Nb d'hôtes | **254** | Formule : 2^(32 − CIDR) − 2 |

## Les masques courants

| CIDR | Masque | Nb d'hôtes | Usage typique |
|---|---|---|---|
| /30 | 255.255.255.252 | 2 | Lien entre 2 routeurs |
| /24 | 255.255.255.0 | 254 | Réseau domestique, petit VLAN |
| /16 | 255.255.0.0 | 65 534 | Grand réseau d'entreprise |
| /8 | 255.0.0.0 | ~16 millions | Très grand réseau |

## Adresses privées vs publiques

Les **adresses privées** ne circulent pas sur Internet (c'est le routeur qui fait la traduction via le NAT) :

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

## À quoi ça sert concrètement

- Savoir si deux machines peuvent se parler **directement** : même réseau = oui, réseaux différents = il faut passer par une passerelle.
- Découper un réseau en sous-réseaux (un par service, un par VLAN…).
- Diagnostiquer : une IP en `169.254.x.x` signifie que la machine **n'a pas obtenu d'adresse du DHCP**.

!!! warning "Pièges fréquents"
    - Oublier que l'adresse réseau et le broadcast **ne sont pas attribuables** (d'où le « − 2 »).
    - Deux machines en `192.168.1.10/24` et `192.168.2.10/24` **ne sont pas** sur le même réseau.
    - Un mauvais masque sur un poste = il croit que ses voisins sont « ailleurs » (ou l'inverse).

## Pour aller plus loin

- 🛠️ [Diagnostiquer un problème de connexion de A à Z](tuto-diagnostic-connexion.md)
- 📋 [Mémo commandes réseau](memo-commandes-reseau.md)

??? quote "Sources"
    - [RFC 791 : IPv4](https://www.rfc-editor.org/rfc/rfc791)
    - [RFC 950 : sous-réseaux](https://www.rfc-editor.org/rfc/rfc950)
    - [RFC 1918 : adresses privées](https://www.rfc-editor.org/rfc/rfc1918)
    - [RFC 4632 : CIDR](https://www.rfc-editor.org/rfc/rfc4632)
