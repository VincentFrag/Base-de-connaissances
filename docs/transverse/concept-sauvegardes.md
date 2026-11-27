---
icon: material/book-open-variant
title: Les sauvegardes (règle 3-2-1)
tags: [transverse, bases, concept, sauvegarde, proxmox]
---

# 🧠 Les sauvegardes et la règle 3-2-1

!!! abstract "En une phrase"
    Une **sauvegarde** est une **copie indépendante** de tes données, rangée **ailleurs** que l'original, que tu as déjà **réussi à restaurer** au moins une fois. Sinon, ce n'est qu'un espoir.

## L'analogie

Tes photos de vacances. Un **snapshot**, c'est un marque-page dans l'album : pratique pour revenir à une page, mais si l'album brûle, le marque-page brûle avec. Une **sauvegarde**, c'est un **double de l'album** rangé chez tes parents. La règle 3-2-1, c'est avoir l'album, un double à la maison sur une autre étagère, et un double chez tes parents.

## La règle 3-2-1

| Chiffre | Règle | Ce qu'on évite | Exemple homelab |
|---|---|---|---|
| **3** | **3 copies** des données (l'original + 2 sauvegardes) | Une sauvegarde corrompue sans plan B | La VM + une sauvegarde locale + une copie externe |
| **2** | Sur **2 supports différents** | Une panne qui touche le même type de disque ou le même serveur | Disque du Proxmox + NAS ou disque USB |
| **1** | **1 copie hors site** | Incendie, vol, dégât des eaux, ransomware | Disque emmené ailleurs, cloud, second site |

!!! tip "Version moderne : 3-2-1-1-0"
    On ajoute souvent **1 copie hors ligne ou immuable** (que personne ne peut modifier, même un ransomware avec les droits admin) et **0 erreur** à la vérification d'une restauration.

## Snapshot ou sauvegarde ?

C'est **la** confusion classique sur un hyperviseur.

| | Snapshot | Sauvegarde |
|---|---|---|
| **C'est quoi** | Un **point de retour** de la VM à un instant | Une **copie complète** des données, exportée |
| **Où c'est stocké** | **Sur le même disque** que la VM | Sur **un autre stockage** (autre disque, NAS, PBS, cloud) |
| **Dépend de l'original** | **Oui** : si le disque de la VM meurt, le snapshot meurt avec | **Non** : elle se restaure sur un autre serveur |
| **Rapidité** | Quasi instantané | Quelques minutes à quelques heures |
| **Durée de vie** | Courte (heures, jours) : il grossit et ralentit la VM | Longue (semaines, mois, selon la rétention) |
| **Usage typique** | « Je fais une mise à jour risquée, je veux pouvoir revenir en arrière » | « Le serveur a brûlé, je dois tout remonter » |

**Retenir :** un snapshot **n'est pas** une sauvegarde. On fait un snapshot *avant* une manip, et des sauvegardes *tout le temps*.

## Les types de sauvegarde

| Type | Ce qui est copié | Avantage | Inconvénient |
|---|---|---|---|
| **Complète** | Tout, à chaque fois | Restauration simple : un seul fichier | Lente, prend beaucoup de place |
| **Incrémentale** | Ce qui a changé **depuis la dernière sauvegarde** (quelle qu'elle soit) | Rapide, peu de place | Pour restaurer, il faut la complète + **toute** la chaîne |
| **Différentielle** | Ce qui a changé **depuis la dernière complète** | Restauration : complète + dernière différentielle | Grossit chaque jour jusqu'à la complète suivante |

!!! note "Et Proxmox dans tout ça ?"
    - **vzdump** (intégré à Proxmox VE) fait des sauvegardes **complètes** à chaque fois.
    - **Proxmox Backup Server (PBS)** fait des sauvegardes qui se restaurent comme des complètes, mais n'envoie et ne stocke que les **morceaux qui ont changé** (*déduplication*). Le meilleur des deux mondes.

## Deux chiffres pour fixer les objectifs

| Sigle | Question | Exemple |
|---|---|---|
| **RPO** (*Recovery Point Objective*) | Combien de données j'accepte de **perdre** ? | Sauvegarde chaque nuit → jusqu'à 24 h de travail perdu |
| **RTO** (*Recovery Time Objective*) | En combien de temps je dois être **reparti** ? | Restaurer la VM en moins de 2 h |

La **rétention**, c'est combien de sauvegardes on garde (ex. 7 quotidiennes, 4 hebdomadaires, 3 mensuelles) : assez pour remonter avant qu'une erreur ou un ransomware ait été remarqué.

## À quoi ça sert concrètement

- Revenir en arrière après une **mise à jour qui casse tout** (snapshot) ou un fichier **supprimé par erreur** (sauvegarde).
- Remonter un service après une **panne de disque** ou la perte complète du serveur.
- Survivre à un **ransomware** grâce à une copie hors ligne qu'il n'a pas pu chiffrer.
- Cloner un environnement : restaurer une sauvegarde sur un autre nœud pour tester.

!!! warning "Pièges fréquents"
    - **Confondre snapshot et sauvegarde** : 30 snapshots sur le disque de la VM = 0 sauvegarde.
    - **Ne jamais tester la restauration** : on découvre le jour J que l'archive est vide ou corrompue.
    - **Sauvegarder sur le même disque** que la VM (ex. stockage `local` du Proxmox) : une seule panne emporte tout.
    - **Un PBS dans une VM du même Proxmox** : pratique pour apprendre, mais si l'hôte meurt, la sauvegarde aussi. Il faut au minimum un disque séparé, idéalement une autre machine.
    - **Garder un snapshot des semaines** : il grossit, ralentit la VM et peut remplir le stockage.
    - **Pas de rétention** : le stockage se remplit et les nouvelles sauvegardes échouent… en silence si personne ne lit les alertes.

## Pour aller plus loin

- 🛠️ **[Sauvegarder et restaurer ses VM avec Proxmox](tuto-proxmox-backup.md)** : la mise en pratique (snapshot, vzdump, tâche planifiée, Proxmox Backup Server)
- 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md)

??? quote "Sources"
    *La règle 3-2-1 n'est pas une norme : c'est une bonne pratique largement reprise, notamment par la CISA.*

    - [CISA : Data Backup Options (PDF)](https://www.cisa.gov/sites/default/files/publications/data_backup_options.pdf)
    - [Proxmox VE : sauvegarde et restauration](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
