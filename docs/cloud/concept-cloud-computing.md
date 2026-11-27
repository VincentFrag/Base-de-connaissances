---
icon: material/book-open-variant
title: Le cloud computing (IaaS, PaaS, SaaS, régions, coûts)
tags: [cloud, bases, concept, iaas, paas, saas, sla, finops, azure]
---

# 🧠 Le cloud computing : modèles, régions et coûts

!!! abstract "En une phrase"
    Le **cloud computing**, c'est louer à la demande des ressources informatiques (serveurs, réseau, stockage, logiciels) chez un fournisseur, au lieu de les acheter et de les héberger soi-même, et ne payer que ce qu'on consomme.

## L'analogie

L'électricité :

- **avant**, chaque usine avait son propre groupe électrogène : il fallait l'acheter, l'entretenir et le dimensionner pour le jour le plus chargé de l'année ;
- **aujourd'hui**, on se branche sur le réseau : on consomme quand on veut, autant qu'on veut, et le compteur tourne ;
- le **cloud** fait la même chose avec les serveurs. Le revers : si on laisse la lumière allumée en partant, la facture arrive quand même.

---

## Les trois modèles de service

Plus on monte, moins on a de choses à gérer soi-même, mais moins on a la main.

| Couche | Sur site | IaaS | PaaS | SaaS |
|---|---|---|---|---|
| Application, données | Toi | Toi | Toi | Fournisseur |
| Environnement d'exécution (PHP, .NET, base) | Toi | Toi | **Fournisseur** | Fournisseur |
| Système d'exploitation | Toi | Toi | Fournisseur | Fournisseur |
| Virtualisation, serveurs, réseau, bâtiment | Toi | **Fournisseur** | Fournisseur | Fournisseur |
| **Exemple Azure** | Proxmox dans ta salle | Une VM Linux | App Service, Azure SQL | Microsoft 365 |

!!! info "La responsabilité partagée"
    Même dans le cloud, **la sécurité de ce que tu gères reste à ta charge**. En IaaS, c'est toi qui patches l'OS de la VM et qui fermes les ports ; le fournisseur, lui, garantit le matériel et le datacenter.

## Les modèles de déploiement

| Modèle | Principe | Exemple |
|---|---|---|
| **Cloud public** | Infrastructure partagée entre des milliers de clients, isolés logiquement | Azure, AWS, Google Cloud, OVHcloud Public Cloud |
| **Cloud privé** | Infrastructure dédiée à une seule organisation, sur site ou hébergée | Un cluster Proxmox ou VMware de l'entreprise |
| **Hybride** | Les deux, reliés par un VPN ou une liaison dédiée | AD sur site, synchronisé avec Entra ID |
| **Multicloud** | Plusieurs fournisseurs publics à la fois | Sauvegardes chez un second fournisseur |

## Régions et zones

| Notion | Ce que c'est | Ce que ça protège |
|---|---|---|
| **Région** | Un ensemble de datacenters dans une zone géographique (ex. *France Central*, à Paris) | Choisir une région proche des utilisateurs, et dans le bon pays pour les données |
| **Zone de disponibilité** | Un ou plusieurs datacenters indépendants (courant, refroidissement, réseau) dans une région, numérotés 1, 2, 3 | La panne d'un datacenter entier |
| **Paire de régions** | Deux régions d'un même pays ou continent, reliées pour la reprise d'activité | La perte d'une région entière |

## Disponibilité : lire un SLA

Un **SLA** (*Service Level Agreement*) est l'engagement de disponibilité d'un service, exprimé en pourcentage du mois.

| SLA | Arrêt maximal par mois (30 jours) |
|---|---|
| 99 % | 7 h 12 min |
| 99,9 % | 43 min |
| 99,95 % | 22 min |
| 99,99 % | 4 min 19 s |

- **Composants en série** (il faut les deux) : on **multiplie**. Une VM à 99,9 % derrière une base à 99,99 % donne 0,999 × 0,9999 ≈ **99,89 %**.
- **Composants en parallèle** (un seul suffit) : on multiplie les **indisponibilités**. Deux VM à 99,9 % : 1 − (0,001 × 0,001) = **99,9999 %** en théorie.
- Si le fournisseur ne tient pas son SLA, tu reçois un **avoir** sur ta facture, pas le remboursement de ta perte d'activité.

## Le modèle de coût

| Ce qu'on paie | Comment | Exemple |
|---|---|---|
| **Calcul** | À la seconde ou à l'heure, tant que la VM est **allouée** | Une petite VM Linux, quelques centimes par heure |
| **Stockage** | Au Go par mois, même si la VM est éteinte | Le disque d'une VM désallouée |
| **Réseau sortant** (*egress*) | Au Go sortant du cloud ; l'entrant est en général gratuit | Télécharger ses sauvegardes chez soi |
| **Services managés** | À l'heure d'existence, qu'on s'en serve ou non | Passerelle VPN, bastion, adresse IP publique |

La démarche **FinOps** consiste à rendre ces coûts visibles et maîtrisés : budgets avec alertes, **tags** sur chaque ressource pour savoir qui consomme quoi, arrêt automatique des VM la nuit, et suppression de ce qui ne sert plus.

---

## À quoi ça sert concrètement

- **Monter un environnement de test en 5 minutes**, et le détruire le soir même.
- **Absorber un pic de charge** (soldes, inscriptions) sans acheter de serveurs qui dormiront le reste de l'année.
- **Héberger une copie hors site** de ses sauvegardes (la « copie 1 » de la règle 3-2-1).
- **Se former** sur des technologies d'entreprise (Active Directory, équilibrage de charge, VPN) sans matériel.

!!! warning "Pièges fréquents"
    - **Éteindre une VM depuis son système ne suffit pas** : elle reste allouée et facturée. Il faut la **désallouer** (bouton *Arrêter* du portail, ou `az vm deallocate`).
    - Oublier les ressources qui coûtent même sans trafic : IP publiques, passerelles VPN, bastions, disques orphelins après suppression d'une VM.
    - Confondre **disponibilité** (le service répond) et **durabilité** (la donnée n'est pas perdue) : un stockage peut être indisponible une heure sans rien perdre.
    - Croire que « c'est dans le cloud, donc c'est sauvegardé » : une suppression ou un chiffrement par rançongiciel est répliqué partout. La sauvegarde reste ton travail.
    - Ignorer le coût de **sortie** des données : c'est souvent lui qui rend un changement de fournisseur cher.

## Pour aller plus loin

- 🛠️ **[Démarrer sur Azure sans exploser son crédit](tuto-demarrer-azure.md)** : la mise en pratique (compte étudiant, budget, tags, Azure CLI)
- 🧠 [Virtualisation et hyperviseurs](../systeme/concept-virtualisation.md) : la brique sur laquelle repose tout l'IaaS
- 🧠 [Sauvegardes](../transverse/concept-sauvegardes.md) : RPO, RTO et règle 3-2-1, aussi valables dans le cloud
