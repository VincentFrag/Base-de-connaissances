---
icon: material/book-open-variant
title: Le modèle OSI et TCP/IP
tags: [réseau, bases, concept]
---

# 🧠 Le modèle OSI et TCP/IP

!!! abstract "En une phrase"
    Le **modèle OSI** découpe une communication réseau en **7 couches**, chacune avec un rôle précis ; **TCP/IP** est sa version simplifiée en 4 couches, celle qu'on utilise réellement.

## L'analogie

Envoyer un colis :

1. Tu écris ta lettre (**le contenu**)
2. Tu la mets dans une enveloppe avec l'adresse du destinataire (**l'adressage**)
3. La poste la met dans un camion qui roule sur la route (**le transport physique**)

À l'arrivée, on fait le chemin inverse : camion → enveloppe → lettre. Chaque étape ajoute (à l'envoi) ou retire (à la réception) **son emballage** : c'est l'**encapsulation**.

## Les 7 couches OSI

| N° | Couche | Rôle | Unité | Exemples |
|---|---|---|---|---|
| 7 | Application | Ce que voit l'utilisateur | Données | HTTP, DNS, SMTP, SSH |
| 6 | Présentation | Format, chiffrement | Données | TLS, encodage |
| 5 | Session | Ouvrir/fermer un dialogue | Données | RPC, NetBIOS |
| 4 | **Transport** | Découper, fiabiliser, **ports** | Segment | **TCP, UDP** |
| 3 | **Réseau** | Adressage logique, **routage** | Paquet | **IP**, ICMP |
| 2 | **Liaison** | Communication locale, **adresses MAC** | Trame | Ethernet, Wi-Fi, VLAN |
| 1 | Physique | Signal électrique / radio / lumière | Bit | Câble RJ45, fibre |

!!! tip "Moyen mnémotechnique (de 1 à 7)"
    **P**our **L**e **R**éseau, **T**out **S**e **P**asse **A**utomatiquement.

## OSI vs TCP/IP

| TCP/IP (4 couches) | Correspond à OSI |
|---|---|
| Application | 7 + 6 + 5 |
| Transport | 4 |
| Internet | 3 |
| Accès réseau | 2 + 1 |

👉 OSI sert à **raisonner et à communiquer** (« c'est un problème de couche 2 »). TCP/IP est ce qui **tourne vraiment** sur les machines.

## Les équipements par couche

| Couche | Équipement | Se base sur |
|---|---|---|
| 1 | Câble, répéteur, hub | Rien (il répète le signal) |
| 2 | **Switch** | Adresse **MAC** |
| 3 | **Routeur**, switch L3 | Adresse **IP** |
| 4 à 7 | **Pare-feu**, proxy, load balancer | Ports, protocoles, contenu |

## TCP ou UDP ?

| | TCP | UDP |
|---|---|---|
| Fiabilité | Vérifie la réception, renvoie si perte | Envoie sans vérifier |
| Vitesse | Plus lent | Plus rapide |
| Usage | Web, mail, SSH, transfert de fichiers | DNS, VoIP, streaming, jeux, DHCP |

## À quoi ça sert concrètement

- **Diagnostiquer couche par couche** : câble branché (1) ? lien actif et bon VLAN (2) ? IP et passerelle (3) ? port ouvert (4) ? application qui répond (7) ?
- **Parler le même langage** que les autres admins : « switch de niveau 3 », « filtrage de couche 7 ».
- Comprendre **ce que filtre** un équipement : un switch ne voit pas les IP, un pare-feu classique ne voit pas le contenu chiffré.

!!! warning "Pièges fréquents"
    - Confondre **adresse MAC** (couche 2, locale) et **adresse IP** (couche 3, routable).
    - Croire qu'un ping qui marche prouve que tout va bien : le ping (ICMP) teste la couche 3, pas le port ni l'application.
    - Les couches 5 et 6 sont rarement distinctes en pratique : pas la peine de s'y attarder.

## Pour aller plus loin

- 🧠 [Adressage IP et masque de sous-réseau](concept-adressage-ip.md)
- 🛠️ [Diagnostiquer un problème de connexion de A à Z](tuto-diagnostic-connexion.md)
