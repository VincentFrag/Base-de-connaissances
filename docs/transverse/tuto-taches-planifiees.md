---
icon: material/wrench-outline
title: Automatiser une tâche avec cron et le Planificateur de tâches
tags: [transverse, tuto, scripting, cron, planificateur, bash, powershell]
---

# 🛠️ Automatiser une tâche avec cron et le Planificateur de tâches

!!! abstract "Objectif"
    Écrire un petit script de **sauvegarde d'un dossier partagé** (archive datée + suppression des archives de plus de 7 jours + fichier de log), puis le faire tourner **tout seul chaque nuit à 2 h** : avec **cron** sur un serveur Linux, avec le **Planificateur de tâches** sur un Windows Server.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 45 min (les deux plateformes)

## Prérequis

- Connaître la syntaxe de base des variables, conditions et boucles → 📋 [Mémo scripting Bash & PowerShell](memo-bash-powershell.md)
- Savoir ce qu'est une bonne sauvegarde (et pourquoi la tester) → 🧠 [Les sauvegardes et la règle 3-2-1](concept-sauvegardes.md)
- **Linux** : une VM Debian, ici `srv-files01` (`10.0.0.20`), avec un compte `admin` qui a `sudo`
- **Windows** : un Windows Server, ici `srv-dc01` (`10.0.0.10`), avec une session administrateur et PowerShell 5.1 ou plus

## Le plan

| | Linux (`srv-files01`) | Windows (`srv-dc01`) |
|---|---|---|
| Dossier à sauvegarder | `/srv/partage` | `C:\Partages` |
| Destination | `/sauvegardes` | `D:\Sauvegardes` *(ou `C:\Sauvegardes` sans second disque)* |
| Script | `/usr/local/bin/sauvegarde-partage.sh` | `C:\Scripts\Sauvegarde-Partage.ps1` |
| Log | `/var/log/sauvegarde-partage.log` | `D:\Sauvegardes\sauvegarde.log` |
| Planificateur | `cron` (crontab de `root`) | Planificateur de tâches (compte `SYSTEM`) |
| Quand | Tous les jours à 2 h 00 | Tous les jours à 2 h 00 |

!!! info "La règle d'or d'une tâche planifiée"
    Une tâche planifiée tourne **sans toi** : pas de terminal, pas de session ouverte, souvent un autre compte et un autre dossier courant. Donc : **chemins absolus partout**, **un log** pour savoir ce qui s'est passé, et **tester le script à la main** avant de le planifier.

## Étape 1 : écrire le script

=== "Linux (Bash)"

    Créer les dossiers et quelques fichiers de test :

    ```bash
    sudo mkdir -p /srv/partage /sauvegardes
    echo "Compte rendu réunion" | sudo tee /srv/partage/cr-reunion.txt
    ```

    Créer le script :

    ```bash
    sudo nano /usr/local/bin/sauvegarde-partage.sh
    ```

    ```bash
    #!/bin/bash
    # Sauvegarde quotidienne de /srv/partage avec rétention de 7 jours
    set -euo pipefail

    SOURCE="/srv/partage"
    DEST="/sauvegardes"
    RETENTION_JOURS=7
    LOG="/var/log/sauvegarde-partage.log"
    ARCHIVE="$DEST/partage_$(date +%F_%H%M).tar.gz"

    log() {
        echo "$(date '+%F %T')  $1" >> "$LOG"
    }

    log "Début de la sauvegarde de $SOURCE"

    # 1. Vérifier que la source existe
    if [ ! -d "$SOURCE" ]; then
        log "ERREUR : $SOURCE introuvable"
        exit 1
    fi

    # 2. Créer l'archive datée
    tar -czf "$ARCHIVE" -C "$(dirname "$SOURCE")" "$(basename "$SOURCE")"
    log "Archive créée : $ARCHIVE ($(du -h "$ARCHIVE" | cut -f1))"

    # 3. Supprimer les archives trop anciennes
    find "$DEST" -name "partage_*.tar.gz" -mtime +"$RETENTION_JOURS" -print -delete | while read -r f; do
        log "Supprimée (plus de $RETENTION_JOURS jours) : $f"
    done

    log "Fin de la sauvegarde : OK"
    ```

    Le rendre exécutable par root uniquement :

    ```bash
    sudo chown root:root /usr/local/bin/sauvegarde-partage.sh
    sudo chmod 750 /usr/local/bin/sauvegarde-partage.sh
    ```

=== "Windows (PowerShell)"

    Créer les dossiers et un fichier de test (PowerShell **en administrateur**) :

    ```powershell
    New-Item -ItemType Directory -Force C:\Partages, D:\Sauvegardes, C:\Scripts
    "Compte rendu réunion" | Out-File C:\Partages\cr-reunion.txt
    ```

    Créer le script :

    ```powershell
    notepad C:\Scripts\Sauvegarde-Partage.ps1
    ```

    ```powershell
    # Sauvegarde quotidienne de C:\Partages avec rétention de 7 jours
    $ErrorActionPreference = "Stop"

    $Source         = "C:\Partages"
    $Dest           = "D:\Sauvegardes"
    $RetentionJours = 7
    $Log            = "$Dest\sauvegarde.log"
    $Archive        = "$Dest\partage_$(Get-Date -Format 'yyyy-MM-dd_HHmm').zip"

    function Write-Log($Message) {
        "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')  $Message" | Add-Content $Log
    }

    try {
        Write-Log "Début de la sauvegarde de $Source"

        # 1. Vérifier que la source existe
        if (-not (Test-Path $Source -PathType Container)) {
            throw "$Source introuvable"
        }

        # 2. Créer l'archive datée
        Compress-Archive -Path "$Source\*" -DestinationPath $Archive
        $tailleKo = [math]::Round((Get-Item $Archive).Length / 1KB, 1)
        Write-Log "Archive créée : $Archive ($tailleKo Ko)"

        # 3. Supprimer les archives trop anciennes
        Get-ChildItem "$Dest\partage_*.zip" |
            Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-$RetentionJours) } |
            ForEach-Object {
                Remove-Item $_.FullName
                Write-Log "Supprimée (plus de $RetentionJours jours) : $($_.Name)"
            }

        Write-Log "Fin de la sauvegarde : OK"
    }
    catch {
        Write-Log "ERREUR : $_"
        exit 1
    }
    ```

    !!! tip "Encodage"
        Enregistrer le fichier en **UTF-8 avec BOM** (choix « UTF-8 » dans le Bloc-notes de Windows Server récent) pour que PowerShell 5.1 affiche correctement les accents dans le log.

## Étape 2 : tester le script à la main

=== "Linux (Bash)"

    ```bash
    sudo /usr/local/bin/sauvegarde-partage.sh
    echo "Code retour : $?"
    ls -lh /sauvegardes
    sudo tail /var/log/sauvegarde-partage.log
    ```

=== "Windows (PowerShell)"

    ```powershell
    powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\Scripts\Sauvegarde-Partage.ps1
    "Code retour : $LASTEXITCODE"
    Get-ChildItem D:\Sauvegardes
    Get-Content D:\Sauvegardes\sauvegarde.log -Tail 10
    ```

Résultat attendu : code retour **0**, une archive `partage_<date>.tar.gz` (ou `.zip`), et un log qui finit par `Fin de la sauvegarde : OK`.

!!! tip "Tester la rétention sans attendre 7 jours"
    Vieillir artificiellement une archive, relancer le script et vérifier qu'elle est supprimée :

    - Linux : `sudo touch -d "10 days ago" /sauvegardes/partage_*.tar.gz`
    - Windows : `(Get-Item D:\Sauvegardes\partage_*.zip).LastWriteTime = (Get-Date).AddDays(-10)`

## Étape 3 : planifier la tâche

=== "Linux (cron)"

    **Comprendre la syntaxe d'une ligne cron** : 5 champs de temps, puis la commande.

    ```text
    ┌───────── minute (0-59)
    │ ┌─────── heure (0-23)
    │ │ ┌───── jour du mois (1-31)
    │ │ │ ┌─── mois (1-12)
    │ │ │ │ ┌─ jour de la semaine (0-7, 0 et 7 = dimanche)
    │ │ │ │ │
    0 2 * * *  /usr/local/bin/sauvegarde-partage.sh
    ```

    | Ligne | Signification |
    |---|---|
    | `0 2 * * *` | Tous les jours à 2 h 00 |
    | `*/15 * * * *` | Toutes les 15 minutes |
    | `30 8 * * 1-5` | Du lundi au vendredi à 8 h 30 |
    | `0 3 * * 0` | Le dimanche à 3 h 00 |
    | `0 0 1 * *` | Le 1er de chaque mois à minuit |
    | `@reboot` | Une fois au démarrage |

    Ouvrir la crontab de **root** (le script a besoin des droits root) :

    ```bash
    sudo crontab -e
    ```

    Ajouter à la fin :

    ```text
    # Sauvegarde du partage tous les jours à 2 h
    0 2 * * * /usr/local/bin/sauvegarde-partage.sh >> /var/log/sauvegarde-partage.log 2>&1
    ```

    Le `>> ... 2>&1` envoie aussi dans le log les erreurs qu'afficherait le script (une commande qui plante avant d'avoir pu écrire elle-même).

    Vérifier que la ligne est enregistrée :

    ```bash
    sudo crontab -l
    ```

=== "Windows (Planificateur de tâches)"

    **En PowerShell** (administrateur) :

    ```powershell
    $action = New-ScheduledTaskAction -Execute "powershell.exe" `
        -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\Scripts\Sauvegarde-Partage.ps1"

    $declencheur = New-ScheduledTaskTrigger -Daily -At 2:00

    $compte = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest

    $reglages = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Hours 1)

    Register-ScheduledTask -TaskName "Sauvegarde du partage" -TaskPath "\Lab\" `
        -Action $action -Trigger $declencheur -Principal $compte -Settings $reglages `
        -Description "Archive C:\Partages vers D:\Sauvegardes, rétention 7 jours"
    ```

    | Paramètre | Pourquoi |
    |---|---|
    | `-ExecutionPolicy Bypass` | Le script s'exécute même si la stratégie bloque les `.ps1` |
    | `-UserId "SYSTEM"` | Tourne sans session ouverte et sans mot de passe à stocker |
    | `-StartWhenAvailable` | Si le serveur était éteint à 2 h, la tâche se lance au démarrage suivant |
    | `-ExecutionTimeLimit` | Coupe la tâche si elle reste bloquée plus d'une heure |
    | `-TaskPath "\Lab\"` | Range la tâche dans un dossier à part, plus facile à retrouver |

    **Ou en graphique** : `taskschd.msc` → clic droit sur **Bibliothèque du Planificateur de tâches** → **Créer une tâche** (pas « tâche de base », pour avoir toutes les options) :

    1. **Général** : nom `Sauvegarde du partage`, **Modifier l'utilisateur** → `SYSTEM`, cocher **Exécuter avec les autorisations maximales**
    2. **Déclencheurs** → **Nouveau** : **Tous les jours**, 02:00:00
    3. **Actions** → **Nouveau** : programme `powershell.exe`, arguments `-NoProfile -ExecutionPolicy Bypass -File C:\Scripts\Sauvegarde-Partage.ps1`
    4. **Paramètres** : cocher **Exécuter la tâche dès que possible si un démarrage planifié est manqué**

!!! warning "Un compte de domaine plutôt que SYSTEM ?"
    `SYSTEM` n'a pas de droits sur les **autres machines**. Si le script doit écrire sur un partage réseau (`\\srv-files01\sauvegardes`), lancer la tâche avec un compte de service du domaine `LAB` qui a les droits sur ce partage, et cocher **Exécuter même si l'utilisateur n'est pas connecté**.

## Étape 4 : forcer une exécution et lire le résultat

Pas besoin d'attendre 2 h du matin.

=== "Linux (cron)"

    Lancer le script **dans les conditions de cron** (environnement minimal, sans ton `PATH`) :

    ```bash
    sudo env -i /bin/sh -c '/usr/local/bin/sauvegarde-partage.sh'
    sudo tail -n 5 /var/log/sauvegarde-partage.log
    ```

    Pour tester cron lui-même, passer temporairement la ligne à `*/2 * * * *` (toutes les 2 minutes), puis regarder ses traces :

    ```bash
    journalctl -u cron --since "10 min ago"
    ```

    Remettre ensuite `0 2 * * *`.

=== "Windows (Planificateur de tâches)"

    ```powershell
    Start-ScheduledTask -TaskPath "\Lab\" -TaskName "Sauvegarde du partage"
    Start-Sleep 10
    Get-ScheduledTaskInfo -TaskPath "\Lab\" -TaskName "Sauvegarde du partage" |
        Select-Object LastRunTime, LastTaskResult, NextRunTime
    Get-Content D:\Sauvegardes\sauvegarde.log -Tail 5
    ```

    Dans `taskschd.msc`, l'onglet **Historique** de la tâche montre chaque lancement (activer **Activer l'historique de toutes les tâches** dans le volet de droite s'il est vide).

### Ce que tu vois / ce que ça veut dire / action

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `LastTaskResult` = `0` | La tâche s'est terminée sans erreur | Rien, vérifier quand même l'archive |
| `LastTaskResult` = `1` | Le script a fait `exit 1` (erreur gérée) | Lire `sauvegarde.log` : la ligne `ERREUR` dit pourquoi |
| `LastTaskResult` = `267009` (0x41301) | La tâche est **en cours** | Attendre et relancer `Get-ScheduledTaskInfo` |
| `LastTaskResult` = `267011` (0x41303) | La tâche n'a **jamais** été lancée | `Start-ScheduledTask` pour la tester |
| `LastTaskResult` = `2147942402` (0x80070002) | Fichier introuvable | Vérifier le chemin du `.ps1` dans l'action |
| Log Linux vide, archive absente | cron n'a pas lancé le script ou il plante avant le premier `log` | `journalctl -u cron`, relancer avec `env -i` |
| `Permission denied` dans le log Linux | Script non exécutable ou tâche dans la crontab d'un utilisateur | `chmod 750`, mettre la ligne dans `sudo crontab -e` |
| `command not found` uniquement sous cron | `PATH` réduit de cron | Chemins absolus (`/usr/bin/tar`) ou `PATH=...` en tête de crontab |

## Vérification finale

- [ ] Le script lancé à la main renvoie le code **0** et crée une archive datée
- [ ] Le log contient `Début` puis `Fin de la sauvegarde : OK`
- [ ] Une archive vieillie de 10 jours est supprimée au lancement suivant
- [ ] Linux : `sudo crontab -l` affiche la ligne `0 2 * * *`
- [ ] Windows : `Get-ScheduledTaskInfo` affiche un `NextRunTime` à 2 h 00 et `LastTaskResult` = `0`
- [ ] **Le lendemain matin**, une nouvelle archive est apparue sans que tu fasses rien
- [ ] Une archive a été **restaurée** pour test (`tar -xzf archive.tar.gz -C /tmp/test` ou `Expand-Archive archive.zip C:\Temp\test`)

## Dépannage courant

!!! warning "Pièges fréquents"
    - **« Ça marche à la main mais pas en tâche planifiée »** : presque toujours un **chemin relatif** ou une variable d'environnement absente. Tout mettre en chemins absolus.
    - **cron : le `%` est spécial** : dans une ligne de crontab, `date +%F` doit s'écrire `date +\%F`. Mettre la logique dans le script plutôt que dans la crontab.
    - **cron : la dernière ligne est ignorée** s'il n'y a pas de retour à la ligne à la fin du fichier.
    - **cron et le fuseau horaire** : cron suit l'heure du système. Vérifier avec `timedatectl` (et `sudo timedatectl set-timezone Europe/Paris`).
    - **Windows : la tâche reste « En cours d'exécution »** : le script attend une saisie (`Read-Host`, une confirmation). Une tâche planifiée ne doit **jamais** poser de question : ajouter `-Force` ou `-Confirm:$false` aux commandes concernées.
    - **Windows : `Compress-Archive` échoue** sur un fichier ouvert ou au-delà de 2 Go par fichier : pour de gros volumes, préférer `robocopy` ou la Sauvegarde Windows Server.
    - **Le disque se remplit quand même** : la rétention ne tourne que si le script va au bout. Surveiller l'espace (`df -h` / `Get-PSDrive`) et le log.

!!! tip "Variante Linux moderne : le timer systemd"
    Sur les distributions récentes, un **timer systemd** remplace avantageusement cron : logs dans `journalctl`, rattrapage si la machine était éteinte (`Persistent=true`). cron reste le plus simple et le plus répandu, c'est lui qu'on croise partout en entreprise et en examen.

## Pour aller plus loin

- 📋 [Mémo scripting Bash & PowerShell](memo-bash-powershell.md)
- 🧠 [Les sauvegardes et la règle 3-2-1](concept-sauvegardes.md) : une archive sur le même disque que la source ne compte pas comme une vraie sauvegarde
- 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](tuto-proxmox-backup.md) : les jobs planifiés côté hyperviseur

??? quote "Sources"
    - [Page de manuel crontab(5)](https://man7.org/linux/man-pages/man5/crontab.5.html)
    - [PowerShell : Register-ScheduledTask](https://learn.microsoft.com/powershell/module/scheduledtasks/register-scheduledtask)
    - [Microsoft Learn : schtasks](https://learn.microsoft.com/fr-fr/windows-server/administration/windows-commands/schtasks)
