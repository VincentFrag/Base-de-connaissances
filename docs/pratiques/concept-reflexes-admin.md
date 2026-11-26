---
icon: material/book-open-variant
title: Les réflexes de l'admin
tags: [bonnes pratiques, bases, concept, méthode, sécurité]
---

# 🧠 Les réflexes de l'admin

!!! abstract "En une phrase"
    Un bon admin n'est pas celui qui ne casse jamais rien : c'est celui qui **sait toujours revenir en arrière**, qui **prouve que ça marche** et qui **laisse une trace** de ce qu'il a fait.

**Niveau** 🟢 Bases

## L'analogie

Un chirurgien ne commence pas à opérer « pour voir ». Il vérifie le dossier du patient, compte ses instruments avant et après, note tout dans le compte rendu, et l'équipe sait quoi faire si ça tourne mal. L'admin fait pareil avec un serveur : **préparer, agir, vérifier, documenter**.

## Le cycle d'une intervention

Toute modification, même « juste une ligne », suit le même cycle :

| Moment | Réflexe | Exemple concret |
|---|---|---|
| **Avant** | Savoir ce qu'on change et pourquoi | « Je désactive la connexion `root` en SSH sur `srv-web01` » |
| | Prévoir le retour arrière | Snapshot de la VM, copie du fichier de conf |
| | Noter l'état de départ | Le service tourne ? quels ports écoutent ? |
| | Choisir le bon moment | Pas un vendredi à 17 h, pas en pleine journée de travail des utilisateurs |
| **Pendant** | Une seule modification à la fois | Si ça casse, on sait tout de suite pourquoi |
| | Valider avant d'appliquer | `sshd -t`, `nginx -t`, `apachectl configtest` |
| | Garder une porte de secours | Une session SSH ouverte, la console Proxmox |
| **Après** | Vérifier que ça marche | Même test qu'avant : le résultat attendu est-il là ? |
| | Vérifier qu'on n'a rien cassé à côté | Les utilisateurs se connectent toujours ? la supervision est verte ? |
| | Documenter | Une ligne dans le journal des changements |
| | Faire le ménage | Supprimer le snapshot quelques jours plus tard |

## Les règles d'or

| Règle | Ce que ça veut dire | Pourquoi |
|---|---|---|
| **Toujours pouvoir revenir en arrière** | Snapshot, sauvegarde, copie `.bak` avant de toucher | Une erreur devient un incident de 2 minutes au lieu d'une nuit blanche |
| **Moindre privilège** | Chacun a juste les droits dont il a besoin, pas plus | Un compte piraté ou une fausse manip fait moins de dégâts |
| **Pas de compte partagé** | Un compte nominatif par personne (`alice`, `bruno`), `sudo` pour l'admin | On sait qui a fait quoi ; on désactive une personne sans changer le mot de passe de tout le monde |
| **Pas de travail en `root` / Administrateur au quotidien** | Compte normal + élévation ponctuelle (`sudo`, « Exécuter en tant qu'administrateur ») | Une commande tapée trop vite ne détruit pas le système |
| **Si ce n'est pas documenté, ça n'existe pas** | Fiche par serveur, journal des changements, schéma réseau à jour | Dans 6 mois, toi aussi tu auras oublié |
| **Nommage cohérent** | `srv-<rôle>01`, `GG_Compta`, `PC-01` | On comprend une machine ou un groupe rien qu'à son nom |
| **Une sauvegarde non testée n'est pas une sauvegarde** | Restaurer régulièrement pour de vrai | C'est le jour de la panne qu'on découvre qu'elle était vide |
| **Automatiser ce qu'on fait deux fois** | Script, tâche planifiée | Moins d'oublis, même résultat à chaque fois |
| **Mettre à jour régulièrement** | Correctifs de sécurité appliqués et planifiés | La plupart des attaques exploitent des failles déjà corrigées |
| **Surveiller plutôt que subir** | Supervision et alertes | Être prévenu avant que les utilisateurs appellent |

## À quoi ça sert concrètement

- **En entreprise** : ces réflexes sont formalisés dans la **gestion des changements** (on demande, on valide, on planifie, on trace). Les tickets et le journal des changements sont souvent demandés lors d'un audit.
- **En stage ou en alternance** : c'est ce qui fait la différence entre « le stagiaire qui a cassé le serveur » et « le stagiaire qui a remis le serveur en route en 2 minutes parce qu'il avait fait un snapshot ».
- **Sur le homelab** : on prend l'habitude sans risque, et chaque TP devient plus facile à refaire.

!!! warning "Pièges fréquents"
    - **« C'est juste une petite modif »** : ce sont les petites modifs non préparées qui provoquent les grosses pannes.
    - **Se couper l'accès** : changer la conf SSH ou le pare-feu et fermer sa seule session. Toujours garder une session ouverte et tester depuis une **deuxième** fenêtre.
    - **Garder les snapshots indéfiniment** : un snapshot n'est pas une sauvegarde et il ralentit la VM et remplit le stockage.
    - **Tout changer d'un coup** : trois modifications en même temps, ça casse, et on ne sait pas laquelle est en cause.
    - **Documenter « plus tard »** : plus tard n'arrive jamais. Une ligne tout de suite vaut mieux qu'une page jamais écrite.

## Pour aller plus loin

- 🛠️ [Faire une intervention propre de A à Z](tuto-intervention-propre.md) : tous ces réflexes appliqués à un vrai cas
- 📋 [Check-lists de l'admin](memo-checklists-admin.md)
- 📋 [Nommage et documentation](memo-nommage-documentation.md)
- 📋 [Commandes du quotidien](memo-commandes-quotidien.md)
- 📋 [Mémo Bash et PowerShell](../transverse/memo-bash-powershell.md) : automatiser ce qu'on fait deux fois
- 🧠 [Comprendre les sauvegardes et la règle 3-2-1](../transverse/concept-sauvegardes.md)
- 🧠 [Comptes, droits et permissions Linux](../systeme/concept-permissions-linux.md)
