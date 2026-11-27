---
icon: material/wrench-outline
title: Sauvegarder et restaurer ses VM avec Proxmox
tags: [transverse, tuto, sauvegarde, proxmox, pbs, snapshot]
---

# 🛠️ Sauvegarder et restaurer ses VM avec Proxmox

!!! abstract "Objectif"
    Protéger les VM d'un Proxmox VE : un **snapshot** avant une manip risquée, une **sauvegarde** ponctuelle, une **tâche planifiée** avec rétention vers un disque séparé, un **test de restauration**, puis (bonus) une copie vers un **Proxmox Backup Server** pour se rapprocher de la règle 3-2-1.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 45 min (1 h 30 avec le bonus PBS)

## Prérequis

- Savoir faire la différence entre snapshot et sauvegarde → 🧠 [Les sauvegardes et la règle 3-2-1](concept-sauvegardes.md)
- Un Proxmox VE avec au moins une VM (ici la VM **100**, `srv-web01`)
- Un accès à l'interface web (`https://<ip-proxmox>:8006`) et au shell du nœud (bouton **Shell** ou SSH en `root`)
- Pour l'étape 3 : un **second disque** dans le Proxmox (ou un disque USB) ; pour le bonus : une autre machine ou VM pour PBS

## Le plan qu'on va monter

| Étape | Quoi | Où c'est stocké | Couvre |
|---|---|---|---|
| 1 | Snapshot avant mise à jour | Disque de la VM | Erreur de manip |
| 2 | Sauvegarde manuelle | Stockage `local` | Premier essai |
| 3 | Sauvegarde planifiée + rétention | Disque `sauvegardes` séparé | Panne du disque système |
| 4 | Test de restauration | Nouvelle VM 900 | Vérifier que ça marche vraiment |
| 5 | Copie vers PBS (bonus) | Autre machine | Panne de l'hôte, 3-2-1 |

```
[VM 100 srv-web01] ──snapshot──▶ (même disque)
        │
        ├──vzdump chaque nuit──▶ [Disque « sauvegardes »]   copie n°2, 2ᵉ support
        └──vzdump──────────────▶ [Proxmox Backup Server]    copie n°3, autre machine
```

---

## Étape 1 — Prendre un snapshot avant une manip risquée

Avant une mise à jour ou un changement de configuration, on pose un point de retour.

=== "Interface web"
    1. Sélectionner la VM **100** → **Snapshots** → **Take Snapshot**.
    2. *Name* : `avant-maj` (lettres, chiffres, `-` et `_`, sans espace).
    3. *Include RAM* : à cocher seulement si tu veux revenir **VM allumée** dans l'état exact (plus long, plus lourd).
    4. *Description* : `Avant apt full-upgrade` → **Take Snapshot**.

    Pour revenir en arrière : sélectionner le snapshot → **Rollback**. Une fois la manip validée : **Remove**.

=== "Ligne de commande"
    ```bash
    qm snapshot 100 avant-maj --description "Avant apt full-upgrade"
    qm listsnapshot 100            # liste les snapshots
    qm rollback 100 avant-maj      # revenir en arrière si ça a mal tourné
    qm delsnapshot 100 avant-maj   # supprimer une fois la manip validée
    ```

    Pour un conteneur LXC, même logique avec `pct snapshot`, `pct rollback`, `pct delsnapshot`.

!!! warning "Un snapshot n'est pas une sauvegarde"
    Il vit sur le même disque que la VM. Supprime-le dès que la manip est validée : il grossit et ralentit la VM.

## Étape 2 — Faire une première sauvegarde manuelle

=== "Interface web"
    1. VM **100** → **Backup** → **Backup now**.
    2. *Storage* : `local` · *Mode* : **Snapshot** · *Compression* : **ZSTD**.
    3. **Backup**, puis suivre la tâche jusqu'à `TASK OK`.

=== "Ligne de commande"
    ```bash
    vzdump 100 --storage local --mode snapshot --compress zstd
    pvesm list local --content backup    # vérifier que l'archive est là
    ```

    L'archive arrive dans `/var/lib/vz/dump/`, sous un nom du type `vzdump-qemu-100-2026_11_26-21_00_00.vma.zst`.

Les trois **modes** de vzdump :

| Mode | La VM pendant la sauvegarde | Quand l'utiliser |
|---|---|---|
| **Snapshot** | Reste allumée, aucune coupure | Le choix par défaut (installer `qemu-guest-agent` dans la VM pour des fichiers cohérents) |
| **Suspend** | Mise en pause un court instant | Rarement utile aujourd'hui |
| **Stop** | Éteinte puis rallumée | Cohérence maximale, si une coupure est acceptable |

## Étape 3 — Planifier une sauvegarde nocturne sur un disque séparé

Le stockage `local` est sur le **même disque** que Proxmox : on ajoute un disque dédié.

### 3a. Préparer le disque de sauvegarde

=== "Interface web"
    1. Nœud → **Disks** : repérer le nouveau disque (ex. `/dev/sdb`). S'il contient d'anciennes partitions : **Wipe Disk** (⚠️ efface tout).
    2. Nœud → **Disks → Directory → Create: Directory** :
        - *Disk* : `/dev/sdb` · *Filesystem* : `ext4` · *Name* : `sauvegardes`
        - ☑ *Add Storage* → **Create**
    3. **Datacenter → Storage** → `sauvegardes` → **Edit** → *Content* : sélectionner **Backup** (VZDump backup file) → **OK**.

=== "Ligne de commande"
    ```bash
    lsblk                                     # repérer le disque vide, ex. /dev/sdb
    mkfs.ext4 -L sauvegardes /dev/sdb         # ⚠️ efface tout le disque
    mkdir -p /mnt/sauvegardes
    echo 'LABEL=sauvegardes /mnt/sauvegardes ext4 defaults,nofail 0 2' >> /etc/fstab
    mount -a
    pvesm add dir sauvegardes --path /mnt/sauvegardes --content backup --is_mountpoint yes
    pvesm status                              # le stockage « sauvegardes » est actif
    ```

    `--is_mountpoint yes` empêche Proxmox d'écrire sur le disque système si le disque de sauvegarde n'est pas monté.

### 3b. Créer la tâche planifiée avec rétention

=== "Interface web"
    1. **Datacenter → Backup → Add**.
    2. Onglet **General** :
        - *Storage* : `sauvegardes` · *Schedule* : `21:00` (tous les soirs)
        - *Selection mode* : **Include selected VMs** et cocher les VM (ou **All**)
        - *Mode* : **Snapshot** · *Compression* : **ZSTD**
    3. Onglet **Retention** : *Keep Daily* `7` · *Keep Weekly* `4` · *Keep Monthly* `3`.
    4. **Create**, puis sélectionner la tâche → **Run now** pour la tester tout de suite.

=== "Ligne de commande"
    ```bash
    pvesh create /cluster/backup \
      --schedule "21:00" --storage sauvegardes --vmid 100 \
      --mode snapshot --compress zstd --enabled 1 \
      --prune-backups keep-daily=7,keep-weekly=4,keep-monthly=3
    pvesh get /cluster/backup                 # liste les tâches planifiées
    ```

    Les tâches sont enregistrées dans `/etc/pve/jobs.cfg`.

Quelques exemples de *Schedule* :

| Écriture | Signification |
|---|---|
| `21:00` | Tous les jours à 21 h |
| `sat 02:00` | Le samedi à 2 h |
| `mon..fri 22:30` | Du lundi au vendredi à 22 h 30 |
| `*/6:00` | Toutes les 6 heures |

!!! tip "Être prévenu en cas d'échec"
    **Datacenter → Notifications** : vérifier qu'une cible mail est configurée. Une sauvegarde qui échoue sans que personne ne le sache, c'est comme pas de sauvegarde.

## Étape 4 — Tester une restauration

C'est l'étape que tout le monde saute, et la seule qui prouve que la sauvegarde sert à quelque chose. On restaure dans une **nouvelle** VM (900) pour ne pas toucher à l'originale.

=== "Interface web"
    1. Nœud → stockage `sauvegardes` → **Backups** → choisir la dernière archive de la VM 100 → **Restore**.
    2. *VM* : `900` · *Storage* : `local-lvm` · ☑ *Unique* (nouvelle adresse MAC) · ☐ *Start after restore*.
    3. **Restore** et attendre `TASK OK`.
    4. VM **900** → **Hardware** → *Network Device* → **Edit** → ☑ *Disconnect* (sinon conflit d'IP avec la VM 100).
    5. **Start**, ouvrir la **Console** et vérifier que le système démarre et que les données sont là.

=== "Ligne de commande"
    ```bash
    pvesm list sauvegardes --content backup            # repérer l'archive
    qmrestore /mnt/sauvegardes/dump/vzdump-qemu-100-2026_11_26-21_00_00.vma.zst 900 \
      --storage local-lvm --unique 1
    qm set 900 --net0 virtio,bridge=vmbr0,link_down=1  # carte réseau débranchée
    qm start 900
    ```

    Pour un conteneur : `pct restore 901 <archive>.tar.zst --storage local-lvm`.

Une fois vérifié, supprimer la VM de test : **More → Remove**, ou `qm stop 900 && qm destroy 900 --purge`.

!!! danger "Restaurer sur l'ID d'origine écrase la VM"
    Le bouton **Restore** depuis la VM 100 elle-même **remplace** son disque actuel. Ne le faire que si la VM est vraiment perdue.

## Étape 5 (bonus) — Envoyer une copie vers Proxmox Backup Server

PBS stocke les sauvegardes avec **déduplication** (seuls les blocs modifiés sont envoyés) et vérifie leur intégrité. Installe-le depuis l'ISO officielle sur **une autre machine**, ou sur une VM avec un disque dédié pour apprendre. Son interface est sur `https://<ip-pbs>:8007`.

### 5a. Côté PBS : datastore, utilisateur, droits

=== "Interface web"
    1. **Administration → Storage / Disks → Directory → Create: Directory** : *Disk* `/dev/sdb`, *Name* `store1`, ☑ *Add as Datastore*.
    2. **Configuration → Access Control → User Management → Add** : *User name* `backup`, *Realm* `pbs`, un mot de passe.
    3. **Datastore → store1 → Permissions → Add → User Permission** : *Path* `/datastore/store1`, *User* `backup@pbs`, *Role* **DatastoreBackup**.
    4. **Dashboard → Show Fingerprint** : copier l'empreinte du certificat.

=== "Ligne de commande"
    ```bash
    proxmox-backup-manager datastore create store1 /mnt/datastore/store1
    proxmox-backup-manager user create backup@pbs --password 'MotDePasse'
    proxmox-backup-manager acl update /datastore/store1 DatastoreBackup --auth-id backup@pbs
    proxmox-backup-manager cert info | grep Fingerprint
    ```

### 5b. Côté Proxmox VE : ajouter PBS comme stockage

=== "Interface web"
    **Datacenter → Storage → Add → Proxmox Backup Server** :

    - *ID* : `pbs` · *Server* : IP du PBS (ex. `192.168.1.50`)
    - *Username* : `backup@pbs` · *Password* : celui créé plus haut
    - *Datastore* : `store1` · *Fingerprint* : l'empreinte copiée → **Add**

=== "Ligne de commande"
    ```bash
    pvesm add pbs pbs --server 192.168.1.50 --datastore store1 \
      --username backup@pbs --password 'MotDePasse' \
      --fingerprint 'aa:bb:cc:...'
    pvesm status                          # « pbs » doit être active
    ```

Créer ensuite une **seconde tâche** (étape 3b) vers le stockage `pbs`, par exemple le dimanche à 2 h (`sun 02:00`). Côté PBS, ajouter dans **Datastore → store1** une tâche **Prune & GC** (rétention et nettoyage) et une tâche **Verify** (vérification des sauvegardes).

---

## ✅ Vérification

```bash
qm listsnapshot 100                         # plus de snapshot oublié
pvesm list sauvegardes --content backup     # des archives récentes
pvesh get /cluster/backup                   # la tâche planifiée existe
pvesm status                                # tous les stockages « active »
```

Dans l'interface : **Datacenter → Backup → Job Detail** et le journal des tâches en bas de l'écran (`TASK OK` chaque nuit).

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `TASK OK` chaque nuit, archives qui tournent (7/4/3) | Tout fonctionne | Refaire un test de restauration de temps en temps |
| `No space left on device` | Le stockage est plein | Réduire la rétention, supprimer de vieilles archives, agrandir le disque |
| Le stockage `sauvegardes` est grisé / inactif | Disque non monté | `mount -a`, vérifier `/etc/fstab` et le câble USB |
| La sauvegarde reste longtemps sur `fs-freeze` | Agent invité activé mais pas installé dans la VM | Installer `qemu-guest-agent` dans la VM, ou désactiver l'option *QEMU Guest Agent* |
| `pbs` inactif avec une erreur de certificat | Mauvaise empreinte | Recopier la *Fingerprint* depuis le tableau de bord PBS |
| La VM restaurée n'a pas de réseau | C'est voulu (carte débranchée) | Décocher *Disconnect* seulement si l'originale est éteinte |

## 🔧 Dépannage courant

!!! question "Le bouton « Take Snapshot » est grisé ?"
    Le stockage du disque ne gère pas les snapshots (ex. disque **raw** sur un stockage *Directory*). Utiliser `local-lvm` (LVM-thin), ZFS ou un disque **qcow2**.

!!! question "« storage 'sauvegardes' does not support backups » ?"
    Le *Content* du stockage n'inclut pas **Backup**. Datacenter → Storage → Edit → *Content* → ajouter **VZDump backup file**.

!!! question "La tâche planifiée ne s'est jamais lancée ?"
    Vérifier qu'elle est *Enabled*, que l'heure du nœud est bonne (`timedatectl`) et regarder le service : `journalctl -u pvescheduler`.

!!! question "PBS refuse la connexion (`401 Unauthorized` ou `permission check failed`) ?"
    Vérifier le nom complet `backup@pbs` (avec le *realm*) et que la permission **DatastoreBackup** est bien posée sur `/datastore/store1`.

## Voir aussi

- 🧠 [Les sauvegardes et la règle 3-2-1](concept-sauvegardes.md)
- 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md)
- 🛠️ [Se connecter en SSH à une VM par son nom](../reseau/tuto-ssh-nom-vm.md)

??? quote "Sources"
    - [Proxmox VE : sauvegarde et restauration](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
    - [Page de manuel vzdump(1)](https://pve.proxmox.com/pve-docs/vzdump.1.html)
    - [Page de manuel qmrestore(1)](https://pve.proxmox.com/pve-docs/qmrestore.1.html)
