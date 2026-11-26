---
icon: material/wrench-outline
title: Faire une intervention propre de A à Z
tags: [bonnes pratiques, tuto, méthode, ssh, smb, proxmox, snapshot]
---

# 🛠️ Faire une intervention propre de A à Z

!!! abstract "Objectif"
    Appliquer **tous les réflexes de l'admin** sur une vraie modification : préparer, prendre un snapshot, sauvegarder la conf, noter l'état de départ, modifier, valider, vérifier, savoir revenir en arrière et documenter.
    Deux cas au choix, en onglets à chaque étape :

    - **Linux** : durcir le SSH de `srv-web01` (plus de connexion `root`, plus de mot de passe, seulement des clés)
    - **Windows** : désactiver le vieux protocole SMBv1 sur `srv-dc01`

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 30 min

## Prérequis

- Les réflexes expliqués → 🧠 [Les réflexes de l'admin](concept-reflexes-admin.md)
- Savoir prendre un snapshot Proxmox → 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](../transverse/tuto-proxmox-backup.md)
- Linux : une connexion SSH par clé qui marche déjà pour le compte `admin` → 🧠 [Comprendre SSH](../reseau/concept-ssh.md)
- Windows : une session PowerShell administrateur sur `srv-dc01`
- Dans cet exemple, `srv-web01` est la VM **100** et `srv-dc01` la VM **110** sur `pve01`

---

## Étape 1 — Préparer : quoi, pourquoi, quand, et comment revenir en arrière

Avant de taper la moindre commande, écrire ces quatre lignes (dans le ticket, ou en brouillon du journal des changements) :

=== "Linux"

    | Question | Réponse |
    |---|---|
    | **Quoi** | Interdire `root` et les mots de passe en SSH sur `srv-web01`, n'autoriser que `admin` |
    | **Pourquoi** | Les robots d'Internet testent des mots de passe sur `root` en permanence |
    | **Quand** | Ce soir, hors heures d'utilisation, 15 min |
    | **Retour arrière** | Supprimer le fichier ajouté et recharger SSH, ou rollback du snapshot |

=== "Windows"

    | Question | Réponse |
    |---|---|
    | **Quoi** | Désactiver SMBv1 sur `srv-dc01` |
    | **Pourquoi** | SMBv1 est obsolète et a servi à des attaques massives (rançongiciel WannaCry) |
    | **Quand** | Ce soir, 15 min, après avoir vérifié qu'aucun vieil appareil ne l'utilise |
    | **Retour arrière** | Réactiver SMBv1 par la même commande, ou rollback du snapshot |

!!! tip "Le test du retour arrière"
    Si tu n'arrives pas à écrire la ligne « Retour arrière », tu n'es pas prêt à faire la modification.

## Étape 2 — Prendre un snapshot de la VM

Dans l'interface Proxmox : VM → **Snapshots** → **Take Snapshot**, ou dans le shell de `pve01` :

=== "Linux"

    ```bash
    qm snapshot 100 avant-durcissement-ssh --description "Avant durcissement SSH, $(date +%F)"
    qm listsnapshot 100
    ```

=== "Windows"

    ```bash
    qm snapshot 110 avant-smb1-off --description "Avant désactivation SMBv1, $(date +%F)"
    qm listsnapshot 110
    ```

!!! info "Pour un contrôleur de domaine"
    Un snapshot d'AD est un filet de sécurité pour une petite modif, pas une sauvegarde : avec plusieurs contrôleurs de domaine, revenir à un vieux snapshot peut désynchroniser l'annuaire. Sur un lab à un seul DC, pas de souci.

## Étape 3 — Sauvegarder la configuration actuelle

=== "Linux"

    ```bash
    sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%F)
    ls -l /etc/ssh/sshd_config*
    ```

    `cp -a` garde les droits et la date d'origine. Le suffixe daté évite d'écraser une ancienne copie.

=== "Windows"

    ```powershell
    New-Item -ItemType Directory -Path C:\Admin\Avant -Force | Out-Null
    Get-SmbServerConfiguration |
        Export-Clixml C:\Admin\Avant\smb-config-$(Get-Date -Format yyyy-MM-dd).xml
    Get-ChildItem C:\Admin\Avant
    ```

## Étape 4 — Noter l'état de départ

On garde la preuve de « comment c'était avant », pour comparer après.

=== "Linux"

    ```bash
    systemctl is-active ssh                       # doit répondre active
    sudo ss -tlpn | grep ssh                      # SSH écoute sur le port 22
    sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|allowusers'
    ```

=== "Windows"

    ```powershell
    Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol
    # Quelqu'un utilise-t-il encore SMBv1 ? (vide = personne)
    Get-SmbSession | Where-Object Dialect -lt 2 | Select-Object ClientComputerName, Dialect
    ```

## Étape 5 — Faire la modification (une seule)

=== "Linux"

    Plutôt que de modifier le gros fichier d'origine, on ajoute un petit fichier à part : c'est plus lisible et le retour arrière se résume à le supprimer.

    ```bash
    sudo tee /etc/ssh/sshd_config.d/10-durcissement.conf > /dev/null <<'CONF'
    # Durcissement SSH - voir journal des changements
    PermitRootLogin no
    PasswordAuthentication no
    MaxAuthTries 3
    AllowUsers admin
    CONF
    ```

=== "Windows"

    ```powershell
    Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
    ```

## Étape 6 — Valider, puis appliquer en gardant une porte de secours

=== "Linux"

    ```bash
    sudo sshd -t && echo "Syntaxe OK"      # aucune sortie d'erreur = OK
    sudo systemctl reload ssh               # reload : les sessions ouvertes ne sont pas coupées
    ```

    !!! danger "Ne ferme pas ta session"
        Garde la session SSH actuelle ouverte. Teste depuis une **deuxième** fenêtre (étape 7). Si ça ne marche pas, tu corriges depuis la première. En dernier recours : console de la VM dans Proxmox.

=== "Windows"

    La commande s'applique tout de suite, sans redémarrage. Garde ta session ouverte (ou la console Proxmox de la VM) le temps des vérifications.

## Étape 7 — Vérifier que ça marche… et que rien d'autre n'est cassé

=== "Linux"

    Depuis ton PC, dans une **nouvelle** fenêtre :

    ```bash
    ssh admin@srv-web01                     # doit marcher (par clé)
    ssh root@srv-web01                      # doit être refusé
    ssh -o PubkeyAuthentication=no admin@srv-web01   # doit être refusé (plus de mot de passe)
    ```

    Puis sur `srv-web01` :

    ```bash
    sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|allowusers'
    curl -I http://localhost                # le site web répond toujours
    ```

=== "Windows"

    ```powershell
    Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol
    # Les partages répondent toujours (SYSVOL, NETLOGON) depuis un poste
    Test-Path \\srv-dc01\SYSVOL
    Test-NetConnection srv-dc01 -Port 445
    ```

    Depuis `PC-01` : ouvrir une session de domaine, vérifier que les lecteurs réseau se montent.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `ssh admin@...` fonctionne, `root` refusé | Le changement a réussi | Passer à l'étape 9 |
| `Permission denied (publickey)` pour `admin` | La clé n'est pas installée pour `admin` | Depuis la session ouverte : `ssh-copy-id` ou retour arrière |
| `sshd -t` affiche une erreur | Faute de frappe dans le fichier | Corriger le fichier, **ne pas recharger** avant que `sshd -t` soit muet |
| `EnableSMB1Protocol : False`, partages OK | Le changement a réussi | Passer à l'étape 9 |
| Un vieil appareil (imprimante, NAS) n'accède plus aux partages | Il ne parlait que SMBv1 | Retour arrière, puis mettre à jour ou remplacer l'appareil |

## Étape 8 — Revenir en arrière (si besoin)

Du plus léger au plus radical :

=== "Linux"

    ```bash
    # 1. Annuler juste la modification
    sudo rm /etc/ssh/sshd_config.d/10-durcissement.conf
    sudo systemctl reload ssh

    # 2. Remettre la copie d'origine (si on avait modifié le fichier principal)
    sudo cp -a /etc/ssh/sshd_config.bak-AAAA-MM-JJ /etc/ssh/sshd_config

    # 3. En dernier recours, sur pve01 : revenir au snapshot (perd tout ce qui a changé depuis)
    qm rollback 100 avant-durcissement-ssh
    ```

=== "Windows"

    ```powershell
    # 1. Annuler juste la modification
    Set-SmbServerConfiguration -EnableSMB1Protocol $true -Force
    ```

    ```bash
    # 2. En dernier recours, sur pve01 : revenir au snapshot
    qm rollback 110 avant-smb1-off
    ```

## Étape 9 — Documenter

Une entrée dans le journal des changements (modèle complet dans 📋 [Nommage et documentation](memo-nommage-documentation.md)) :

```markdown
## 2026-11-26 — srv-web01 — Durcissement SSH
- Qui : admin
- Quoi : ajout de /etc/ssh/sshd_config.d/10-durcissement.conf
  (PermitRootLogin no, PasswordAuthentication no, MaxAuthTries 3, AllowUsers admin)
- Pourquoi : limiter les attaques par mot de passe
- Vérifié : connexion admin par clé OK, root refusé, site web OK
- Retour arrière : supprimer le fichier + systemctl reload ssh
- Snapshot : avant-durcissement-ssh (à supprimer le 2026-12-03)
```

Mettre aussi à jour la **fiche serveur** de `srv-web01` (section « Accès »).

## Étape 10 — Faire le ménage

Quelques jours plus tard, si tout va bien :

```bash
qm delsnapshot 100 avant-durcissement-ssh     # ou 110 avant-smb1-off
qm listsnapshot 100                           # plus de snapshot oublié
```

---

## Vérification finale

- [x] Le changement était décrit (quoi, pourquoi, quand, retour arrière) **avant** de commencer
- [x] Un snapshot et une copie de la conf existaient avant la modification
- [x] L'état « avant » et l'état « après » ont été comparés
- [x] Le service visé **et** les services voisins fonctionnent
- [x] Le journal des changements et la fiche serveur sont à jour
- [x] Le snapshot a été supprimé (ou une date de suppression est notée)

## Dépannage courant

??? question "Je me suis coupé l'accès SSH et j'ai fermé ma session"
    Ouvre la **console** de la VM dans Proxmox (VM → Console) : elle ne passe pas par SSH. Connecte-toi en local, supprime le fichier de durcissement et recharge SSH.

??? question "`sshd -t` ne dit rien mais mes réglages ne s'appliquent pas"
    Les fichiers de `sshd_config.d/` sont lus **par ordre alphabétique** et c'est la **première** valeur trouvée qui gagne. Un autre fichier (ex. `50-cloud-init.conf` avec `PasswordAuthentication yes`) lu avant le tien peut l'emporter. Vérifie avec `sudo sshd -T` et numérote ton fichier plus bas (`10-…`).

??? question "Sur Rocky Linux, `systemctl reload ssh` répond `Unit ssh.service not found`"
    Sur Rocky / RHEL, le service s'appelle `sshd` (sur Debian, `ssh`) : `sudo systemctl reload sshd`.

??? question "Le rollback du snapshot a fait disparaître d'autres modifications"
    C'est normal : un rollback ramène **toute** la VM à l'instant du snapshot. D'où l'ordre du retour arrière : d'abord annuler la modif elle-même, le snapshot seulement en dernier recours.
