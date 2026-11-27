---
icon: material/wrench-outline
title: Partager un dossier en NFS et le monter sur des serveurs web
tags: [système, tuto, nfs, linux, debian, apache, nginx, partage]
---

# 🛠️ Partager un dossier en NFS et le monter sur des serveurs web

!!! abstract "Objectif"
    Stocker les pages du site **une seule fois** sur un serveur de fichiers (`srv-files01`), les partager en **NFS**, et les faire servir par les deux serveurs web placés derrière HAProxy. Une modification sur `srv-files01` est alors visible **immédiatement** quel que soit le backend qui répond.

**Niveau :** 🟡 Intermédiaire · **Durée :** 20 min

## Prérequis

- Comprendre NFS → 🧠 [Le partage de fichiers NFS](concept-nfs.md)
- Le load balancer en place → 🛠️ [Répartir la charge avec HAProxy](tuto-haproxy-repartition-charge.md)
- Une VM `srv-files01` préparée → 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)

## Le lab

| VM | IP | Rôle NFS |
|---|---|---|
| `srv-files01` | `10.0.0.20` | **Serveur** : exporte `/srv/web` |
| `srv-web01` | `10.0.0.31` | **Client** : monte le partage sur `/var/www/html` |
| `srv-web02` | `10.0.0.32` | **Client** : monte le partage sur `/var/www/html` |
| `srv-lb01` | `10.0.0.30` | Sert aux tests (HAProxy) |

---

## Étape 1 — Créer le partage sur srv-files01

```bash
apt install -y nfs-kernel-server

# Le dossier et une page de test
mkdir -p /srv/web
echo "<h1>Site servi depuis srv-files01 (NFS)</h1>" > /srv/web/index.html

# Propriétaire www-data (UID/GID 33 sur Debian) et lecture pour tous
chown -R 33:33 /srv/web && chmod -R 755 /srv/web

# Export en lecture seule, réservé aux deux serveurs web
echo "/srv/web 10.0.0.31(ro,sync,no_subtree_check) 10.0.0.32(ro,sync,no_subtree_check)" >> /etc/exports

# Appliquer et contrôler
exportfs -ra && exportfs -v
```

`exportfs -v` doit lister `/srv/web` deux fois, une par client, avec les options `ro,sync,…`.

!!! tip "Pourquoi en lecture seule (`ro`) ?"
    Les serveurs web n'ont qu'à **lire** les pages. Si l'un d'eux est compromis, il ne pourra pas modifier le site. On publie le contenu **sur `srv-files01`** uniquement.

## Étape 2 — Monter le partage sur chaque serveur web

À faire **sur `srv-web01` puis sur `srv-web02`** :

```bash
apt install -y nfs-common

# Le serveur propose-t-il bien le partage ?
showmount -e 10.0.0.20

# Vider l'ancienne page locale (elle serait masquée de toute façon)
rm -rf /var/www/html/*

# Montage permanent ; _netdev = attendre le réseau au démarrage
echo "10.0.0.20:/srv/web /var/www/html nfs defaults,_netdev 0 0" >> /etc/fstab
systemctl daemon-reload && mount -a

# Le partage est-il monté ?
df -h | grep 10.0.0.20
```

`df -h` doit afficher une ligne `10.0.0.20:/srv/web … /var/www/html`.

## Étape 3 — Vérifier

1. Les deux backends servent la même page, depuis `srv-lb01` :
    ```bash
    for i in 1 2 3 4; do curl -s http://10.0.0.30; done
    ```
    Les 4 réponses : `<h1>Site servi depuis srv-files01 (NFS)</h1>`.

2. La répartition continue : le contenu est identique, mais l'en-tête `X-Backend` montre toujours l'alternance.
    ```bash
    for i in $(seq 1 100); do curl -sI http://10.0.0.30/ | grep -i x-backend; done | sort | uniq -c
    ```

3. Une modification sur `srv-files01` est vue partout :
    ```bash
    # Sur srv-files01
    echo "<h1>Version 2 depuis srv-files01</h1>" > /srv/web/index.html
    ```
    ```bash
    # Sur srv-lb01
    for i in 1 2 3 4; do curl -s http://10.0.0.30; done
    ```
    Les 4 réponses doivent afficher la **nouvelle** version.

---

## ✅ Vérification

| Test | Résultat attendu |
|---|---|
| `exportfs -v` sur `srv-files01` | `/srv/web` exporté vers `.31` et `.32` en `ro` |
| `showmount -e 10.0.0.20` sur un client | `/srv/web 10.0.0.31,10.0.0.32` |
| `df -h` sur chaque client | Partage monté sur `/var/www/html` |
| `curl` ×4 via `srv-lb01` | Même page, 4 fois |
| Modification sur `srv-files01` | Visible sur les 4 réponses |
| Redémarrage d'un serveur web | Le partage est remonté automatiquement |

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `access denied by server while mounting` | L'IP du client n'est pas dans `/etc/exports` | Corriger `/etc/exports` puis `exportfs -ra` |
| `clnt_create: RPC: Unable to receive` ou délai dépassé sur `showmount` | Serveur NFS injoignable ou arrêté | `systemctl status nfs-server` sur `srv-files01` ; case **Firewall** Proxmox |
| `403 Forbidden` via le site | Le service web ne peut pas lire les fichiers | Sur `srv-files01` : `chown -R 33:33 /srv/web && chmod -R 755 /srv/web` |
| Une réponse sur deux est l'ancienne page | Un seul des deux serveurs web a monté le partage | `df -h \| grep 10.0.0.20` sur **chaque** backend |
| L'ancienne version reste affichée quelques secondes | Cache d'attributs NFS côté client | Attendre ~30 s ; pour un test, monter avec l'option `actimeo=1` |
| Le serveur web reste bloqué au démarrage | Montage NFS sans `_netdev`, ou `srv-files01` éteint | Ajouter `_netdev` dans `/etc/fstab` ; démarrer `srv-files01` **avant** les serveurs web |

## 🔧 Dépannage courant

!!! question "J'ai modifié `/etc/exports` mais rien ne change ?"
    Le fichier n'est relu qu'avec `exportfs -ra`. Contrôle ensuite avec `exportfs -v`.

!!! question "`mount -a` ne dit rien, mais `df` ne montre pas le partage ?"
    Vérifie la ligne ajoutée dans `/etc/fstab` (`tail -n 2 /etc/fstab`) : une faute de frappe ou une ligne en double empêche le montage. Pour tester à la main : `mount -t nfs 10.0.0.20:/srv/web /var/www/html`.

!!! question "Dans quel ordre démarrer les VM du lab ?"
    `srv-dc01` (DNS), puis `srv-files01` (NFS), puis les serveurs web, puis `srv-lb01`.

## Pour aller plus loin

- 🛠️ [Protéger une page web par un groupe AD](tuto-apache-auth-ad.md) : la zone `/admin` du site
- 🧠 [Le partage de fichiers NFS](concept-nfs.md)
- 🧪 [TP : Load balancer HAProxy et pages web sur NFS](tp-load-balancer-haproxy-nfs.md)

## Sources

- [Page de manuel `exports(5)`](https://manpages.debian.org/stable/nfs-kernel-server/exports.5.en.html) et [`exportfs(8)`](https://manpages.debian.org/stable/nfs-kernel-server/exportfs.8.en.html)
- [Page de manuel `nfs(5)`](https://manpages.debian.org/stable/nfs-common/nfs.5.en.html) : options de montage, cache d'attributs `actimeo`
- [Page de manuel `fstab(5)`](https://manpages.debian.org/stable/mount/fstab.5.en.html) : option `_netdev`
- [Wiki Debian : NFSServerSetup](https://wiki.debian.org/NFSServerSetup)
