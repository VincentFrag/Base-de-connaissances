---
icon: material/clipboard-text-outline
title: Mémo commandes du quotidien
tags: [bonnes pratiques, mémo, commandes, linux, windows, powershell]
---

# 📋 Mémo commandes du quotidien

!!! abstract "Usage"
    Les commandes qu'un admin tape tous les jours pour savoir **comment va une machine**, classées par besoin. Pour le réseau, voir 📋 [Mémo commandes réseau](../reseau/memo-commandes-reseau.md) ; pour les ports, 📋 [Mémo ports et protocoles](../references/memo-ports-protocoles.md).
    Côté Windows, les commandes sont en PowerShell (à lancer en administrateur quand il faut des droits).

## Où suis-je, qui suis-je ?

| Besoin | Windows | Linux |
|---|---|---|
| Nom de la machine | `hostname` | `hostname` · `hostnamectl` |
| Version du système | `Get-ComputerInfo OsName, OsVersion` · `winver` | `cat /etc/os-release` · `uname -r` |
| Mon compte et mes groupes | `whoami /all` | `id` · `whoami` |
| Depuis quand la machine tourne | `(Get-CimInstance Win32_OperatingSystem).LastBootUpTime` | `uptime` |
| Qui est connecté | `query user` | `who` · `w` |
| Dernières connexions | `Get-WinEvent -FilterHashtable @{LogName='Security';Id=4624} -MaxEvents 10` | `last -n 10` |

## Ressources : processeur, mémoire, disque

| Besoin | Windows | Linux |
|---|---|---|
| Vue d'ensemble en direct | `taskmgr` · `resmon` | `top` · `htop` |
| Mémoire libre | `Get-CimInstance Win32_OperatingSystem \| Select FreePhysicalMemory, TotalVisibleMemorySize` | `free -h` |
| Espace disque | `Get-Volume` · `Get-PSDrive -PSProvider FileSystem` | `df -h` |
| Qui prend la place dans un dossier | `Get-ChildItem C:\Dossier -Recurse \| Measure-Object Length -Sum` | `sudo du -sh /var/* \| sort -h` |
| Plus gros fichiers | `Get-ChildItem C:\ -Recurse -File -EA 0 \| Sort Length -Desc \| Select -First 10 FullName, Length` | `sudo find / -xdev -type f -size +500M -exec ls -lh {} +` |
| Disques et partitions | `Get-Disk` · `Get-Partition` | `lsblk` · `sudo fdisk -l` |

## Processus

| Besoin | Windows | Linux |
|---|---|---|
| Lister les processus | `Get-Process` · `tasklist` | `ps aux` |
| Les plus gourmands | `Get-Process \| Sort CPU -Desc \| Select -First 10` | `ps aux --sort=-%cpu \| head` · `ps aux --sort=-%mem \| head` |
| Trouver un processus | `Get-Process -Name *nginx*` | `pgrep -a nginx` |
| Arrêter un processus | `Stop-Process -Name notepad` · `taskkill /PID 1234 /F` | `kill 1234` · `kill -9 1234` (en dernier recours) |

## Services

| Besoin | Windows | Linux (systemd) |
|---|---|---|
| État d'un service | `Get-Service Spooler` | `systemctl status nginx` |
| Démarrer / arrêter / redémarrer | `Start-Service` · `Stop-Service` · `Restart-Service Spooler` | `sudo systemctl start\|stop\|restart nginx` |
| Recharger la conf sans couper | (selon le service) | `sudo systemctl reload nginx` |
| Activer au démarrage | `Set-Service Spooler -StartupType Automatic` | `sudo systemctl enable --now nginx` |
| Services en échec | `Get-Service \| Where Status -eq Stopped \| Where StartType -eq Automatic` | `systemctl --failed` |
| Tester une conf avant de recharger | — | `sudo nginx -t` · `sudo apachectl configtest` · `sudo sshd -t` |

## Journaux (logs)

| Besoin | Windows | Linux |
|---|---|---|
| Journal d'un service | `Get-WinEvent -LogName System -MaxEvents 20` | `journalctl -u nginx -n 50` |
| Suivre en direct | — | `journalctl -fu nginx` · `tail -f /var/log/syslog` |
| Erreurs depuis le dernier démarrage | `Get-WinEvent -FilterHashtable @{LogName='System';Level=2} -MaxEvents 20` | `journalctl -b -p err` |
| Depuis une heure précise | `Get-WinEvent -FilterHashtable @{LogName='System';StartTime=(Get-Date).AddHours(-1)}` | `journalctl --since "1 hour ago"` |
| Interface graphique | `eventvwr.msc` | — |

## Utilisateurs et groupes

| Besoin | Windows (AD) | Linux |
|---|---|---|
| Infos sur un compte | `Get-ADUser alice -Properties *` | `id alice` · `getent passwd alice` |
| Membres d'un groupe | `Get-ADGroupMember GG_Compta` | `getent group compta` |
| Créer un compte | `New-ADUser -Name alice ...` | `sudo adduser alice` |
| Ajouter à un groupe | `Add-ADGroupMember GG_Compta -Members alice` | `sudo usermod -aG compta alice` |
| Désactiver un compte | `Disable-ADAccount alice` | `sudo usermod -L alice` · `sudo chage -E 0 alice` |
| Comptes verrouillés | `Search-ADAccount -LockedOut` | `sudo faillock --user alice` |
| Comptes inactifs depuis 90 jours | `Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 -UsersOnly` | `sudo lastlog -b 90` |

Pour le détail des droits Linux : 🛠️ [Gérer utilisateurs, groupes et permissions sous Linux](../systeme/tuto-permissions-linux.md).

## Mises à jour et paquets

| Besoin | Windows | Linux (Debian) |
|---|---|---|
| Chercher les mises à jour | Paramètres → Windows Update · `Get-WindowsUpdate` (module PSWindowsUpdate) | `sudo apt update && apt list --upgradable` |
| Installer les mises à jour | `Install-WindowsUpdate -AcceptAll` | `sudo apt full-upgrade` |
| Installer un logiciel | `winget install <nom>` | `sudo apt install <paquet>` |
| Logiciels installés | `winget list` · `Get-Package` | `apt list --installed` · `dpkg -l` |
| Redémarrage nécessaire ? | `Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'` | `ls /var/run/reboot-required` |

!!! tip "Sur Rocky Linux / RHEL"
    Remplacer `apt` par `dnf` : `sudo dnf check-update`, `sudo dnf upgrade`, `sudo dnf install <paquet>`, `needs-restarting -r`.

## Fichiers et configuration

| Besoin | Windows | Linux |
|---|---|---|
| Sauvegarder une conf avant de la modifier | `Copy-Item fichier.conf "fichier.conf.bak-$(Get-Date -f yyyy-MM-dd)"` | `sudo cp -a fichier.conf fichier.conf.bak-$(date +%F)` |
| Comparer deux versions | `Compare-Object (gc a.conf) (gc b.conf)` | `diff -u a.conf b.conf` |
| Chercher un texte dans des fichiers | `Select-String -Path C:\Logs\*.log -Pattern "error"` | `grep -ri "error" /var/log/` |
| Voir la fin d'un fichier | `Get-Content fichier.log -Tail 20 -Wait` | `tail -n 20 -f fichier.log` |
| Fichiers modifiés récemment | `Get-ChildItem -Recurse \| Where LastWriteTime -gt (Get-Date).AddDays(-1)` | `find /etc -mtime -1` |
| Droits d'un fichier | `Get-Acl fichier \| Format-List` · `icacls fichier` | `ls -l fichier` · `stat fichier` |

## Arrêt, redémarrage, accès distant

| Besoin | Windows | Linux |
|---|---|---|
| Redémarrer | `Restart-Computer` · `shutdown /r /t 0` | `sudo systemctl reboot` |
| Éteindre | `Stop-Computer` · `shutdown /s /t 0` | `sudo systemctl poweroff` |
| Programmer un redémarrage | `shutdown /r /t 3600` (dans 1 h) · annuler `shutdown /a` | `sudo shutdown -r 22:00` · annuler `sudo shutdown -c` |
| Ouvrir une session distante | `Enter-PSSession srv-dc01` · `mstsc /v:srv-dc01` | `ssh admin@srv-web01` |
| Lancer une commande à distance | `Invoke-Command srv-dc01 { Get-Service DNS }` | `ssh admin@srv-web01 'systemctl status nginx'` |

## Proxmox (sur `pve01`)

| Besoin | Commande |
|---|---|
| Lister les VM / conteneurs | `qm list` · `pct list` |
| Démarrer / arrêter proprement | `qm start 100` · `qm shutdown 100` |
| Snapshot avant une modif | `qm snapshot 100 avant-modif` |
| Lister / supprimer les snapshots | `qm listsnapshot 100` · `qm delsnapshot 100 avant-modif` |
| Sauvegarde immédiate | `vzdump 100 --mode snapshot --storage local` |
| État des stockages | `pvesm status` |

!!! info "Pour écrire des scripts"
    Variables, conditions, boucles et fonctions en Bash et PowerShell côte à côte : 📋 [Mémo Bash et PowerShell](../transverse/memo-bash-powershell.md). Pour lancer un script à heure fixe : 🛠️ [Planifier une tâche](../transverse/tuto-taches-planifiees.md).
