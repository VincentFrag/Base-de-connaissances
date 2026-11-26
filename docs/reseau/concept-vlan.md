---
title: Les VLAN
tags: [réseau, bases, concept, vlan, switch]
---

# 🧠 Les VLAN

!!! abstract "En une phrase"
    Un **VLAN** (*Virtual LAN*) découpe un même switch physique en **plusieurs réseaux logiques isolés**, comme s'il y avait plusieurs switchs séparés.

## L'analogie

Un immeuble de bureaux avec un seul couloir (le switch). Les VLAN, ce sont des **badges de couleur** : les badges bleus (Compta) n'ouvrent que les portes bleues, les badges verts (Invités) que les portes vertes. Tout le monde partage le même bâtiment, mais personne n'entre chez les autres sans passer par l'**accueil** (le routeur).

## Pourquoi faire des VLAN

| Objectif | Exemple |
|---|---|
| **Sécurité** | Le Wi-Fi invités ne voit pas les serveurs |
| **Organisation** | Un VLAN par service, par étage ou par type d'équipement |
| **Performance** | Limiter le broadcast : il reste dans son VLAN |
| **Économie** | Un seul switch au lieu de plusieurs |

## Un VLAN = un sous-réseau

La règle de base : **chaque VLAN a son propre sous-réseau IP**.

| VLAN | Nom | Sous-réseau | Passerelle |
|---|---|---|---|
| 10 | Utilisateurs | `192.168.10.0/24` | `192.168.10.254` |
| 20 | Serveurs | `192.168.20.0/24` | `192.168.20.254` |
| 30 | Téléphonie | `192.168.30.0/24` | `192.168.30.254` |
| 99 | Administration | `192.168.99.0/24` | `192.168.99.254` |

!!! tip "Astuce de nommage"
    Faire correspondre le numéro de VLAN et le 3ᵉ octet (VLAN **20** → `192.168.**20**.0`) rend le réseau lisible au premier coup d'œil.

## Ports access et ports trunk

| Type de port | Transporte | Branché à | Étiquette (tag) |
|---|---|---|---|
| **Access** | **Un seul** VLAN | PC, imprimante, serveur | Non : le poste ne sait même pas qu'il est dans un VLAN |
| **Trunk** | **Plusieurs** VLAN | Autre switch, routeur, pare-feu, hyperviseur | Oui : norme **802.1Q** |

Sur un trunk, chaque trame porte une **étiquette 802.1Q** avec le numéro de son VLAN, pour que l'équipement d'en face sache à quel VLAN elle appartient.

```
[PC compta]──access VLAN 10──┐
                             [Switch]══trunk (VLAN 10,20,30)══[Pare-feu / routeur]
[Serveur]───access VLAN 20───┘
```

**VLAN natif** : sur un trunk, le VLAN dont les trames circulent **sans étiquette** (VLAN 1 par défaut). Il doit être identique des deux côtés du trunk.

## Faire communiquer deux VLAN

Par défaut, deux VLAN **ne se parlent pas** : c'est le but. Pour les relier, il faut un équipement de **couche 3** :

- un **routeur** ou un **pare-feu** (ex. pfSense) avec une sous-interface par VLAN, c'est le *router on a stick* ;
- un **switch de niveau 3**, avec une interface virtuelle (SVI) par VLAN.

Avec un pare-feu, on choisit précisément **qui a le droit de parler à qui** (ex. Utilisateurs → Serveurs sur le port 443 uniquement).

## À quoi ça sert concrètement

- Isoler le Wi-Fi invités, les caméras ou les équipements IoT du reste du réseau.
- Donner un VLAN à la téléphonie IP pour lui garantir de la qualité de service.
- Faire passer plusieurs réseaux vers un hyperviseur (VMware, Proxmox) sur un seul câble, en trunk.

!!! warning "Pièges fréquents"
    - **Poste sans IP après un changement de port** : le port n'est pas dans le bon VLAN, ou le VLAN n'a pas de relais DHCP.
    - **VLAN absent du trunk** : le VLAN existe sur les deux switchs, mais n'est pas autorisé sur le lien entre eux.
    - **VLAN natif différent** de chaque côté d'un trunk : le trafic se retrouve dans le mauvais VLAN.
    - Laisser des postes dans le **VLAN 1** par défaut : mauvaise pratique de sécurité, mieux vaut le laisser vide.

## Pour aller plus loin

- 🛠️ **[Créer et configurer des VLAN](tuto-creer-vlan.md)** : la mise en pratique (Cisco, pfSense, Linux)
- 🧠 [Adressage IP et masque de sous-réseau](concept-adressage-ip.md)
- 🧠 [DHCP et DNS](concept-dhcp-dns.md) (le relais DHCP entre VLAN)
- 🧠 [Le modèle OSI et TCP/IP](concept-modele-osi-tcpip.md) (VLAN = couche 2, routage = couche 3)
