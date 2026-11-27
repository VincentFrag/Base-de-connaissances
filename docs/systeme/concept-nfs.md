---
icon: material/book-open-variant
title: Le partage de fichiers NFS
tags: [système, bases, concept, nfs, linux, partage, stockage]
---

# 🧠 Le partage de fichiers NFS

!!! abstract "En une phrase"
    **NFS** (*Network File System*) permet à un serveur Linux de **partager un dossier** sur le réseau, que d'autres machines **montent** ensuite comme s'il s'agissait d'un dossier local.

## L'analogie

C'est une **armoire commune dans un couloir** : un seul exemplaire des dossiers, et chaque bureau a une porte qui donne dessus. Quand quelqu'un modifie un document, tous les autres voient tout de suite la nouvelle version, sans copie à faire.

## Comment ça marche

```
srv-files01 10.0.0.20                    srv-web01 10.0.0.31
/srv/web  ── export NFS (port 2049) ──▶  monté sur /var/www/html
                                   └──▶  srv-web02 10.0.0.32
                                         monté sur /var/www/html
```

| Élément | Côté | Rôle |
|---|---|---|
| `nfs-kernel-server` | Serveur | Le service NFS (paquet Debian) |
| `/etc/exports` | Serveur | La liste des dossiers partagés, **qui** y a droit et **comment** |
| `exportfs -ra` | Serveur | Recharge `/etc/exports` sans redémarrer |
| `nfs-common` | Client | Les outils pour monter un partage NFS |
| `showmount -e <serveur>` | Client | Liste les partages proposés par un serveur |
| `/etc/fstab` | Client | Monte le partage automatiquement au démarrage |

## Les options d'export à connaître

Une ligne de `/etc/exports` ressemble à : `/srv/web 10.0.0.31(ro,sync,no_subtree_check)`

| Option | Signification |
|---|---|
| `ro` / `rw` | Lecture seule / lecture-écriture |
| `sync` | Le serveur confirme une écriture seulement une fois sur disque (plus sûr) |
| `no_subtree_check` | Désactive une vérification coûteuse et source d'erreurs ; recommandé |
| `root_squash` (défaut) | Le root du client devient « personne » (`nobody`) sur le partage : il n'a pas les droits de root |
| `no_root_squash` | Le root du client reste root sur le partage (à éviter) |

!!! info "Les droits se jouent sur les numéros, pas sur les noms"
    NFS transmet des **UID/GID** (numéros d'utilisateur), pas des noms. Sur Debian, `www-data` a l'UID **33** sur toutes les machines : en donnant les fichiers à `33:33` sur le serveur, Apache et Nginx peuvent les lire sur les clients.

## À quoi ça sert concrètement

- Servir **le même site web** depuis plusieurs serveurs derrière un répartiteur de charge.
- Centraliser les dossiers personnels ou les données partagées d'un réseau Linux.
- Stocker les disques de VM ou les sauvegardes d'un hyperviseur (Proxmox sait utiliser un stockage NFS).

!!! warning "Pièges fréquents"
    - **Export limité à des IP** : une machine absente de `/etc/exports` reçoit `access denied by server`. Vérifier les IP et relancer `exportfs -ra`.
    - **Oublier `_netdev` dans `/etc/fstab`** : le système tente de monter le partage avant que le réseau soit prêt, et le démarrage peut bloquer.
    - **Mauvais propriétaire** : les fichiers appartiennent à root sur le serveur et le service web du client ne peut pas les lire (erreur 403).
    - **Monter par-dessus un dossier non vide** : l'ancien contenu est **caché** (pas effacé) tant que le partage est monté, ce qui prête à confusion.
    - **Pas d'authentification par défaut** : NFS fait confiance à l'adresse IP du client. À réserver à un réseau interne de confiance.

## Pour aller plus loin

- 🛠️ **[Partager un dossier en NFS et le monter sur des serveurs web](tuto-partage-nfs.md)** : la mise en pratique
- 🧠 [La répartition de charge](concept-repartition-charge.md)
- 🧪 [TP : Load balancer HAProxy et pages web sur NFS](tp-load-balancer-haproxy-nfs.md)

??? quote "Sources"
    - [Page de manuel `exports(5)`](https://manpages.debian.org/stable/nfs-kernel-server/exports.5.en.html) : options `ro`, `rw`, `sync`, `no_subtree_check`, `root_squash`
    - [Wiki Debian : NFSServerSetup](https://wiki.debian.org/NFSServerSetup)
