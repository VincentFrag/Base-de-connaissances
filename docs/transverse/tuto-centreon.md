---
icon: material/wrench-outline
title: Superviser ses VM avec Centreon
tags: [transverse, tuto, supervision, monitoring, centreon, snmp, linux, windows-server]
---

# 🛠️ Superviser ses VM avec Centreon

!!! abstract "Objectif"
    Installer **Centreon** (édition gratuite *Open Source*) sur une VM dédiée `srv-centreon01`, activer **SNMP** sur les VM Linux et Windows du lab, les déclarer avec les **connecteurs de supervision** prêts à l'emploi, **exporter la configuration** vers le moteur, puis recevoir les **notifications** par mail.

**Niveau :** 🟡 Intermédiaire · **Durée :** 2 h 30

## Prérequis

- Savoir ce qu'on supervise et pourquoi → 🧠 [La supervision : métriques, logs et alertes](concept-supervision.md)
- Une VM **Debian 13** neuve → 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md) (si ton template est en Debian 12, installer une VM Debian 13 à part)
- Joindre le réseau interne depuis ton PC → 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)
- Conseillé : avoir fait 🛠️ [Superviser ses VM avec Zabbix](tuto-zabbix.md) pour comparer

## Centreon en deux mots

Centreon est un éditeur **français**, très présent dans les entreprises, les collectivités et les hôpitaux en France. Il descend de **Nagios** : on retrouve ses notions d'**hôtes**, de **services**, de **commandes** et de **plugins**. Il interroge les machines surtout par **SNMP** (ou par son agent, *Centreon Monitoring Agent*).

| Notion Centreon | Ce que c'est | Équivalent Zabbix |
|---|---|---|
| **Hôte** | Une machine supervisée | Hôte |
| **Service** | Une vérification sur un hôte (CPU, disque, ping…) | Élément + déclencheur |
| **Modèle d'hôte** (*host template*) | Applique d'un coup une liste de services | Modèle |
| **Connecteur de supervision** (*Monitoring Connector*) | Paquet de modèles + plugin pour une technologie (Linux SNMP, Windows SNMP…) | Modèle fourni |
| **Collecteur** (*poller*) | Le moteur qui exécute les vérifications ; le serveur central en est un | Serveur ou proxy Zabbix |
| **Exporter la configuration** | Envoyer les changements au moteur : **rien n'est pris en compte avant** | Automatique |

## Le plan qu'on va monter

| Machine | IP | Rôle | Modèle d'hôte |
|---|---|---|---|
| `srv-centreon01` (nouvelle VM) | `10.0.0.62` | Serveur central Centreon + MariaDB | — |
| `srv-web01`, `srv-web02`, `srv-lb01` | `10.0.0.31`, `.32`, `.30` | SNMP Linux | `OS-Linux-SNMP-custom` |
| `srv-dc01` | `10.0.0.10` | SNMP Windows | `OS-Windows-SNMP-custom` |

Communauté SNMP en lecture seule : **`lab-ro`**, autorisée uniquement depuis `10.0.0.62`.

Prévoir **2 à 4 vCPU, 4 Go de RAM et 40 Go de disque** pour `srv-centreon01` (la documentation officielle donne 4 vCPU / 4 Go pour un serveur central jusqu'à 500 hôtes).

---

## Étape 1 — Préparer la VM `srv-centreon01`

IP `10.0.0.62/24`, passerelle `10.0.0.1`, DNS `10.0.0.10`, nom `srv-centreon01`.

```bash
hostnamectl                           # srv-centreon01
cat /etc/debian_version               # 13.x
timedatectl set-timezone Europe/Paris # Centreon reprend ce fuseau pour PHP
apt update && apt full-upgrade -y
apt install -y lsb-release ca-certificates wget gnupg
```

## Étape 2 — Ajouter les dépôts Centreon et installer

On installe ici **Centreon 26.10** (dernière version au moment de l'écriture). Pour une autre version, remplacer le numéro dans la première ligne et suivre la page *Installation using packages* de la documentation officielle (`docs.centreon.com`).

```bash
VERSION=26.10
echo "deb https://packages.centreon.com/apt-standard-${VERSION}-stable/ $(lsb_release -sc) main" \
  | tee /etc/apt/sources.list.d/centreon.list
echo "deb https://packages.centreon.com/apt-plugins-stable/ $(lsb_release -sc) main" \
  | tee /etc/apt/sources.list.d/centreon-plugins.list
wget -O- https://apt-key.centreon.com | gpg --dearmor | tee /etc/apt/trusted.gpg.d/centreon.gpg > /dev/null 2>&1
apt update

apt install -y centreon-mariadb centreon     # Centreon + base MariaDB locale
systemctl daemon-reload
systemctl restart mariadb
systemctl enable mariadb
```

Sécuriser MariaDB et définir le mot de passe `root` de la base (demandé à l'étape 3) :

```bash
mariadb-secure-installation
```

Répondre **oui** à toutes les questions **sauf** « *Disallow root login remotely?* ».

## Étape 3 — Terminer l'installation dans le navigateur

Depuis ton PC, tunnel par `pve01` puis **http://localhost:8080/centreon** :

```bash
ssh -L 8080:10.0.0.62:80 pve01        # Windows (PowerShell) ou Linux
```

| Étape de l'assistant | Quoi faire |
|---|---|
| 1. Bienvenue | **Next** |
| 2. Vérification des dépendances | Tout doit être vert (sinon corriger, **Refresh**) |
| 3. Moteur de supervision | Laisser les valeurs par défaut |
| 4. Module Broker | Laisser les valeurs par défaut |
| 5. Compte administrateur | Identifiant `admin` ; mot de passe d'**au moins 12 caractères** avec majuscule, minuscule, chiffre et caractère spécial (`@$!%*?&`) |
| 6. Base de données | *Hôte* : vide (= `localhost`) ; *Root* : `root` + le mot de passe de l'étape 2 ; *Utilisateur Centreon* : `centreon` + un nouveau mot de passe (créé par l'assistant) |
| 7. Installation | Attendre la création de la base → **Next** |
| 8. Modules | **Install**, puis **Next** |
| 9. Fin | **Finish**, puis se connecter avec `admin` |

Ensuite, dans **Administration → Extensions → Gestionnaire**, cliquer sur **Install all** si des modules restent à installer.

## Étape 4 — Démarrer la supervision du serveur central

**Configuration → Collecteurs → Collecteurs**, cocher **Central** → **Exporter la configuration** → cocher aussi **Déplacer les fichiers d'export** → **Exporter**.

Puis, sur `srv-centreon01` :

```bash
systemctl restart cbd centengine
systemctl restart gorgoned
systemctl start snmptrapd centreontrapd
systemctl enable cbd centengine gorgoned centreon
```

| Service | Rôle |
|---|---|
| `centengine` | Le moteur qui lance les vérifications (héritier de Nagios) |
| `cbd` | Centreon Broker : range les résultats dans la base |
| `gorgoned` | Fait le lien entre l'interface et les collecteurs |
| `snmptrapd`, `centreontrapd` | Reçoivent les alertes SNMP envoyées par les équipements (*traps*) |

## Étape 5 — Activer SNMP sur les VM

SNMP v2c en lecture seule, avec une communauté qui sert de mot de passe et n'est acceptée que depuis Centreon.

=== "Linux (Debian)"
    Sur chaque VM (`srv-web01`, `srv-web02`, `srv-lb01`) :

    ```bash
    apt install -y snmpd
    cp /etc/snmp/snmpd.conf /etc/snmp/snmpd.conf.orig
    cat > /etc/snmp/snmpd.conf <<'EOF'
    agentaddress udp:161
    rocommunity lab-ro 10.0.0.62
    sysLocation Lab Proxmox
    sysContact admin@lab.local
    EOF
    systemctl restart snmpd && systemctl enable snmpd
    ss -ulnp | grep 161              # snmpd écoute en UDP 161
    ```

=== "Windows Server (PowerShell)"
    Sur `srv-dc01`, dans un PowerShell administrateur :

    ```powershell
    Install-WindowsFeature SNMP-Service -IncludeManagementTools
    $p = "HKLM:\SYSTEM\CurrentControlSet\Services\SNMP\Parameters"
    # Communauté lab-ro en lecture seule (4 = READ ONLY)
    New-ItemProperty -Path "$p\ValidCommunities" -Name "lab-ro" -Value 4 -PropertyType DWord -Force
    # N'accepter que srv-centreon01 (remplace l'entrée « localhost » par défaut)
    Set-ItemProperty -Path "$p\PermittedManagers" -Name "1" -Value "10.0.0.62"
    Restart-Service SNMP
    New-NetFirewallRule -DisplayName "SNMP depuis Centreon" -Direction Inbound -Protocol UDP `
      -LocalPort 161 -RemoteAddress 10.0.0.62 -Action Allow
    ```

    Le service SNMP de Windows est marqué *déprécié* par Microsoft mais reste disponible sur Windows Server. L'alternative moderne est le **Centreon Monitoring Agent** (voir plus bas).

Tester depuis `srv-centreon01` :

```bash
apt install -y snmp
snmpwalk -v2c -c lab-ro 10.0.0.31 1.3.6.1.2.1.1     # renvoie sysName = srv-web01, sysLocation…
snmpwalk -v2c -c lab-ro 10.0.0.10 1.3.6.1.2.1.1     # idem pour srv-dc01
```

## Étape 6 — Installer les connecteurs Linux et Windows

**Configuration → Gestionnaire de connecteurs de supervision** (*Monitoring Connectors Manager*) : chercher **Linux SNMP** puis **Windows SNMP** et cliquer sur **+** pour chacun. Le connecteur **Base Generic** (ping) est déjà installé avec Centreon.

Installer aussi les **plugins** correspondants sur le collecteur :

```bash
apt install -y centreon-plugin-operatingsystems-linux-snmp centreon-plugin-operatingsystems-windows-snmp
```

## Étape 7 — Déclarer les hôtes

**Configuration → Hôtes → Hôtes → Ajouter** :

| Champ | `srv-web01` | `srv-dc01` |
|---|---|---|
| *Nom* / *Alias* | `srv-web01` / `Serveur web 1` | `srv-dc01` / `Contrôleur de domaine` |
| *Adresse* | `10.0.0.31` | `10.0.0.10` |
| *Communauté SNMP* / *Version* | `lab-ro` / `2c` | `lab-ro` / `2c` |
| *Collecteur* | `Central` | `Central` |
| *Modèles* | `OS-Linux-SNMP-custom` | `OS-Windows-SNMP-custom` |
| *Créer aussi les services liés aux modèles* | **Oui** | **Oui** |

**Enregistrer**, puis **exporter la configuration** comme à l'étape 4 (sans cette étape, l'hôte n'apparaît jamais dans la supervision).

Après quelques minutes, **Supervision → Statut des ressources** affiche l'hôte en **UP** et ses services (**Cpu**, **Load**, **Memory**, **Swap**…) en **OK**. Pour les disques, ajouter un service à partir d'un modèle *Disk* du connecteur Linux SNMP (ex. sur `/`).

| État | Couleur | Signification |
|---|---|---|
| **OK** / **UP** | Vert | Tout va bien |
| **WARNING** | Orange | Seuil d'avertissement dépassé |
| **CRITICAL** / **DOWN** | Rouge | Problème : alerte |
| **UNKNOWN** | Gris | La vérification n'a pas pu se faire (souvent SNMP) |
| **PENDING** | Bleu | Pas encore vérifié |

## Étape 8 — Recevoir les notifications par mail

Centreon envoie les mails par le **serveur de messagerie local** du système : il faut un relais SMTP sur `srv-centreon01` (ex. `postfix` en *relayhost* vers ton fournisseur de mail, avec un mot de passe d'application).

1. Vérifier l'envoi depuis le serveur : `echo test | mail -s "Test Centreon" ton.adresse@exemple.fr`.
2. **Configuration → Utilisateurs → Contacts / Utilisateurs → admin** : renseigner l'*E-mail*, activer les notifications d'hôte et de service avec les commandes **host-notify-by-email** et **service-notify-by-email**, période **24x7**.
3. Sur chaque hôte, onglet **Notification** : *Notifications activées* **Oui**, ajouter le contact `admin`.
4. **Exporter la configuration**.

!!! tip "Tester pour de vrai"
    Arrêter `snmpd` sur `srv-web02` (`systemctl stop snmpd`) : les services passent en **UNKNOWN**. Éteindre la VM : l'hôte passe **DOWN** et le mail arrive. Relancer et attendre le mail de retour (**RECOVERY**).

!!! note "Et le Centreon Monitoring Agent ?"
    Centreon propose désormais son propre agent (CMA), plus complet que SNMP pour Windows. Dans **Configuration → Collecteurs → Configurations d'agent**, le bouton **Command** génère la commande d'installation à lancer sur chaque hôte. C'est une bonne suite à ce tuto.

---

## ✅ Vérification

```bash
# Sur srv-centreon01
systemctl is-active mariadb cbd centengine gorgoned        # 4 × active
snmpwalk -v2c -c lab-ro 10.0.0.31 1.3.6.1.2.1.1.5         # sysName = srv-web01
```

Dans l'interface : **Supervision → Statut des ressources**, tous les hôtes **UP** et les services **OK** ; un mail de test reçu.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| Hôte **UP**, services **OK** | Tout fonctionne | Rien à faire |
| L'hôte n'apparaît pas dans la supervision | Configuration pas exportée | Étape 4 : **Exporter la configuration** avec *Déplacer les fichiers* |
| Services **UNKNOWN** : *SNMP Table Request: Timeout* | SNMP ne répond pas | Tester `snmpwalk`, vérifier la communauté, `rocommunity … 10.0.0.62`, pare-feu UDP 161 |
| Services **UNKNOWN** : *Can't find plugin* | Plugin pas installé sur le collecteur | `apt install centreon-plugin-operatingsystems-…` |
| Hôte **DOWN** alors qu'il répond en SSH | Le ping est bloqué | Autoriser l'ICMP (voir le tuto [Uptime Kuma](tuto-uptime-kuma.md), étape 5a) |
| Aucun mail reçu | Pas de relais SMTP ou notifications désactivées | `mail` depuis le serveur, `journalctl -u postfix`, onglet *Notification* de l'hôte |

## 🔧 Dépannage courant

!!! question "L'assistant d'installation bloque sur la base de données ?"
    Vérifier le mot de passe `root` défini par `mariadb-secure-installation` : `mariadb -uroot -p`. Si tu as refusé la connexion root « à distance » ce n'est pas grave, mais la connexion **locale** doit fonctionner.

!!! question "J'ai modifié un hôte mais rien ne change ?"
    Centreon applique les modifications **seulement après l'export** de la configuration (étape 4). C'est l'oubli numéro un.

!!! question "`snmpwalk` répond depuis la VM elle-même mais pas depuis Centreon ?"
    Sous Linux, vérifier `agentaddress udp:161` (et pas `127.0.0.1`). Sous Windows, vérifier la liste `PermittedManagers` et la règle de pare-feu.

!!! question "Le moteur refuse de redémarrer après un export ?"
    Utiliser **Vérifier la configuration** dans la fenêtre d'export : elle affiche l'erreur (souvent une commande ou un modèle manquant). Les journaux sont dans `/var/log/centreon-engine/`.

## Voir aussi

- 🧠 [La supervision : métriques, logs et alertes](concept-supervision.md)
- 🛠️ [Superviser ses VM avec Zabbix](tuto-zabbix.md) : l'autre grand outil d'entreprise, avec agents
- 🛠️ [Superviser ses VM avec Uptime Kuma](tuto-uptime-kuma.md) : plus simple, pour la disponibilité
