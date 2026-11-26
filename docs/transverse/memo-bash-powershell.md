---
icon: material/clipboard-text-outline
title: Mémo scripting Bash & PowerShell
tags: [transverse, mémo, scripting, bash, powershell]
---

# 📋 Mémo scripting Bash & PowerShell

!!! abstract "Usage"
    Retrouver en 10 secondes la syntaxe d'une variable, d'une condition ou d'une boucle, côte à côte en **Bash** (Linux) et **PowerShell** (Windows). Pour mettre un script en tâche planifiée : 🛠️ [Automatiser une tâche avec cron et le Planificateur de tâches](tuto-taches-planifiees.md).

**Niveau :** 🟢 Bases → 🟡 Intermédiaire

!!! info "La grande différence à garder en tête"
    **Bash manipule du texte** : chaque commande renvoie des lignes qu'on découpe avec `grep`, `cut`, `awk`.
    **PowerShell manipule des objets** : `Get-Service` renvoie des objets avec des propriétés (`.Name`, `.Status`) qu'on filtre avec `Where-Object` et `Select-Object`.

## Le fichier de script

| Besoin | Bash | PowerShell |
|---|---|---|
| Extension | `script.sh` | `script.ps1` |
| Première ligne | `#!/bin/bash` | *(aucune obligatoire)* |
| Rendre exécutable / autoriser | `chmod +x script.sh` | `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` |
| Lancer | `./script.sh` ou `bash script.sh` | `.\script.ps1` ou `powershell -File script.ps1` |
| Commentaire | `# commentaire` | `# commentaire` · bloc `<# ... #>` |
| Arrêter à la première erreur | `set -euo pipefail` | `$ErrorActionPreference = "Stop"` |
| Afficher un message | `echo "Bonjour"` | `Write-Host "Bonjour"` (ou `Write-Output`) |
| Obtenir l'aide d'une commande | `man ls` · `ls --help` | `Get-Help Get-Service -Examples` |

## Variables

| Besoin | Bash | PowerShell |
|---|---|---|
| Créer | `nom="alice"` *(pas d'espace autour de `=`)* | `$nom = "alice"` |
| Lire | `echo "$nom"` | `Write-Host $nom` |
| Dans une chaîne | `"Bonjour $nom"` · `"${nom}_ok"` | `"Bonjour $nom"` · `"$($nom)_ok"` |
| Chaîne sans interprétation | `'$nom reste tel quel'` | `'$nom reste tel quel'` |
| Résultat d'une commande | `date_jour=$(date +%F)` | `$dateJour = Get-Date -Format "yyyy-MM-dd"` |
| Calcul | `total=$(( a + b ))` | `$total = $a + $b` |
| Variable d'environnement | `$HOME` · `export VAR=1` | `$env:USERPROFILE` · `$env:VAR = 1` |
| Arguments du script | `$1`, `$2` · tous : `"$@"` · nombre : `$#` | `$args[0]` ou un bloc `param($Chemin)` |
| Code retour de la dernière commande | `$?` (0 = OK) | `$?` (True/False) · `$LASTEXITCODE` pour un `.exe` |
| Demander une saisie | `read -p "Nom ? " nom` | `$nom = Read-Host "Nom ?"` |

## Tableaux

| Besoin | Bash | PowerShell |
|---|---|---|
| Créer | `serveurs=(srv-web01 srv-web02)` | `$serveurs = @("srv-web01", "srv-web02")` |
| Premier élément | `${serveurs[0]}` | `$serveurs[0]` |
| Tous les éléments | `"${serveurs[@]}"` | `$serveurs` |
| Nombre d'éléments | `${#serveurs[@]}` | `$serveurs.Count` |
| Ajouter | `serveurs+=(srv-lb01)` | `$serveurs += "srv-lb01"` |

## Conditions

=== "Bash"

    ```bash
    if [ -f /etc/hosts ]; then
        echo "Le fichier existe"
    elif [ -d /etc ]; then
        echo "C'est un dossier"
    else
        echo "Introuvable"
    fi
    ```

=== "PowerShell"

    ```powershell
    if (Test-Path C:\Windows\System32\drivers\etc\hosts -PathType Leaf) {
        Write-Host "Le fichier existe"
    } elseif (Test-Path C:\Windows -PathType Container) {
        Write-Host "C'est un dossier"
    } else {
        Write-Host "Introuvable"
    }
    ```

| Test | Bash | PowerShell |
|---|---|---|
| Égal / différent (texte) | `[ "$a" = "$b" ]` · `!=` | `$a -eq $b` · `-ne` |
| Plus grand / plus petit (nombres) | `-gt` `-ge` `-lt` `-le` | `-gt` `-ge` `-lt` `-le` |
| Contient un motif | `[[ "$a" == *web* ]]` | `$a -like "*web*"` · regex : `-match` |
| Chaîne vide | `[ -z "$a" ]` | `[string]::IsNullOrEmpty($a)` |
| Fichier existe | `[ -f fichier ]` | `Test-Path fichier -PathType Leaf` |
| Dossier existe | `[ -d dossier ]` | `Test-Path dossier -PathType Container` |
| ET / OU / NON | `&&` · <code>&#124;&#124;</code> · `!` | `-and` · `-or` · `-not` |
| La commande a réussi | `if ping -c1 10.0.0.10 >/dev/null; then` | `if (Test-Connection 10.0.0.10 -Count 1 -Quiet) {` |

!!! tip "Choix multiple"
    Bash : `case "$choix" in start) ... ;; stop) ... ;; *) ... ;; esac`
    PowerShell : `switch ($choix) { "start" { ... } "stop" { ... } default { ... } }`

## Boucles

| Besoin | Bash | PowerShell |
|---|---|---|
| Sur une liste | `for s in srv-web01 srv-web02; do echo "$s"; done` | `foreach ($s in $serveurs) { Write-Host $s }` |
| De 1 à 5 | `for i in {1..5}; do echo "$i"; done` | `foreach ($i in 1..5) { Write-Host $i }` |
| Sur des fichiers | `for f in /var/log/*.log; do echo "$f"; done` | <code>Get-ChildItem C:&#92;Logs&#92;*.log &#124; ForEach-Object { $_.Name }</code> |
| Ligne par ligne d'un fichier | `while read -r ligne; do echo "$ligne"; done < liste.txt` | <code>Get-Content liste.txt &#124; ForEach-Object { $_ }</code> |
| Tant que | `while [ "$n" -lt 3 ]; do n=$((n+1)); done` | `while ($n -lt 3) { $n++ }` |
| Sortir / passer au suivant | `break` · `continue` | `break` · `continue` |

!!! example "Le même script dans les deux langages : vérifier que les serveurs répondent"

    === "Bash"

        ```bash
        #!/bin/bash
        serveurs=(10.0.0.10 10.0.0.20 10.0.0.31 10.0.0.32)

        for ip in "${serveurs[@]}"; do
            if ping -c 1 -W 1 "$ip" > /dev/null; then
                echo "OK   $ip"
            else
                echo "KO   $ip"
            fi
        done
        ```

    === "PowerShell"

        ```powershell
        $serveurs = @("10.0.0.10", "10.0.0.20", "10.0.0.31", "10.0.0.32")

        foreach ($ip in $serveurs) {
            if (Test-Connection $ip -Count 1 -Quiet) {
                Write-Host "OK   $ip" -ForegroundColor Green
            } else {
                Write-Host "KO   $ip" -ForegroundColor Red
            }
        }
        ```

## Fonctions

=== "Bash"

    ```bash
    saluer() {
        local nom="$1"
        echo "Bonjour $nom"
    }
    saluer alice
    ```

=== "PowerShell"

    ```powershell
    function Saluer {
        param([string]$Nom)
        Write-Host "Bonjour $Nom"
    }
    Saluer -Nom alice
    ```

## Entrées, sorties et redirections

| Besoin | Bash | PowerShell |
|---|---|---|
| Écrire dans un fichier (écrase) | `echo "texte" > fichier.txt` | <code>"texte" &#124; Out-File fichier.txt</code> *(ou `>`)* |
| Ajouter à la fin | `echo "texte" >> fichier.txt` | <code>"texte" &#124; Add-Content fichier.txt</code> |
| Lire un fichier | `cat fichier.txt` | `Get-Content fichier.txt` |
| Ignorer la sortie | `commande > /dev/null 2>&1` | <code>commande &#124; Out-Null</code> |
| Rediriger les erreurs | `2> erreurs.log` | `2> erreurs.log` |
| Chaîner (pipe) | <code>cmd1 &#124; cmd2</code> *(passe du texte)* | <code>cmd1 &#124; cmd2</code> *(passe des objets)* |
| Filtrer | `grep "erreur" app.log` | `Select-String "erreur" app.log` · `Where-Object { $_.Status -eq "Running" }` |
| Garder certaines colonnes | `cut -d: -f1 /etc/passwd` · `awk '{print $1}'` | `Select-Object Name, Status` |
| Compter | `wc -l` | `Measure-Object` · `.Count` |
| Trier | `sort` · `sort -u` | `Sort-Object` · `Sort-Object -Unique` |
| Exporter en CSV | *(à la main avec `echo`)* | `Export-Csv rapport.csv -NoTypeInformation` |

## Gestion d'erreurs

=== "Bash"

    ```bash
    set -euo pipefail          # stop à la première erreur

    if ! cp source.txt /sauvegardes/; then
        echo "Échec de la copie" >&2
        exit 1                 # code retour ≠ 0 = échec
    fi
    ```

=== "PowerShell"

    ```powershell
    $ErrorActionPreference = "Stop"

    try {
        Copy-Item source.txt D:\Sauvegardes\
    } catch {
        Write-Error "Échec de la copie : $_"
        exit 1
    }
    ```

## Commandes d'admin utiles en script

| Besoin | Bash (Linux) | PowerShell (Windows) |
|---|---|---|
| Date du jour pour un nom de fichier | `$(date +%F_%H%M)` | `Get-Date -Format "yyyy-MM-dd_HHmm"` |
| Archiver un dossier | `tar -czf archive.tar.gz /srv/partage` | `Compress-Archive C:\Partage archive.zip` |
| Supprimer les fichiers de + de 7 jours | `find /sauvegardes -name "*.tar.gz" -mtime +7 -delete` | <code>Get-ChildItem D:&#92;Sauvegardes&#92;*.zip &#124; Where-Object LastWriteTime -lt (Get-Date).AddDays(-7) &#124; Remove-Item</code> |
| Espace disque | `df -h` | `Get-PSDrive -PSProvider FileSystem` |
| État d'un service | `systemctl is-active nginx` | `(Get-Service Spooler).Status` |
| Redémarrer un service | `sudo systemctl restart nginx` | `Restart-Service Spooler` |
| Liste des utilisateurs | `cut -d: -f1 /etc/passwd` | `Get-LocalUser` · AD : `Get-ADUser -Filter *` |
| Créer un utilisateur | `sudo useradd -m bruno` | `New-LocalUser bruno` · AD : `New-ADUser` |
| Exécuter sur une machine distante | `ssh admin@10.0.0.31 "uptime"` | `Invoke-Command -ComputerName srv-dc01 -ScriptBlock { Get-Service }` |
| Écrire dans le journal système | `logger "Sauvegarde OK"` | `Write-EventLog` *(PS 5)* ou fichier de log |

!!! warning "Pièges fréquents"
    - **Bash : espaces autour du `=`** : `nom = alice` est lu comme la commande `nom`. Écrire `nom=alice`.
    - **Bash : variables sans guillemets** : `rm $fichier` casse si le nom contient un espace. Toujours `"$fichier"`.
    - **Bash : `[ ]` exige des espaces** : `[ "$a" = "$b" ]`, pas `["$a"="$b"]`.
    - **PowerShell : `=` n'est pas une comparaison** : `if ($a = 1)` affecte. Utiliser `-eq`.
    - **PowerShell : `>` et `<` ne comparent pas**, ce sont des redirections : utiliser `-gt` / `-lt`.
    - **PowerShell : script bloqué** (« l'exécution de scripts est désactivée ») : `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser`.
    - **Fins de ligne Windows** : un `.sh` écrit sous Windows (CRLF) donne `$'\r': command not found`. Corriger avec `sed -i 's/\r$//' script.sh`.

## Pour aller plus loin

- 🛠️ [Automatiser une tâche avec cron et le Planificateur de tâches](tuto-taches-planifiees.md)
- 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](tuto-proxmox-backup.md)
- 🛠️ [Gérer utilisateurs, groupes et permissions sous Linux](../systeme/tuto-permissions-linux.md)
