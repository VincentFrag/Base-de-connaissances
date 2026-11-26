---
icon: material/clipboard-text-outline
title: Mémo commandes Linux et PowerShell
tags: [référence, mémo, commandes, linux, powershell, bash]
---

# 📋 Mémo commandes Linux et PowerShell

!!! abstract "Usage"
    Les commandes qu'on tape tout le temps (`curl`, `grep`, `sed -i`, `find`, `tar`, `ssh`…), **avec leurs options utiles expliquées**, un exemple concret et l'équivalent de l'autre côté (Linux ⇄ PowerShell).

    Cette page part de la **commande** (« que fait `sed -i` ? »). Les deux pages sœurs partent d'ailleurs :

    - 📋 [Mémo commandes du quotidien](../pratiques/memo-commandes-quotidien.md) part du **besoin** (« voir l'espace disque », « redémarrer un service »).
    - 📋 [Mémo scripting Bash & PowerShell](../transverse/memo-bash-powershell.md) donne la **syntaxe de script** (variables, conditions, boucles).

**Niveau :** 🟢 Bases → 🟡 Intermédiaire

## Lire une commande

```text
grep  -r -i  "error"   /var/log/
│     │      │         └── argument : où chercher
│     │      └── argument : quoi chercher
│     └── options : -r (récursif) et -i (ignorer la casse)
└── la commande
```

| Notion | Linux | PowerShell |
|---|---|---|
| Option courte / longue | `-i` = `--ignore-case` | paramètre nommé : `-Path`, `-Recurse` |
| Combiner des options courtes | `ls -lah` = `ls -l -a -h` | non : chaque paramètre s'écrit à part |
| Abréger | non (sauf options longues uniques) | oui tant que c'est sans ambiguïté : `-Rec` pour `-Recurse` |
| Casse | **sensible** : `-r` ≠ `-R` | insensible : `get-childitem` marche |
| Aide | `man grep` · `grep --help` · `tldr grep` | `Get-Help Select-String -Examples` |
| Trouver une commande | `apropos archive` | `Get-Command *archive*` |

!!! tip "Les alias PowerShell"
    PowerShell connaît des alias aux noms Linux : `ls`, `cat`, `cp`, `mv`, `rm`, `pwd`, `ps`, `kill`. **Ce sont des cmdlets déguisées** : `ls` lance `Get-ChildItem`, donc `ls -la` échoue. Pour voir ce qui se cache derrière : `Get-Alias ls`.

## Correspondance rapide

| Linux | PowerShell | Alias PS | Rôle |
|---|---|---|---|
| `ls` | `Get-ChildItem` | `ls`, `dir`, `gci` | Lister |
| `cd` · `pwd` | `Set-Location` · `Get-Location` | `cd` · `pwd` | Se déplacer, savoir où on est |
| `cp` · `mv` · `rm` | `Copy-Item` · `Move-Item` · `Remove-Item` | `cp` · `mv` · `rm` | Copier, déplacer, supprimer |
| `mkdir` · `touch` | `New-Item -ItemType Directory` · `New-Item -ItemType File` | `mkdir` · `ni` | Créer |
| `cat` · `head` · `tail` | `Get-Content` (`-TotalCount`, `-Tail`) | `cat`, `gc` | Lire un fichier |
| `grep` | `Select-String` | `sls` | Chercher du texte |
| `sed` | `-replace` + `Set-Content` | — | Remplacer du texte |
| `awk` · `cut` | `Select-Object`, `.Split()` | `select` | Extraire des colonnes |
| `sort` · `uniq` · `wc` | `Sort-Object` · `-Unique` · `Measure-Object` | `sort` · — · `measure` | Trier, dédoublonner, compter |
| `find` | `Get-ChildItem -Recurse` + `Where-Object` | `gci`, `?` | Chercher des fichiers |
| `tar` · `zip` | `Compress-Archive` · `Expand-Archive` | — | Archiver |
| `chmod` · `chown` | `icacls` · `Set-Acl` | — | Droits |
| `curl` · `wget` | `Invoke-WebRequest` · `Invoke-RestMethod` | `iwr` · `irm` | Requêtes web, téléchargement |
| `ip a` · `ip r` | `Get-NetIPAddress` · `Get-NetRoute` | — | Configuration IP |
| `ss` | `Get-NetTCPConnection` | — | Ports ouverts, connexions |
| `nc -zv` | `Test-NetConnection -Port` | `tnc` | Tester un port |
| `dig` | `Resolve-DnsName` | — | Interroger le DNS |
| `ssh` · `scp` | `ssh` · `scp` (OpenSSH intégré) · `Enter-PSSession` | — | Accès distant |
| `systemctl` | `Get-Service`, `Restart-Service`… | `gsv` | Services |
| `journalctl` | `Get-WinEvent` | — | Journaux |
| `ps` · `kill` | `Get-Process` · `Stop-Process` | `ps` · `kill` | Processus |

## Fichiers et dossiers

### `ls` : lister

| Option | Effet |
|---|---|
| `-l` | Format long : droits, propriétaire, taille, date |
| `-a` | Affiche aussi les fichiers cachés (qui commencent par `.`) |
| `-h` | Tailles lisibles (`4,0K`, `12M`) avec `-l` |
| `-t` · `-r` | Trie par date (plus récent en haut) · inverse l'ordre |
| `-R` | Récursif (descend dans les sous-dossiers) |

```bash
ls -lah /etc/nginx/          # tout, en détail, tailles lisibles
ls -ltr /var/log/ | tail     # les derniers fichiers modifiés en bas
```

**PowerShell :** `Get-ChildItem C:\Logs -Force` (`-Force` = fichiers cachés) · `Get-ChildItem C:\Logs | Sort-Object LastWriteTime`

### `cp`, `mv`, `rm`, `mkdir`

| Commande et option | Effet |
|---|---|
| `cp -r` | Copie un dossier et son contenu |
| `cp -a` | Copie **à l'identique** (droits, propriétaire, dates) : idéal pour une sauvegarde de conf |
| `cp -i` · `mv -i` | Demande avant d'écraser |
| `mv a b` | Déplace **ou renomme** |
| `rm -r` | Supprime un dossier et son contenu |
| `rm -f` | Force, sans confirmation ni erreur si absent |
| `mkdir -p` | Crée toute l'arborescence, sans erreur si elle existe |

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%F)
mkdir -p /srv/partage/{compta,rh,direction}   # 3 dossiers d'un coup
rm -ri ancien-dossier/                         # supprime en demandant à chaque fichier
```

**PowerShell :**

```powershell
Copy-Item C:\Partage D:\Sauvegarde -Recurse
Move-Item rapport.txt rapport-2026.txt
Remove-Item C:\Temp\vieux -Recurse -Force
New-Item -ItemType Directory C:\Partage\Compta -Force   # -Force = comme mkdir -p
```

!!! danger "`rm -rf` ne pardonne pas"
    Pas de corbeille sous Linux. Avant un `rm -rf "$dossier"/`, vérifie que la variable n'est pas vide (sinon c'est `rm -rf /`). Côté PowerShell, ajoute `-WhatIf` pour voir ce qui **serait** supprimé : `Remove-Item C:\Temp\* -Recurse -WhatIf`.

### `find` : chercher des fichiers

| Option | Effet |
|---|---|
| `-name "*.log"` · `-iname` | Par nom (motif entre guillemets) · sans tenir compte de la casse |
| `-type f` · `-type d` | Seulement les fichiers · seulement les dossiers |
| `-mtime -1` · `-mtime +7` | Modifié il y a moins d'un jour · il y a plus de 7 jours |
| `-size +100M` | Plus gros que 100 Mo |
| `-user alice` | Appartenant à alice |
| `-maxdepth 2` | Ne descend pas plus de 2 niveaux |
| `-xdev` | Reste sur le même disque (évite `/proc`, les partages montés) |
| `-exec cmd {} \;` · `-exec cmd {} +` | Lance une commande sur chaque résultat · sur tous d'un coup (plus rapide) |
| `-delete` | Supprime les résultats (à tester **sans** d'abord) |

```bash
find /etc -name "*.conf" -mtime -1               # confs modifiées depuis hier
sudo find / -xdev -type f -size +500M -exec ls -lh {} +
find /sauvegardes -name "*.tar.gz" -mtime +7 -delete   # rétention 7 jours
find /srv/partage -type d -exec chmod 2770 {} \;
```

**PowerShell :**

```powershell
Get-ChildItem C:\ -Recurse -Filter *.log -ErrorAction SilentlyContinue
Get-ChildItem D:\Sauvegardes -File | Where-Object LastWriteTime -lt (Get-Date).AddDays(-7) | Remove-Item
Get-ChildItem C:\ -Recurse -File -EA 0 | Where-Object Length -gt 500MB
```

## Lire et traiter du texte

### `cat`, `less`, `head`, `tail`

| Commande et option | Effet |
|---|---|
| `cat -n` | Affiche avec les numéros de ligne |
| `less` | Lecture page par page : `/mot` cherche, `n` suivant, `G` fin, `q` quitte |
| `head -n 20` | Les 20 premières lignes |
| `tail -n 50` | Les 50 dernières lignes |
| `tail -f` | Suit le fichier **en direct** (Ctrl+C pour sortir) |

```bash
tail -f /var/log/nginx/access.log
grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'   # la conf sans commentaires ni lignes vides
```

**PowerShell :** `Get-Content fichier.log -TotalCount 20` (début) · `Get-Content fichier.log -Tail 50 -Wait` (fin, en direct)

### `grep` : chercher du texte

| Option | Effet |
|---|---|
| `-i` | Ignore majuscules / minuscules |
| `-r` · `-R` | Cherche dans tous les fichiers d'un dossier (`-R` suit aussi les liens) |
| `-n` | Affiche le numéro de ligne |
| `-v` | **Inverse** : les lignes qui ne contiennent **pas** le motif |
| `-l` | N'affiche que le nom des fichiers qui contiennent le motif |
| `-c` | Compte les lignes trouvées |
| `-w` | Mot entier (`-w ip` ne trouve pas `zip`) |
| `-E` | Expression régulière étendue : <code>-E "error&#124;fail"</code> trouve error **ou** fail |
| `-A 3` · `-B 3` · `-C 3` | 3 lignes après · avant · autour du résultat |
| `-o` | N'affiche que la partie trouvée |

```bash
grep -rin "listen" /etc/nginx/                  # où est défini le port d'écoute ?
grep -c "Failed password" /var/log/auth.log     # combien d'échecs SSH ?
grep -E "error|warn" -i /var/log/syslog | tail
ps aux | grep -v grep | grep nginx
```

**PowerShell :**

```powershell
Select-String -Path C:\Logs\*.log -Pattern "error"                 # -i par défaut
Select-String -Path .\app.log -Pattern "error" -CaseSensitive
Select-String -Path .\app.log -Pattern "error" -NotMatch           # = grep -v
Get-ChildItem C:\inetpub -Recurse -Filter *.config | Select-String "connectionString" -List   # = grep -rl
Select-String -Path .\app.log -Pattern "error" -Context 2,3        # = grep -B2 -A3
```

### `sed` : remplacer dans un fichier

`sed` lit un texte ligne par ligne et le transforme. L'usage n° 1 : **remplacer une valeur dans un fichier de conf** sans l'ouvrir.

| Option ou commande | Effet |
|---|---|
| `s/ancien/nouveau/` | Remplace la **première** occurrence de chaque ligne |
| `s/ancien/nouveau/g` | Remplace **toutes** les occurrences (*global*) |
| `-i` | Modifie le fichier **en place** (sinon `sed` affiche seulement le résultat) |
| `-i.bak` | Modifie en place **et** garde une copie `fichier.bak` |
| `-n '10,20p'` | Affiche seulement les lignes 10 à 20 |
| `/motif/d` | Supprime les lignes qui contiennent le motif |
| <code>s&#124;a&#124;b&#124;</code> | Autre séparateur, pratique quand le texte contient des `/` |

```bash
# 1. Tester sans -i : on voit le résultat, le fichier ne change pas
sed 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config | grep Port

# 2. Appliquer en gardant une sauvegarde
sudo sed -i.bak 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config

sed -i 's|/var/www/html|/srv/site|g' site.conf       # chemins : séparateur |
sed -i '/^#/d' fichier.conf                          # supprime les commentaires
sed -i 's/\r$//' script.sh                           # fins de ligne Windows → Linux
sed -n '120,140p' /var/log/syslog                    # extrait de lignes
```

**PowerShell :** pas de `sed`, on lit, on remplace avec `-replace` (expression régulière), on réécrit.

```powershell
(Get-Content .\site.conf) -replace 'C:\\inetpub', 'D:\\Sites' | Set-Content .\site.conf
(Get-Content .\app.config -Raw).Replace('localhost', 'srv-web01') | Set-Content .\app.config   # .Replace() = texte brut, sans regex
```

!!! warning "`sed -i` sur macOS / BSD"
    Sur macOS, `-i` exige un argument : `sed -i '' 's/a/b/' fichier`. Sous Linux (GNU), `sed -i 's/a/b/' fichier` suffit.

### `awk` et `cut` : extraire des colonnes

`awk` découpe chaque ligne en champs (`$1`, `$2`… ; `$NF` = dernier champ) séparés par des espaces.

| Commande et option | Effet |
|---|---|
| `awk '{print $1}'` | Affiche la 1re colonne |
| `awk -F: '{print $1}'` | Séparateur `:` au lieu de l'espace |
| `awk '$3 > 80'` | Ne garde que les lignes où la 3e colonne dépasse 80 |
| `awk '{s+=$2} END {print s}'` | Fait la somme de la 2e colonne |
| `cut -d: -f1,7` | Colonnes 1 et 7, séparateur `:` (plus simple qu'`awk` pour un séparateur fixe) |

```bash
awk -F: '$3 >= 1000 {print $1}' /etc/passwd           # comptes humains (UID ≥ 1000)
df -h | awk 'NR>1 && $5+0 > 80 {print $6, $5}'        # partitions remplies à plus de 80 %
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head   # top 10 des IP clientes
cut -d: -f1 /etc/passwd
```

**PowerShell :** les objets ont déjà des colonnes, on les choisit par leur nom.

```powershell
Get-Process | Select-Object Name, CPU | Sort-Object CPU -Descending | Select-Object -First 10
Get-Content .\export.csv | ForEach-Object { $_.Split(';')[0] }   # 1re colonne d'un texte
Import-Csv .\export.csv -Delimiter ';' | Select-Object Nom, Service   # mieux : lire le CSV en objets
```

### `sort`, `uniq`, `wc`

| Commande et option | Effet |
|---|---|
| `sort` · `sort -r` | Trie · à l'envers |
| `sort -n` · `sort -h` | Trie des nombres · des tailles (`2K`, `1G`) |
| `sort -k2` | Trie sur la 2e colonne |
| `uniq` · `uniq -c` | Supprime les doublons **qui se suivent** (trier avant) · compte chaque ligne |
| `wc -l` | Compte les lignes |

```bash
sudo du -sh /var/* | sort -h          # dossiers du plus petit au plus gros
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

**PowerShell :** `Sort-Object Length -Descending` · `Sort-Object -Unique` · `Group-Object` (= `uniq -c`) · `(Get-Content f.txt).Count` ou `Measure-Object -Line`

## Archives

### `tar`

| Option | Effet |
|---|---|
| `-c` | **Crée** une archive |
| `-x` | **Extrait** une archive |
| `-t` | Liste le contenu sans extraire |
| `-z` | Compresse / décompresse en gzip (`.tar.gz`) |
| `-J` | En xz (`.tar.xz`), plus compact, plus lent |
| `-f fichier` | Nom de l'archive (toujours en dernier dans le groupe d'options) |
| `-v` | Affiche les fichiers traités |
| `-C dossier` | Extrait dans ce dossier |
| `-p` | Conserve les droits à l'extraction |

Moyen mnémotechnique : **c**rée **z**ippé **f**ichier → `czf` ; e**x**trait **z**ippé **f**ichier → `xzf`.

```bash
tar -czf conf-nginx-$(date +%F).tar.gz /etc/nginx/
tar -tzf conf-nginx-2026-11-26.tar.gz | head
sudo tar -xzpf conf-nginx-2026-11-26.tar.gz -C /tmp/restauration/
```

**PowerShell :**

```powershell
Compress-Archive -Path C:\Partage\Compta -DestinationPath D:\compta.zip
Expand-Archive D:\compta.zip -DestinationPath C:\Restauration
tar -xzf archive.tar.gz      # tar.exe est fourni avec Windows 10 / 11 et Server 2019+
```

## Droits

### `chmod` et `chown`

| Commande et option | Effet |
|---|---|
| `chmod 640 fichier` | Notation octale : propriétaire `rw-` (6), groupe `r--` (4), autres `---` (0) |
| `chmod u+x script.sh` | Notation symbolique : ajoute l'exécution au propriétaire (`u`, `g`, `o`, `a` ; `+`, `-`, `=`) |
| `chmod -R` | Récursif |
| `chmod 2770 dossier` | Le `2` (setgid) : les fichiers créés héritent du groupe du dossier |
| `chown alice:compta fichier` | Change propriétaire **et** groupe |
| `chown -R www-data: /srv/site` | Récursif ; `user:` = groupe principal de l'utilisateur |

```bash
chmod +x sauvegarde.sh
sudo chown -R admin:compta /srv/partage/compta && sudo chmod -R 2770 /srv/partage/compta
chmod 600 ~/.ssh/id_ed25519           # SSH refuse une clé privée lisible par d'autres
```

**PowerShell / Windows :**

```powershell
icacls D:\Partage\Compta                                   # voir les droits
icacls D:\Partage\Compta /grant "LAB\GG_Compta:(OI)(CI)M"  # Modifier, hérité par sous-dossiers et fichiers
icacls D:\Partage\Compta /inheritance:d                    # couper l'héritage en gardant une copie
```

Pour comprendre `rwx` et l'octal : 🧠 [Comptes et droits Linux](../systeme/concept-permissions-linux.md).

## Réseau et web

### `curl` : parler à un serveur web

| Option | Effet |
|---|---|
| `-I` | Seulement les **en-têtes** de la réponse (code HTTP, serveur) |
| `-i` | En-têtes **et** contenu |
| `-v` | Mode bavard : résolution, connexion, TLS, en-têtes envoyés et reçus |
| `-s` | Silencieux (pas de barre de progression) |
| `-o fichier` · `-O` | Enregistre dans `fichier` · sous le nom d'origine |
| `-L` | Suit les redirections (301, 302) |
| `-k` | Accepte un certificat non valide (lab, auto-signé) |
| `-X POST` | Méthode HTTP (GET par défaut) |
| `-H "Nom: valeur"` | Ajoute un en-tête |
| `-d '...'` | Envoie des données (passe en POST) |
| `-u admin` | Authentification (demande le mot de passe) |
| `-w "%{http_code}\n"` | Affiche une info choisie, ex. seulement le code HTTP |
| `--resolve nom:443:IP` | Force l'IP d'un nom, sans toucher au DNS |

```bash
curl -I http://10.0.0.30                                   # le répartiteur répond-il ? quel code ?
for i in 1 2 3 4; do curl -s http://10.0.0.30 | grep -o "srv-web0[12]"; done   # alternance des serveurs
curl -sk -o /dev/null -w "%{http_code}\n" https://pve01:8006
curl -LO https://exemple.org/fichier.iso                   # télécharger
curl -s -X POST -H "Content-Type: application/json" -d '{"nom":"alice"}' http://srv-web01/api/users
```

**PowerShell :**

```powershell
Invoke-WebRequest http://10.0.0.30 -UseBasicParsing | Select-Object StatusCode, Headers
Invoke-WebRequest https://exemple.org/fichier.iso -OutFile fichier.iso
Invoke-RestMethod http://srv-web01/api/users -Method Post -ContentType 'application/json' -Body '{"nom":"alice"}'
Invoke-WebRequest https://pve01:8006 -SkipCertificateCheck   # PowerShell 7 seulement
```

`Invoke-RestMethod` convertit directement le JSON reçu en objets : `(Invoke-RestMethod https://api.exemple.org/items).name`.

!!! warning "`curl` dans Windows PowerShell 5.1"
    Dans Windows PowerShell 5.1, `curl` et `wget` sont des **alias** de `Invoke-WebRequest` : `curl -I` échoue. Tape **`curl.exe`** pour avoir le vrai curl (fourni avec Windows 10 / 11). PowerShell 7 a supprimé ces alias.

### `wget`

| Option | Effet |
|---|---|
| `-O fichier` | Nom du fichier de sortie (`-O -` : affiche à l'écran) |
| `-c` | Reprend un téléchargement interrompu |
| `-q` | Silencieux |
| `--no-check-certificate` | Ignore un certificat non valide |

```bash
wget -c https://exemple.org/debian.iso
```

### `ip` : configuration réseau

| Commande | Effet |
|---|---|
| `ip a` (`ip addr`) | Adresses IP de chaque interface |
| `ip -br a` | Version courte : une ligne par interface |
| `ip r` (`ip route`) | Table de routage, passerelle par défaut (`default via`) |
| `ip link` | Interfaces, état UP/DOWN, adresse MAC |
| `ip neigh` | Table ARP (voisins connus) |
| `sudo ip addr add 10.0.0.50/24 dev ens18` | Ajoute une IP (temporaire, perdue au redémarrage) |
| `sudo ip link set ens18 up` | Active une interface |

```bash
ip -br a
ip r get 1.1.1.1          # par quelle interface et quelle passerelle part ce trafic ?
```

**PowerShell :** `Get-NetIPAddress -AddressFamily IPv4` · `Get-NetRoute -DestinationPrefix 0.0.0.0/0` · `Get-NetAdapter` · `Get-NetNeighbor` · `New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 10.0.0.50 -PrefixLength 24`

### `ss` : ports en écoute et connexions

| Option | Effet |
|---|---|
| `-t` · `-u` | TCP · UDP |
| `-l` | Seulement les ports **en écoute** |
| `-n` | Numéros au lieu des noms (`22` et pas `ssh`) |
| `-p` | Le processus qui tient le port (avec `sudo`) |
| `-a` | Tout : en écoute et connexions établies |

Le réflexe : **`sudo ss -tulpn`** (« tulpn ») : qui écoute sur quel port ?

```bash
sudo ss -tulpn
sudo ss -tlpn | grep :80       # qui occupe le port 80 ?
ss -tn state established       # connexions en cours
```

**PowerShell :**

```powershell
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
Get-NetTCPConnection -LocalPort 80 | Select-Object LocalPort, OwningProcess, @{n='Processus';e={(Get-Process -Id $_.OwningProcess).Name}}
netstat -ano | findstr :80     # l'ancienne méthode, toujours là
```

### Tester un port et le DNS

| Besoin | Linux | PowerShell |
|---|---|---|
| Un port TCP répond-il ? | `nc -zv 10.0.0.31 80` | `Test-NetConnection 10.0.0.31 -Port 80` |
| Ping + chemin | `ping -c 4 srv-dc01` · `traceroute srv-dc01` | `Test-NetConnection srv-dc01 -TraceRoute` |
| Résoudre un nom | `dig srv-dc01.lab.local` · `dig +short srv-dc01.lab.local` | `Resolve-DnsName srv-dc01.lab.local` |
| Interroger un DNS précis | `dig @10.0.0.10 srv-web01.lab.local` | `Resolve-DnsName srv-web01.lab.local -Server 10.0.0.10` |
| Un type d'enregistrement | `dig lab.local MX` · `dig -x 10.0.0.10` (inverse) | `Resolve-DnsName lab.local -Type MX` |

Plus de commandes réseau, classées par besoin : 📋 [Mémo commandes réseau](../reseau/memo-commandes-reseau.md).

## Accès distant

### `ssh`

| Option | Effet |
|---|---|
| `ssh admin@10.0.0.31` | Connexion avec le compte `admin` |
| `-p 2222` | Port non standard |
| `-i ~/.ssh/id_ed25519` | Clé privée à utiliser |
| `-J admin@pve01` | Rebond : passe par `pve01` pour atteindre une VM du réseau interne |
| `-L 8080:10.0.0.31:80` | Tunnel local : `localhost:8080` sur ton PC mène au port 80 de la VM |
| `-v` | Mode bavard, pour comprendre un échec |
| `ssh hote 'commande'` | Lance une commande à distance et revient |
| `ssh-keygen -t ed25519` | Crée une paire de clés |
| `ssh-copy-id admin@10.0.0.31` | Installe ta clé publique sur le serveur |

```bash
ssh -J admin@pve01 admin@10.0.0.31
ssh admin@srv-web01 'sudo systemctl status nginx --no-pager'
ssh -L 8080:10.0.0.60:3001 admin@pve01      # Uptime Kuma sur http://localhost:8080
```

**PowerShell :** le client `ssh` est intégré à Windows 10 / 11, mêmes options. Pas de `ssh-copy-id` :

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh admin@10.0.0.31 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
Enter-PSSession srv-dc01                          # session PowerShell distante (WinRM)
Invoke-Command srv-dc01 { Get-Service DNS }
```

Pour tout comprendre : 🧠 [SSH](../reseau/concept-ssh.md) · 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md).

### `scp` et `rsync` : copier par SSH

| Commande et option | Effet |
|---|---|
| `scp fichier admin@hote:/chemin/` | Envoie un fichier |
| `scp admin@hote:/chemin/fichier .` | Récupère un fichier dans le dossier courant |
| `scp -r` | Copie un dossier |
| `scp -P 2222` | Port (**P majuscule**, contrairement à `ssh -p`) |
| `scp -J admin@pve01` | Par rebond |
| `rsync -avh` | Synchronise : archive (droits, dates), détaillé, tailles lisibles |
| `rsync --delete` | Supprime à la destination ce qui n'existe plus à la source |
| `rsync -n` | **À blanc** : montre ce qui serait fait |
| `rsync --progress` | Affiche l'avancement |

```bash
scp -J admin@pve01 nginx.conf admin@10.0.0.31:/tmp/
scp admin@10.0.0.20:/srv/partage/rapport.pdf .
rsync -avhn --delete /srv/partage/ admin@10.0.0.20:/sauvegardes/partage/   # tester
rsync -avh  --delete /srv/partage/ admin@10.0.0.20:/sauvegardes/partage/   # puis lancer
```

!!! tip "Le `/` final compte avec `rsync`"
    `rsync -a src/ dest/` copie **le contenu** de `src` dans `dest`. `rsync -a src dest/` crée `dest/src/`.

**PowerShell :** `scp` marche pareil sous Windows. Entre machines Windows : `Copy-Item .\fichier.txt -Destination C:\Temp -ToSession (New-PSSession srv-dc01)` · ou `robocopy C:\Partage \\srv-files01\Sauvegarde /MIR` (`/MIR` = `rsync --delete`).

## Services et journaux

### `systemctl`

| Commande | Effet |
|---|---|
| `systemctl status nginx` | État, PID, dernières lignes de log |
| `start` · `stop` · `restart` | Démarrer · arrêter · redémarrer |
| `reload` | Relire la conf sans couper les connexions |
| `enable` · `disable` | Lancer ou non au démarrage |
| `enable --now` | Activer au démarrage **et** démarrer tout de suite |
| `is-active` · `is-enabled` | Réponse courte, pratique en script |
| `systemctl --failed` | Services en échec |
| `systemctl list-units --type=service` | Tous les services chargés |
| `systemctl daemon-reload` | Relire les fichiers `.service` après une modification |
| `--no-pager` | Affiche tout d'un coup (sans `less`) |

```bash
sudo nginx -t && sudo systemctl reload nginx     # tester la conf avant de recharger
sudo systemctl enable --now wg-quick@wg0
systemctl is-active nginx || echo "nginx est arrêté"
```

**PowerShell :**

```powershell
Get-Service Spooler
Restart-Service Spooler
Set-Service Spooler -StartupType Automatic
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' }   # ≈ systemctl --failed
```

### `journalctl`

| Option | Effet |
|---|---|
| `-u nginx` | Journal d'un service |
| `-f` | Suit en direct |
| `-n 50` | Les 50 dernières lignes |
| `-e` | Saute à la fin |
| `-b` · `-b -1` | Depuis le démarrage actuel · le précédent |
| `-p err` | Seulement les erreurs (et plus grave) |
| `--since "1 hour ago"` · `--since today` | Depuis une date |
| `-k` | Messages du noyau (≈ `dmesg`) |
| `--disk-usage` · `--vacuum-time=7d` | Place prise par les journaux · ne garder que 7 jours |

```bash
journalctl -u ssh --since today | grep -i failed
journalctl -b -p err --no-pager
sudo journalctl -fu nginx
```

**PowerShell :**

```powershell
Get-WinEvent -LogName System -MaxEvents 50
Get-WinEvent -FilterHashtable @{ LogName='System'; Level=2; StartTime=(Get-Date).AddHours(-1) }   # erreurs de la dernière heure
Get-WinEvent -FilterHashtable @{ LogName='Security'; Id=4625 } -MaxEvents 20                       # échecs de connexion
```

## Les briques PowerShell à connaître

Côté Windows, la plupart des besoins se résolvent en enchaînant **une commande qui produit des objets** et quelques cmdlets de pipeline.

| Cmdlet | Alias | Rôle | Exemple |
|---|---|---|---|
| `Where-Object` | `?`, `where` | Filtrer (≈ `grep`) | <code>Get-Service &#124; Where-Object Status -eq Running</code> |
| `Select-Object` | `select` | Choisir des colonnes, les N premiers (≈ `cut`, `head`) | <code>Get-Process &#124; Select-Object -First 5 Name, CPU</code> |
| `Sort-Object` | `sort` | Trier | <code>Get-ChildItem &#124; Sort-Object Length -Descending</code> |
| `ForEach-Object` | `%`, `foreach` | Agir sur chaque objet (≈ `xargs`) | <code>Get-Content serveurs.txt &#124; ForEach-Object { Test-Connection $_ -Count 1 }</code> |
| `Group-Object` | `group` | Regrouper et compter (≈ <code>sort &#124; uniq -c</code>) | <code>Get-Service &#124; Group-Object Status</code> |
| `Measure-Object` | `measure` | Compter, sommer (≈ `wc`) | <code>Get-ChildItem -File &#124; Measure-Object Length -Sum</code> |
| `Format-Table` · `Format-List` | `ft` · `fl` | Mise en forme **à l'écran seulement** | <code>Get-Service Spooler &#124; Format-List *</code> |
| `Export-Csv` · `Import-Csv` | — | Écrire / lire un CSV | <code>Get-ADUser -Filter * &#124; Export-Csv users.csv -NoTypeInformation -Encoding UTF8</code> |
| `ConvertTo-Json` · `ConvertFrom-Json` | — | Objets ⇄ JSON | <code>Get-Service Spooler &#124; ConvertTo-Json</code> |
| `Out-File` · `Set-Content` · `Add-Content` | — | Écrire dans un fichier (≈ `>` et `>>`) | <code>Get-Date &#124; Add-Content log.txt</code> |
| `Get-Member` | `gm` | Voir les propriétés d'un objet | <code>Get-Service &#124; Get-Member</code> |
| `Get-Help` · `Get-Command` | `help` · `gcm` | Aide et recherche de commandes | `Get-Help Test-NetConnection -Examples` |

!!! tip "Paramètres communs à toutes les cmdlets"
    - `-WhatIf` : montre ce qui **serait** fait, sans le faire.
    - `-Confirm` : demande avant chaque action.
    - `-ErrorAction SilentlyContinue` (`-EA 0`) : ignore les erreurs (droits refusés pendant une recherche…).
    - `-Verbose` : explique ce que fait la commande.

## Enchaîner et rediriger

| Besoin | Linux (Bash) | PowerShell |
|---|---|---|
| Envoyer le résultat à la commande suivante | <code>cmd1 &#124; cmd2</code> (du texte) | <code>cmd1 &#124; cmd2</code> (des objets) |
| Écrire dans un fichier (écrase) | `cmd > fichier` | `cmd > fichier` · <code>cmd &#124; Out-File fichier</code> |
| Ajouter à la fin | `cmd >> fichier` | `cmd >> fichier` · `Add-Content` |
| Les erreurs aussi | `cmd > log 2>&1` | `cmd *> log` |
| Jeter la sortie | `cmd > /dev/null 2>&1` | <code>cmd &#124; Out-Null</code> · `$null = cmd` |
| Afficher **et** enregistrer | <code>cmd &#124; tee fichier</code> | <code>cmd &#124; Tee-Object fichier</code> |
| Lancer la 2e seulement si la 1re réussit | `cmd1 && cmd2` | `cmd1 && cmd2` (PS 7) · `cmd1; if ($?) { cmd2 }` |
| Lancer la 2e si la 1re échoue | <code>cmd1 &#124;&#124; cmd2</code> | <code>cmd1 &#124;&#124; cmd2</code> (PS 7) |
| Passer chaque résultat en argument | <code>find . -name "*.log" &#124; xargs rm</code> | <code>... &#124; ForEach-Object { Remove-Item $_ }</code> |
| En administrateur | `sudo cmd` · `sudo -i` (shell root) | Clic droit → Exécuter en tant qu'administrateur |
| Relancer la dernière commande en root | `sudo !!` | — |
| Historique | `history` · Ctrl+R pour chercher | `Get-History` · Ctrl+R aussi |

## Pièges fréquents

!!! warning "Pièges fréquents"
    - **`sed -i` sans test** : lance d'abord sans `-i` (ou avec `-i.bak`) pour voir le résultat. Un mauvais motif peut casser toute une conf.
    - **`grep` qui se trouve lui-même** : `ps aux | grep nginx` affiche aussi la ligne `grep nginx`. Utiliser `pgrep -a nginx`.
    - **`find -delete` en premier** : l'ordre compte. `find . -delete -name "*.tmp"` supprime **tout**. Mettre `-delete` en dernier, et tester sans d'abord.
    - **`scp -p` n'est pas le port** : c'est `-P` (majuscule). `-p` conserve les dates.
    - **`ls` en PowerShell** : c'est `Get-ChildItem`, les options Linux (`-la`) ne marchent pas.
    - **`curl` en Windows PowerShell 5.1** : alias d'`Invoke-WebRequest`. Taper `curl.exe`.
    - **`Format-Table` au milieu d'un pipeline** : il produit de l'affichage, plus des objets. Le mettre **en dernier** (jamais avant `Export-Csv`).
    - **Guillemets** : `'...'` garde le texte tel quel, `"..."` remplace les variables, dans les deux mondes. En `sed` et `awk`, toujours des guillemets simples.
    - **Fichier `.sh` écrit sous Windows** : `$'\r': command not found`. Corriger avec `sed -i 's/\r$//' script.sh`.

## Pour aller plus loin

- 📋 [Mémo commandes du quotidien](../pratiques/memo-commandes-quotidien.md) : les mêmes outils classés par besoin (disque, processus, services, comptes, mises à jour)
- 📋 [Mémo scripting Bash & PowerShell](../transverse/memo-bash-powershell.md) : variables, conditions, boucles, fonctions
- 📋 [Mémo commandes réseau](../reseau/memo-commandes-reseau.md) : configuration IP, tests, DNS
- 🛠️ [Gérer utilisateurs, groupes et permissions sous Linux](../systeme/tuto-permissions-linux.md)
- 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)
