---
icon: material/book-open-variant
title: Identités et droits dans Azure (Entra ID, RBAC, identité managée)
tags: [cloud, intermédiaire, concept, azure, entra id, rbac, identité managée, sécurité]
---

# 🧠 Identités et droits dans Azure : Entra ID, RBAC et identités managées

!!! abstract "En une phrase"
    Dans Azure, **qui** tu es est géré par **Entra ID** (l'annuaire), et **ce que tu as le droit de faire** est donné par le **RBAC** : un **rôle** attribué à un **principal** sur une **portée** ; les programmes et les VM, eux, s'authentifient avec une **identité managée**, sans mot de passe à cacher.

## L'analogie

Un hôtel :

- **Entra ID** = la réception, qui vérifie ta pièce d'identité ;
- le **rôle** = le type de badge : femme de ménage, technicien, directeur ;
- la **portée** = les portes que ce badge ouvre : un étage, un couloir, tout l'hôtel ;
- l'**identité managée** = le badge fixé sur le chariot du room-service : il ouvre l'ascenseur de service, sans que personne n'ait à prêter le sien.

---

## Entra ID et Active Directory

| | **Active Directory** (sur site) | **Microsoft Entra ID** (cloud) |
|---|---|---|
| Rôle | Annuaire des comptes, groupes et ordinateurs d'un domaine | Annuaire des comptes pour le cloud et les applications SaaS |
| Protocoles | Kerberos, LDAP, GPO | OAuth 2.0, OpenID Connect, SAML (le web) |
| Structure | Domaine, OU, GPO | Tenant, utilisateurs, groupes, applications |
| On s'y connecte depuis | Un poste joint au domaine | N'importe où, avec MFA |
| Lien entre les deux | Entra Connect synchronise les comptes AD vers Entra ID (identité **hybride**) | |

Un **tenant** est une instance d'Entra ID (celle de ton école, de ton entreprise). Un ou plusieurs **abonnements** Azure lui font confiance pour savoir qui se connecte.

## Le RBAC : trois questions

Une **attribution de rôle** répond à : **qui ?** **quoi ?** **où ?**

| Question | Terme | Exemples |
|---|---|---|
| Qui ? | **Principal** | Un utilisateur, un groupe, une identité managée, une application |
| Quoi ? | **Rôle** (liste d'actions permises) | Lecteur, Contributeur, Propriétaire, Contributeur de machines virtuelles |
| Où ? | **Portée** | Groupe d'administration → abonnement → groupe de ressources → ressource |

Les droits **descendent** : un rôle donné sur un groupe de ressources vaut pour toutes les ressources de ce groupe.

### Les rôles à connaître

| Rôle | Ce qu'il permet | Ce qu'il ne permet pas |
|---|---|---|
| **Lecteur** (*Reader*) | Tout voir | Rien modifier |
| **Contributeur** (*Contributor*) | Tout créer, modifier, supprimer | Donner des droits à d'autres |
| **Propriétaire** (*Owner*) | Tout, y compris attribuer des rôles | — |
| **Contributeur de machines virtuelles** | Gérer les VM | Toucher au réseau ou au stockage autour |
| **Contributeur de données Blob du stockage** | Lire et écrire le **contenu** des blobs | Gérer le compte de stockage lui-même |

!!! info "Plan de contrôle et plan de données"
    **Gérer** une ressource (créer un compte de stockage) et **lire son contenu** (ouvrir un fichier dedans) sont deux choses différentes. Être Propriétaire de l'abonnement ne donne pas automatiquement accès aux données d'un blob : il faut un rôle « … de données ».

## Les identités managées

Un script sur une VM qui doit lire un coffre de secrets ou écrire dans un stockage a besoin de s'authentifier. Plutôt que de lui donner un mot de passe (qui finira dans un fichier ou dans Git), on lui attache une **identité managée** : Azure fournit et renouvelle tout seul les jetons.

| Type | Durée de vie | Quand l'utiliser |
|---|---|---|
| **Attribuée par le système** | Créée et supprimée **avec** la ressource | Une VM, une seule identité |
| **Attribuée par l'utilisateur** | Ressource indépendante, réutilisable | Plusieurs VM qui partagent les mêmes droits |

## Au-delà du RBAC

- **Rôles personnalisés** : quand aucun rôle intégré ne colle, on écrit la liste exacte des actions (ex. démarrer et arrêter des VM, sans pouvoir les supprimer).
- **ABAC** : des conditions sur des attributs s'ajoutent au rôle (ex. lire seulement les blobs étiquetés `projet=site`).
- **Accès conditionnel** (Entra ID) : exiger la MFA, un poste de l'entreprise ou un pays donné pour se connecter.

---

## À quoi ça sert concrètement

- Donner à un **stagiaire** le rôle Lecteur sur un seul groupe de ressources : il voit tout, ne casse rien.
- Laisser une **équipe projet** Contributeur sur **son** groupe, sans accès aux autres.
- Permettre à un **script de sauvegarde** sur une VM d'écrire dans un stockage, sans aucun secret stocké sur la VM.
- **Auditer** : le journal d'activité indique qui a fait quoi, avec quel rôle.

!!! warning "Pièges fréquents"
    - Donner **Propriétaire** ou **Contributeur** sur tout l'abonnement « pour que ça marche » : c'est l'inverse du moindre privilège. Viser le plus petit rôle, sur la plus petite portée.
    - Attribuer des rôles **à des personnes** plutôt qu'à des **groupes** : impossible à maintenir quand l'équipe change.
    - Croire que le RBAC est immédiat : une nouvelle attribution peut mettre **quelques minutes** à s'appliquer.
    - Laisser traîner des **attributions orphelines** (« Identité introuvable ») après la suppression d'une VM ou d'un compte.
    - Mettre une clé de compte de stockage dans un script alors qu'une identité managée ferait l'affaire.

## Pour aller plus loin

- 🛠️ **[Prouver le RBAC avec une identité managée](tuto-rbac-identite-managee.md)** : la mise en pratique (attribution, test de refus, rôle personnalisé)
- 🧠 [Active Directory](../systeme/concept-active-directory.md) : l'annuaire sur site, pour comparer
- 🧠 [Le stockage dans Azure](concept-stockage-azure.md) : le plan de données en pratique

??? quote "Sources"
    - [Microsoft Learn : qu'est-ce que Microsoft Entra ID ?](https://learn.microsoft.com/fr-fr/entra/fundamentals/whatis)
    - [Microsoft Learn : contrôle d'accès en fonction du rôle (RBAC)](https://learn.microsoft.com/fr-fr/azure/role-based-access-control/overview)
    - [Microsoft Learn : identités managées](https://learn.microsoft.com/fr-fr/entra/identity/managed-identities-azure-resources/overview)
