---
icon: material/wrench-outline
title: Gérer utilisateurs, groupes et permissions sous Linux
tags: [système, tuto, linux, utilisateurs, groupes, permissions, chmod, chown, sudo, debian]
---

# 🛠️ Gérer utilisateurs, groupes et permissions sous Linux

!!! abstract "Objectif"
    Sur un serveur Linux, créer les comptes d'une petite équipe, monter un **dossier partagé** réservé au service comptabilité, et donner les droits **sudo** à un technicien, sans jamais utiliser `chmod 777`.

**Niveau :** 🟢 Bases · **Durée :** 30 à 40 min

## Prérequis

- Comprendre comptes, groupes, `rwx` et sudo → 🧠 [Comptes, droits et permissions Linux](concept-permissions-linux.md)
- Une VM Linux (Debian/Ubuntu ou Rocky/RHEL), par exemple un clone Proxmox → 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)
- Un compte qui a déjà les droits sudo (ou le mot de passe root)

## Le scénario

| Personne | Compte | Rôle | Besoin |
|---|---|---|---|
| Alice | `alice` | Comptable | Lire et écrire dans `/srv/partage/compta` |
| Bruno | `bruno` | Comptable | Idem |
| Chloé | `chloe` | Technicienne | Administrer le serveur avec `sudo`, **sans** accès à la compta |

---

## Étape 1 — Créer le groupe et les utilisateurs

```bash
# Le groupe du service
sudo groupadd compta
```

=== "Debian / Ubuntu"
    ```bash
    # adduser est interactif : il crée le dossier personnel et demande le mot de passe
    sudo adduser alice
    sudo adduser bruno
    sudo adduser chloe
    ```

=== "Rocky / RHEL"
    ```bash
    # -m crée le dossier personnel, -s choisit le shell
    sudo useradd -m -s /bin/bash alice
    sudo useradd -m -s /bin/bash bruno
    sudo useradd -m -s /bin/bash chloe

    # Définir les mots de passe
    sudo passwd alice
    sudo passwd bruno
    sudo passwd chloe
    ```

Vérifie que les comptes existent :

```bash
getent passwd alice bruno chloe
```

## Étape 2 — Mettre les comptables dans le groupe

```bash
# -aG : AJOUTER au groupe (sans le -a, les autres groupes sont effacés)
sudo usermod -aG compta alice
sudo usermod -aG compta bruno

# Vérifier
getent group compta
id alice
```

Résultat attendu de `getent group compta` : `compta:x:1001:alice,bruno`.

## Étape 3 — Créer le dossier partagé

```bash
sudo mkdir -p /srv/partage/compta

# Propriétaire root, groupe compta
sudo chown root:compta /srv/partage/compta

# 2770 = setgid (2) + rwx propriétaire + rwx groupe + rien pour les autres
sudo chmod 2770 /srv/partage/compta

ls -ld /srv/partage/compta
```

Résultat attendu : `drwxrws--- 2 root compta ... /srv/partage/compta`.

!!! info "Pourquoi le `s` (setgid) ?"
    Sans lui, un fichier créé par Alice appartient au groupe `alice` : Bruno ne peut pas le modifier. Avec le setgid, **tout fichier créé dans le dossier prend le groupe `compta`**.

Pour que les nouveaux fichiers soient aussi **modifiables** par le groupe, il faut que le masque de création (`umask`) de chaque utilisateur laisse l'écriture au groupe :

```bash
# Pour la session en cours
umask 0002

# Pour les connexions futures d'Alice et Bruno
echo 'umask 0002' | sudo tee -a /home/alice/.profile /home/bruno/.profile
```

## Étape 4 — Donner sudo à la technicienne

=== "Debian / Ubuntu"
    ```bash
    sudo usermod -aG sudo chloe
    ```

=== "Rocky / RHEL"
    ```bash
    sudo usermod -aG wheel chloe
    ```

Pour des droits plus fins (par exemple seulement redémarrer des services), crée une règle dédiée, **toujours via `visudo`** :

```bash
sudo visudo -f /etc/sudoers.d/chloe
```

```text
# Chloé peut seulement gérer les services et lire les journaux
chloe ALL=(root) /usr/bin/systemctl, /usr/bin/journalctl
```

!!! tip "Retirer les droits plus tard"
    `sudo gpasswd -d chloe sudo` (ou `wheel`) retire Chloé du groupe ; supprimer le fichier `/etc/sudoers.d/chloe` retire la règle.

## Étape 5 — Ajuster les droits d'un fichier existant

Quelques cas typiques à connaître :

```bash
# Rendre un script exécutable par son propriétaire
chmod u+x sauvegarde.sh

# Fichier lisible par tous, modifiable par le propriétaire
chmod 644 notes.txt

# Protéger sa clé SSH
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys

# Remettre tout un dossier au bon groupe (récursif)
sudo chown -R :compta /srv/partage/compta

# Retirer tout droit aux « autres », récursivement
sudo chmod -R o-rwx /srv/partage/compta
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `Permission denied` en lisant un fichier | Tu n'as pas `r` dans la catégorie qui te concerne | `ls -l` puis `id` : es-tu propriétaire ? dans le groupe ? |
| `Permission denied` en faisant `cd` | Le dossier n'a pas `x` pour toi | `chmod g+x dossier` (ou ajouter l'utilisateur au groupe) |
| `drwxrws---` | Dossier avec setgid, accès réservé au groupe | Normal pour un dossier partagé |
| `drwxrwS---` (S majuscule) | setgid sans `x` pour le groupe : personne du groupe ne peut entrer | `chmod g+x dossier` |
| `alice is not in the sudoers file` | Alice n'est ni dans `sudo`/`wheel` ni dans une règle sudoers | Étape 4, si elle doit vraiment être admin |
| `bash: ./script.sh: Permission denied` | Le script n'est pas exécutable | `chmod u+x script.sh` |

## Vérification finale

Teste chaque besoin du scénario en te mettant **à la place** de chaque utilisateur :

```bash
# Alice crée un fichier dans le partage
sudo -u alice bash -c 'umask 0002; echo "bilan" > /srv/partage/compta/bilan.txt'
ls -l /srv/partage/compta/bilan.txt     # groupe compta, droits rw-rw-r-- (les autres sont bloqués par le dossier)

# Bruno peut le modifier
sudo -u bruno bash -c 'echo "corrigé" >> /srv/partage/compta/bilan.txt' && echo OK

# Chloé n'a PAS accès à la compta
sudo -u chloe ls /srv/partage/compta    # doit répondre Permission denied

# Chloé a bien sudo (à lancer depuis une session ouverte en tant que chloe)
su - chloe
sudo -l                                 # liste ses droits sudo
exit
```

| Test | Résultat attendu |
|---|---|
| Alice écrit dans le partage | ✅ Fichier créé, groupe `compta` |
| Bruno modifie le fichier d'Alice | ✅ `OK` |
| Chloé liste le partage | ❌ `Permission denied` |
| `sudo -l` en tant que Chloé | ✅ Droits listés |

## Dépannage courant

- **Alice vient d'être ajoutée au groupe mais n'a toujours pas accès** : ses groupes sont lus à la connexion. Elle doit se **déconnecter/reconnecter** (ou `newgrp compta`). Vérifier avec `id` dans sa session.
- **Bruno ne peut pas modifier un fichier créé par Alice** : le fichier est en `rw-r-----`. L'`umask` d'Alice était `0022`. Corriger avec `sudo chmod g+w fichier` puis régler l'`umask` (étape 3).
- **Un fichier n'a pas le groupe `compta`** : il a été créé avant le setgid, ou copié avec `cp -p` / déplacé avec `mv` (qui garde le groupe d'origine). `sudo chown :compta fichier`.
- **sudo ne fonctionne plus après une modification** : une erreur dans sudoers. Se connecter en root (console Proxmox) et lancer `visudo` pour corriger.
- **Un utilisateur a perdu ses groupes** : `usermod -G` a été utilisé sans `-a`. Les rajouter un par un avec `usermod -aG`.

??? quote "Sources"
    - [Page de manuel useradd(8)](https://man7.org/linux/man-pages/man8/useradd.8.html)
    - [Page de manuel usermod(8)](https://man7.org/linux/man-pages/man8/usermod.8.html)
    - [Page de manuel chmod(1)](https://man7.org/linux/man-pages/man1/chmod.1.html)
    - [Page de manuel chown(1)](https://man7.org/linux/man-pages/man1/chown.1.html)
