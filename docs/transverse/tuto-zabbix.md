---
icon: material/wrench-outline
title: Superviser ses VM avec Zabbix
tags: [transverse, tuto, supervision, monitoring, zabbix, proxmox, windows-server, linux]
---

# 🛠️ Superviser ses VM avec Zabbix

!!! abstract "Objectif"
    Monter un serveur **Zabbix 7.0 LTS** sur une VM dédiée `srv-zabbix01`, installer l'**agent Zabbix** sur les VM Linux et Windows du lab, appliquer les **modèles** prêts à l'emploi (CPU, RAM, disques, services), superviser l'hôte **Proxmox** par son API, puis recevoir les **alertes** par mail.

**Niveau :** 🟡 Intermédiaire · **Durée :** 2 h

## Prérequis

- Savoir ce qu'on supervise et pourquoi → 🧠 [La supervision : métriques, logs et alertes](concept-supervision.md)
- Une VM Debian 12 ou 13 → 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md)
- Joindre le réseau interne depuis ton PC → 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)
- Conseillé : avoir déjà fait 🛠️ [Superviser ses VM avec Uptime Kuma](tuto-uptime-kuma.md) pour comparer

## Uptime Kuma ou Zabbix ?

| | Uptime Kuma | Zabbix |
|---|---|---|
| Ce qu'il voit | L'extérieur : la machine répond-elle ? le service ? | L'intérieur aussi : CPU, RAM, disques, services, journaux Windows |
| Comment | Sondes réseau, sans rien installer sur les cibles | Un **agent** sur chaque machine (+ SNMP, API, sondes réseau) |
| Configuration | Une sonde à la fois | Des **modèles** qui créent d'un coup des dizaines d'éléments, déclencheurs et graphiques |
| Où on le croise | Homelab, petites structures | Entreprises, collectivités, hébergeurs |

## Le plan qu'on va monter

| Machine | IP | Rôle dans ce tuto | Modèle Zabbix |
|---|---|---|---|
| `srv-zabbix01` (nouvelle VM) | `10.0.0.61` | Serveur Zabbix + base MariaDB + interface web | Zabbix server health |
| `srv-web01`, `srv-web02`, `srv-lb01` | `10.0.0.31`, `.32`, `.30` | Agent Linux | Linux by Zabbix agent |
| `srv-dc01` | `10.0.0.10` | Agent Windows | Windows by Zabbix agent |
| `pve01` | `10.0.0.1` | Supervisé par l'API Proxmox (sans agent) | Proxmox VE by HTTP |

```
[srv-zabbix01] ──interroge le port 10050──▶ agents (srv-web01, srv-dc01…)
      │        ──API HTTPS :8006──────────▶ pve01
      └── 📧 alerte mail
```

Prévoir **2 vCPU, 2 Go de RAM et 20 Go de disque** pour `srv-zabbix01`.

---

## Étape 1 — Préparer la VM `srv-zabbix01`

Cloner `tpl-debian01` en `srv-zabbix01`, IP `10.0.0.61/24`, passerelle `10.0.0.1`, DNS `10.0.0.10` (voir [Préparer une VM clonée](../systeme/tuto-preparer-clone-vm.md)).

```bash
hostnamectl                       # srv-zabbix01
cat /etc/debian_version           # 12.x ou 13.x : on en a besoin à l'étape 2
timedatectl set-timezone Europe/Paris
```

## Étape 2 — Ajouter le dépôt Zabbix et installer les paquets

En `root` (ou avec `sudo`). La commande choisit automatiquement le paquet de dépôt pour Debian 12 ou 13.

```bash
apt update && apt install -y wget mariadb-server
. /etc/os-release
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian${VERSION_ID}_all.deb
dpkg -i zabbix-release_latest_7.0+debian${VERSION_ID}_all.deb
apt update
apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent2
```

| Paquet | Rôle |
|---|---|
| `zabbix-server-mysql` | Le moteur qui collecte et évalue les données (compatible MariaDB) |
| `zabbix-frontend-php` + `zabbix-apache-conf` | L'interface web, servie par Apache |
| `zabbix-sql-scripts` | Le schéma de la base de données |
| `zabbix-agent2` | L'agent, pour que le serveur se supervise lui-même |

## Étape 3 — Créer la base de données

```bash
mariadb -uroot
```

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'MotDePasseZabbix';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
SET GLOBAL log_bin_trust_function_creators = 1;
QUIT;
```

Importer le schéma (une à deux minutes, demande le mot de passe de l'utilisateur `zabbix`), puis remettre l'option de sécurité :

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mariadb --default-character-set=utf8mb4 -uzabbix -p zabbix
mariadb -uroot -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

Indiquer le mot de passe au serveur Zabbix, puis tout démarrer :

```bash
sed -i 's/^# DBPassword=/DBPassword=MotDePasseZabbix/' /etc/zabbix/zabbix_server.conf
grep ^DBPassword /etc/zabbix/zabbix_server.conf      # DBPassword=MotDePasseZabbix
systemctl restart zabbix-server zabbix-agent2 apache2
systemctl enable zabbix-server zabbix-agent2 apache2
```

## Étape 4 — Terminer l'installation dans le navigateur

Depuis ton PC, ouvrir un tunnel par `pve01` (comme pour [Uptime Kuma](tuto-uptime-kuma.md), étape 4), puis aller sur **http://localhost:8080/zabbix** :

```bash
ssh -L 8080:10.0.0.61:80 pve01        # Windows (PowerShell) ou Linux, même commande
```

1. *Default language* : **Français** → **Étape suivante**.
2. Vérification des prérequis : tout doit être **OK**.
3. Base de données : type **MySQL**, hôte `localhost`, base `zabbix`, utilisateur `zabbix`, mot de passe `MotDePasseZabbix`.
4. Paramètres : *Nom du serveur Zabbix* `srv-zabbix01`, fuseau `Europe/Paris`.
5. Récapitulatif → **Terminer**.
6. Se connecter avec **`Admin`** / **`zabbix`**, puis **changer ce mot de passe tout de suite** : **Utilisateurs → Utilisateurs → Admin → Changer le mot de passe**.

## Étape 5 — Installer l'agent sur les VM

L'agent écoute sur le port **10050** et ne répond qu'au serveur déclaré dans sa configuration.

=== "Linux (Debian)"
    Sur chaque VM (`srv-web01`, `srv-web02`, `srv-lb01`…), en adaptant `Hostname` :

    ```bash
    . /etc/os-release
    wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_latest_7.0+debian${VERSION_ID}_all.deb
    dpkg -i zabbix-release_latest_7.0+debian${VERSION_ID}_all.deb
    apt update && apt install -y zabbix-agent2

    sed -i -e 's/^Server=.*/Server=10.0.0.61/' \
           -e 's/^ServerActive=.*/ServerActive=10.0.0.61/' \
           -e 's/^Hostname=.*/Hostname=srv-web01/' /etc/zabbix/zabbix_agent2.conf
    systemctl restart zabbix-agent2 && systemctl enable zabbix-agent2
    ss -tlnp | grep 10050          # l'agent écoute
    ```

=== "Windows Server (PowerShell)"
    Sur `srv-dc01`, télécharger l'installeur **MSI** de l'agent 2 (Zabbix 7.0, Windows, amd64, OpenSSL) depuis la page *Download Zabbix agents* du site officiel, puis dans un PowerShell administrateur :

    ```powershell
    msiexec /i zabbix_agent2-7.0.<version>-windows-amd64-openssl.msi /qn `
      SERVER=10.0.0.61 SERVERACTIVE=10.0.0.61 HOSTNAME=srv-dc01
    Get-Service "Zabbix Agent 2"                  # Status : Running
    New-NetFirewallRule -DisplayName "Zabbix agent" -Direction Inbound -Protocol TCP `
      -LocalPort 10050 -RemoteAddress 10.0.0.61 -Action Allow
    ```

!!! warning "Le `Hostname` doit être identique des deux côtés"
    Le nom mis dans la configuration de l'agent doit être **exactement** le *Nom de l'hôte* déclaré dans l'interface (étape 6), majuscules comprises. Sinon les vérifications actives échouent.

## Étape 6 — Déclarer les hôtes et leur appliquer un modèle

**Collecte de données → Hôtes → Créer un hôte** :

| Champ | `srv-web01` | `srv-dc01` |
|---|---|---|
| *Nom de l'hôte* | `srv-web01` | `srv-dc01` |
| *Modèles* | `Linux by Zabbix agent` | `Windows by Zabbix agent` |
| *Groupes d'hôtes* | `Linux servers` | `Windows servers` (à créer si absent) |
| *Interfaces* → **Agent** | `10.0.0.31`, port `10050` | `10.0.0.10`, port `10050` |

Ajouter aussi le modèle **`ICMP Ping`** à chaque hôte pour l'alerte « machine injoignable ». Après une à deux minutes, la pastille **ZBX** passe au **vert** dans la liste des hôtes, et **Surveillance → Dernières données** se remplit (CPU, mémoire, systèmes de fichiers, interfaces réseau…).

| Ce que crée un modèle | Exemple dans `Linux by Zabbix agent` |
|---|---|
| **Éléments** (*items*) : les valeurs collectées | `vfs.fs.size[/,pused]` : % d'occupation de `/` |
| **Déclencheurs** (*triggers*) : les conditions de problème | « FS [/]: Space is low » au-dessus de 80 % |
| **Graphiques** et tableaux de bord | Utilisation CPU, mémoire, trafic réseau |
| **Règles de découverte** | Trouve tout seul chaque disque et chaque carte réseau |

## Étape 7 — Superviser Proxmox par son API

Pas d'agent sur l'hyperviseur : Zabbix interroge l'API avec un **jeton en lecture seule**. Sur `pve01` :

```bash
pveum user add zabbix@pve --comment "Supervision Zabbix"
pveum acl modify / --users zabbix@pve --roles PVEAuditor
pveum user token add zabbix@pve monitoring --privsep 0
# Noter la valeur « value » affichée : elle n'est montrée qu'une fois
```

Dans Zabbix, créer l'hôte `pve01` : modèle **`Proxmox VE by HTTP`**, sans interface agent, puis onglet **Macros** :

| Macro | Valeur |
|---|---|
| `{$PVE.URL.HOST}` | `10.0.0.1` |
| `{$PVE.TOKEN.ID}` | `zabbix@pve!monitoring` |
| `{$PVE.TOKEN.SECRET}` | la valeur du jeton (type *Texte secret*) |

Zabbix découvre alors le nœud, les stockages et chaque VM (état, CPU, RAM).

## Étape 8 — Recevoir les alertes par mail

1. **Alertes → Types de média → Email** : *Serveur SMTP* (ex. `smtp.gmail.com`), *Port* `587`, *Sécurité de la connexion* **STARTTLS**, *Authentification* nom d'utilisateur + **mot de passe d'application**, *E-mail SMTP* l'adresse d'envoi → **Actualiser**, puis **Tester**.
2. **Utilisateurs → Utilisateurs → Admin → Média → Ajouter** : type **Email**, *Envoyer à* ton adresse, gravité à partir de **Avertissement**.
3. **Alertes → Actions → Actions de déclencheur** : activer **Report problems to Zabbix administrators**.

Zabbix propose aussi des types de média **Telegram**, **Discord**, **Slack**… dans la même liste.

!!! tip "Tester pour de vrai"
    Sur `srv-web02`, remplir le disque au-delà du seuil puis libérer la place :
    ```bash
    df -h /                                   # repérer l'espace libre
    fallocate -l 5G /root/remplissage.img     # adapter pour dépasser 80 %
    # attendre le problème « Space is low » et le mail, puis :
    rm /root/remplissage.img
    ```

---

## ✅ Vérification

```bash
# Sur srv-zabbix01
systemctl is-active zabbix-server zabbix-agent2 apache2 mariadb   # 4 × active
tail -n 20 /var/log/zabbix/zabbix_server.log                       # pas d'erreur de base
# Depuis srv-zabbix01 vers un agent
apt install -y zabbix-get
zabbix_get -s 10.0.0.31 -k agent.ping                              # 1
```

Dans l'interface : **Collecte de données → Hôtes**, pastilles **ZBX** vertes ; **Surveillance → Problèmes** vide (ou seulement de vrais problèmes) ; un mail de test reçu.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| Pastille **ZBX** verte, données qui arrivent | L'agent répond | Rien à faire |
| Pastille **ZBX** rouge : *Get value from agent failed: cannot connect* | Port 10050 fermé ou agent arrêté | `systemctl status zabbix-agent2`, pare-feu de la VM |
| *Received empty response… check allowed connections* | Le serveur n'est pas dans `Server=` de l'agent | Corriger `Server=10.0.0.61`, redémarrer l'agent |
| *host [srv-web01] not found* dans le journal de l'agent | Le `Hostname` ne correspond pas | Même nom dans l'agent et dans l'interface |
| « Zabbix server is not running » en haut de l'interface | Le serveur ne démarre pas | `journalctl -u zabbix-server`, vérifier `DBPassword` |
| Hôte `pve01` sans données, erreur 401 | Jeton faux ou sans droits | Revoir `{$PVE.TOKEN.ID}`, le secret et le rôle `PVEAuditor` |

## 🔧 Dépannage courant

!!! question "L'installeur web affiche « Cannot connect to the database » ?"
    Vérifier le mot de passe en se connectant à la main : `mariadb -uzabbix -p zabbix`. Si ça échoue, recréer l'utilisateur (étape 3).

!!! question "L'import du schéma échoue avec « You do not have the SUPER privilege… » ?"
    L'option `log_bin_trust_function_creators` n'était pas à `1` pendant l'import. La remettre à `1`, supprimer puis recréer la base, relancer l'import.

!!! question "L'agent Windows est installé mais la pastille reste rouge ?"
    Vérifier la règle de pare-feu (port TCP 10050 depuis `10.0.0.61`) et le fichier `C:\Program Files\Zabbix Agent 2\zabbix_agent2.conf` (`Server=`, `Hostname=`), puis `Restart-Service "Zabbix Agent 2"`.

!!! question "Trop d'alertes dès le premier jour ?"
    Les seuils des modèles sont prévus pour la production. Les adapter par hôte avec les **macros** (ex. `{$VFS.FS.PUSED.MAX.WARN}` pour le seuil disque) plutôt que de modifier le modèle.

## Voir aussi

- 🧠 [La supervision : métriques, logs et alertes](concept-supervision.md)
- 🛠️ [Superviser ses VM avec Uptime Kuma](tuto-uptime-kuma.md) : plus simple, pour la disponibilité
- 🛠️ [Superviser ses VM avec Centreon](tuto-centreon.md) : l'autre grand outil d'entreprise, basé sur SNMP
- 🛠️ [Monter un Active Directory sur une VM Proxmox](../systeme/tuto-monter-ad-proxmox.md)

??? quote "Sources"
    - [Zabbix 7.0 : manuel](https://www.zabbix.com/documentation/7.0/en/manual)
    - [Zabbix : téléchargement et installation](https://www.zabbix.com/download)
    - [Zabbix : supervision de Proxmox par HTTP](https://www.zabbix.com/integrations/proxmox)
