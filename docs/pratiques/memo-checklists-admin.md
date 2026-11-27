---
icon: material/clipboard-text-outline
title: Check-lists de l'admin
tags: [bonnes pratiques, mémo, check-list, méthode]
---

# 📋 Check-lists de l'admin

!!! abstract "Usage"
    Des listes à dérouler pour ne rien oublier dans les situations qui reviennent. Les cases se cochent dans la page (elles se remettent à zéro au rechargement) : copie la liste dans ton ticket ou ton journal pour garder une trace.
    Le pourquoi de chaque réflexe : 🧠 [Les réflexes de l'admin](concept-reflexes-admin.md).

## Avant toute intervention

- [ ] Je sais **quoi** je change, **pourquoi**, et sur **quelle machine** (bon nom d'hôte, bonne VM)
- [ ] Le changement est demandé ou validé (ticket, responsable)
- [ ] Les utilisateurs concernés sont prévenus si le service sera coupé
- [ ] Le créneau est adapté (hors heures d'utilisation, pas juste avant de partir)
- [ ] **Snapshot** de la VM pris, ou sauvegarde récente vérifiée
- [ ] Copie datée des fichiers de conf que je vais toucher
- [ ] État de départ noté (service actif, ports, version)
- [ ] Le **retour arrière** est écrit noir sur blanc
- [ ] J'ai un accès de secours (console Proxmox, deuxième session)

## Après une intervention

- [ ] La conf a été validée avant application (`-t`, `configtest`)
- [ ] Le service fonctionne (même test qu'avant)
- [ ] Les services voisins fonctionnent (connexion des utilisateurs, site web, supervision verte)
- [ ] Le journal des changements est rempli
- [ ] La fiche serveur est à jour (IP, rôle, accès, version)
- [ ] Les utilisateurs sont prévenus de la fin
- [ ] Date de suppression du snapshot notée

👉 Le déroulé complet : 🛠️ [Faire une intervention propre de A à Z](tuto-intervention-propre.md)

## Mise en service d'un nouveau serveur

- [ ] Nom selon la convention (`srv-<rôle>01`) et IP fixe libre réservée dans le plan d'adressage
- [ ] Système à jour (`apt full-upgrade` / Windows Update)
- [ ] Fuseau horaire et synchronisation NTP corrects (`timedatectl` / `w32tm /query /status`)
- [ ] Enregistrement DNS créé (et vérifié avec `nslookup` / `dig`)
- [ ] Compte d'admin nominatif, pas de connexion `root` directe, SSH par clé
- [ ] Pare-feu activé, seuls les ports utiles ouverts
- [ ] Services inutiles désactivés
- [ ] Mises à jour automatiques de sécurité ou créneau de maintenance défini
- [ ] Ajouté à la **sauvegarde** et une restauration testée
- [ ] Ajouté à la **supervision** (ping, disque, service principal)
- [ ] Fiche serveur créée et schéma réseau mis à jour

Liens utiles : 🛠️ [Préparer un clone de VM](../systeme/tuto-preparer-clone-vm.md) · 🛠️ [Sauvegarder et restaurer ses VM](../transverse/tuto-proxmox-backup.md) · 🧠 [Comprendre la supervision](../transverse/concept-supervision.md)

## Arrivée d'un utilisateur

- [ ] Compte créé selon la convention de nommage, mot de passe provisoire à changer à la première connexion
- [ ] Placé dans la bonne **OU** et les bons **groupes** (jamais de droits donnés directement au compte)
- [ ] Droits limités à ce dont il a besoin (**moindre privilège**)
- [ ] Lecteurs réseau, imprimantes, messagerie fonctionnels
- [ ] Poste prêt (nommé `PC-xx`, à jour, dans le domaine)
- [ ] Demande et date d'arrivée tracées

## Départ d'un utilisateur

- [ ] Compte **désactivé** le jour du départ (pas supprimé tout de suite)
- [ ] Retiré des groupes, sessions et accès VPN révoqués
- [ ] Mots de passe partagés qu'il connaissait changés (Wi-Fi, comptes de service)
- [ ] Données transférées ou archivées (dossier personnel, boîte mail)
- [ ] Matériel récupéré
- [ ] Suppression définitive planifiée (ex. dans 30 à 90 jours)

## Réagir à un incident

- [ ] **Ne pas paniquer, ne rien redémarrer au hasard** : un redémarrage efface des indices
- [ ] Qui est touché ? un utilisateur, un service, tout le monde ?
- [ ] Depuis quand ? qu'est-ce qui a changé juste avant ? (journal des changements)
- [ ] Prévenir les utilisateurs et le responsable
- [ ] Diagnostiquer couche par couche : réseau → service → application → 🛠️ [Diagnostiquer une connexion](../reseau/tuto-diagnostic-connexion.md)
- [ ] Lire les journaux (`journalctl -b -p err`, Observateur d'événements)
- [ ] Noter chaque action et son heure au fur et à mesure
- [ ] Rétablir le service (contournement), puis chercher la vraie cause
- [ ] Rédiger un court retour d'expérience : cause, correction, comment éviter que ça recommence

!!! danger "Si tu soupçonnes une attaque"
    Isoler la machine du réseau (débrancher la carte réseau virtuelle dans Proxmox) **sans l'éteindre**, et prévenir immédiatement le responsable sécurité. Ne pas effacer les traces.

## Revue hebdomadaire

- [ ] Les sauvegardes de la semaine ont réussi (et une restauration test ce mois-ci)
- [ ] Aucun snapshot oublié (`qm listsnapshot <id>`)
- [ ] Espace disque sous 80 % partout (`df -h`, `Get-Volume`)
- [ ] Mises à jour en attente appliquées ou planifiées
- [ ] Alertes de supervision traitées, aucune ignorée « par habitude »
- [ ] Certificats qui expirent bientôt
- [ ] Comptes inactifs ou verrouillés vérifiés
- [ ] Journaux d'erreurs parcourus (`journalctl -p err --since "7 days ago"`)
- [ ] Documentation à jour des changements de la semaine

??? quote "Sources"
    *Pas de documentation officielle unique sur ce sujet : bonnes pratiques issues de l'expérience, recoupées avec le guide de l'ANSSI.*

    - [ANSSI : guide d'hygiène informatique](https://cyber.gouv.fr/publications/guide-dhygiene-informatique)
