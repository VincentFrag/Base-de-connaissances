---
icon: material/book-open-variant
title: Active Directory (domaine, OU, GPO)
tags: [système, bases, concept, active directory, windows server, gpo]
---

# 🧠 Active Directory : domaine, OU et GPO

!!! abstract "En une phrase"
    **Active Directory (AD)** est l'annuaire de Windows Server : il centralise les **comptes** (utilisateurs, ordinateurs, groupes), **vérifie qui se connecte** et **applique des règles** (GPO) à tout le parc depuis un seul endroit.

## L'analogie

Une entreprise avec un badge d'accès :

- le **domaine** = l'entreprise et son fichier du personnel ;
- le **contrôleur de domaine** = l'accueil qui vérifie ton badge à chaque entrée ;
- les **OU** = les services (Compta, RH, Informatique) qui rangent les fiches ;
- les **GPO** = le règlement intérieur, affiché dans un service ou dans tout le bâtiment.

---

## Les briques à connaître

| Brique | Ce que c'est | Exemple |
|---|---|---|
| **Domaine** | Ensemble de comptes et de machines gérés ensemble, identifié par un nom DNS | `lab.local` |
| **Contrôleur de domaine (DC)** | Serveur qui héberge l'annuaire et authentifie les connexions | `srv-dc01` · `10.0.0.10` |
| **Forêt** | Un ou plusieurs domaines qui se font confiance ; le premier DC crée la forêt | forêt `lab.local` |
| **Objet** | Une entrée de l'annuaire | utilisateur `alice`, ordinateur `PC-01`, groupe `GG_Compta` |
| **OU** (*unité d'organisation*) | Dossier qui range les objets, pour **déléguer** et **cibler les GPO** | `OU=Compta,OU=LAB,DC=lab,DC=local` |
| **Groupe** | Ensemble de comptes, sert à **donner des droits** (partages, dossiers) | `GG_Compta` → accès au partage `\\srv-files01\compta` |
| **GPO** | Lot de paramètres appliqués aux utilisateurs ou ordinateurs d'une OU | fond d'écran, lecteur réseau, mot de passe |

!!! info "OU ou groupe ?"
    **OU** = *où est rangé* l'objet (une seule OU à la fois) → sert aux **GPO** et à la délégation.
    **Groupe** = *à quoi* l'objet a droit (plusieurs groupes possibles) → sert aux **permissions**. Une GPO ne s'applique pas à un groupe « simplement » : on la lie à une OU.

### Comment un poste trouve le domaine

AD repose entièrement sur le **DNS**. Le poste demande au DNS l'enregistrement **SRV** `_ldap._tcp.dc._msdcs.lab.local` pour trouver un contrôleur de domaine, puis s'authentifie en **Kerberos**.

| Service | Port | Rôle |
|---|---|---|
| DNS | 53 | Trouver le DC |
| Kerberos | 88 | Authentifier (tickets) |
| LDAP | 389 | Interroger l'annuaire |
| SMB | 445 | Lire les GPO dans `SYSVOL` |

### Comment s'appliquent les GPO : LSDOU

Les GPO sont appliquées dans cet ordre, **la dernière gagne** en cas de conflit :

| Ordre | Niveau | Exemple |
|---|---|---|
| 1 | **L**ocale (le poste lui-même) | stratégie locale |
| 2 | **S**ite | rarement utilisé en lab |
| 3 | **D**omaine | *Default Domain Policy* (mots de passe) |
| 4 | **OU** (de la plus haute à la plus profonde) | GPO liée à `OU=Compta` |

Les paramètres **Ordinateur** s'appliquent au démarrage, les paramètres **Utilisateur** à l'ouverture de session ; ensuite rafraîchissement toutes les 90 min environ (`gpupdate /force` pour forcer).

---

## À quoi ça sert concrètement

- **Un seul compte** par personne pour se connecter à n'importe quel poste du domaine.
- **Désactiver un départ** en un clic : il perd l'accès partout.
- **Configurer 200 postes** d'un coup : lecteurs réseau, imprimantes, sécurité, mises à jour.
- **Donner des droits par groupe** plutôt que personne par personne (méthode AGDLP).

!!! warning "Pièges fréquents"
    - Un poste client avec un **DNS externe** (`8.8.8.8`, box) **ne trouve pas le domaine** : son seul DNS doit être le DC.
    - Laisser utilisateurs et ordinateurs dans les conteneurs par défaut `Users` et `Computers` : ce **ne sont pas des OU**, on ne peut pas y lier de GPO.
    - Lier une GPO de paramètres **Utilisateur** à une OU qui ne contient **que des ordinateurs** (ou l'inverse) : elle ne fait rien.
    - Nommer le domaine comme un vrai domaine Internet que tu ne possèdes pas. En lab, `lab.local` convient ; en entreprise, on préfère un sous-domaine réel (`ad.entreprise.fr`).
    - Un seul DC = aucune tolérance de panne : s'il tombe, plus personne ne se connecte. En production, **deux DC minimum**.

## Pour aller plus loin

- 🛠️ **[Monter un Active Directory sur une VM Proxmox](tuto-monter-ad-proxmox.md)** : la mise en pratique (domaine, OU, utilisateurs, GPO)
- 🧠 [DHCP et DNS](../reseau/concept-dhcp-dns.md) : enregistrements SRV et rôle du DNS dans AD
- 🛠️ [Installer et configurer DHCP et DNS](../reseau/tuto-dhcp-dns.md) : ajouter le DHCP sur le contrôleur de domaine
