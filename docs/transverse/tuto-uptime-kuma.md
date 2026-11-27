---
icon: material/wrench-outline
title: Superviser ses VM avec Uptime Kuma
tags: [transverse, tuto, supervision, monitoring, uptime-kuma, docker, proxmox]
---

# 🛠️ Superviser ses VM avec Uptime Kuma

!!! abstract "Objectif"
    Installer **Uptime Kuma** sur une VM dédiée `srv-mon01`, y déclarer des sondes pour les VM du lab (**ping**, **site web**, **port TCP**, **DNS**, interface Proxmox), recevoir une **alerte** quand l'une d'elles tombe, vérifier que la **sauvegarde de la nuit** a tourné, puis publier une **page de statut**.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 1 h

## Prérequis

- Savoir ce qu'on supervise et pourquoi → 🧠 [La supervision : métriques, logs et alertes](concept-supervision.md)
- Une VM Debian clonée depuis le template → 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md)
- Joindre le réseau interne depuis ton PC → 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)
- Pour l'étape 6 : une tâche de sauvegarde Proxmox → 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](tuto-proxmox-backup.md)

## Le plan qu'on va monter

| Machine | IP | Ce qu'on vérifie | Type de sonde |
|---|---|---|---|
| `srv-mon01` (nouvelle VM) | `10.0.0.60` | Héberge Uptime Kuma (port `3001`) | — |
| `pve01` | `10.0.0.1` | Interface web Proxmox | HTTP(s) sur `:8006` |
| `srv-dc01` | `10.0.0.10` | Joignable + résout `lab.local` | Ping + DNS |
| `srv-files01` | `10.0.0.20` | Joignable + SSH | Ping + port TCP 22 |
| `srv-lb01` | `10.0.0.30` | Le site répond (via le répartiteur) | HTTP |
| `srv-web01`, `srv-web02` | `10.0.0.31`, `10.0.0.32` | Chaque serveur web répond | HTTP |
| Sauvegarde nocturne | — | Le job s'est terminé sans erreur | Push |

```
                 ┌──── ping / HTTP / TCP / DNS toutes les 60 s ────▶ srv-dc01, srv-web01…
[srv-mon01]──────┤
 Uptime Kuma     └◀─── « je suis vivant » après chaque sauvegarde ──── pve01 (vzdump)
      │
      └── 🔔 alerte ──▶ Discord / mail / Telegram
```

!!! warning "Pourquoi une VM à part ?"
    Si Uptime Kuma tournait sur `srv-web01` et que cette VM tombait, personne ne serait prévenu. La supervision vit **à côté** de ce qu'elle surveille. Pour aller plus loin : une seconde instance hors du Proxmox (Raspberry Pi, autre PC) qui surveille `pve01` lui-même.

---

## Étape 1 — Préparer la VM `srv-mon01`

Cloner `tpl-debian01` en `srv-mon01` (1 vCPU, 1 Go de RAM, 10 Go de disque suffisent) et la personnaliser comme dans [Préparer une VM clonée](../systeme/tuto-preparer-clone-vm.md), avec l'IP `10.0.0.60/24`, la passerelle `10.0.0.1` et le DNS `10.0.0.10`.

```bash
hostnamectl                # Static hostname: srv-mon01
ip -br a                   # ens18  UP  10.0.0.60/24
ping -c 2 10.0.0.31        # srv-web01 répond
```

## Étape 2 — Installer Docker

Uptime Kuma est distribué en image Docker : c'est la méthode la plus simple à installer et à mettre à jour. On ajoute le dépôt officiel de Docker (en `root`, ou avec `sudo` depuis le compte `admin`).

```bash
apt update && apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list
apt update && apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
docker run --rm hello-world          # « Hello from Docker! »
```

!!! tip "Lancer Docker sans `sudo` depuis le compte `admin`"
    `usermod -aG docker admin`, puis se déconnecter et se reconnecter. Attention : être dans le groupe `docker` revient à être `root` sur la machine.

## Étape 3 — Lancer Uptime Kuma

On décrit le conteneur dans un fichier Compose, pour pouvoir le recréer à l'identique.

```bash
mkdir -p /opt/uptime-kuma && cd /opt/uptime-kuma
cat > compose.yaml <<'EOF'
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
EOF
docker compose up -d
docker compose ps                    # STATUS : Up … (healthy)
docker compose logs -f               # Ctrl+C pour quitter
```

| Ligne | Rôle |
|---|---|
| `image: louislam/uptime-kuma:2` | Version 2 de l'image officielle (les mises à jour mineures arrivent avec `docker compose pull`) |
| `restart: unless-stopped` | Le conteneur redémarre tout seul avec la VM |
| `ports: "3001:3001"` | L'interface web est sur `http://10.0.0.60:3001` |
| `volumes: ./data:/app/data` | Toute la configuration et l'historique restent dans `/opt/uptime-kuma/data`, même si le conteneur est recréé |

!!! note "Sans Docker ?"
    Uptime Kuma s'installe aussi avec Node.js et `pm2` (voir la documentation officielle), mais les mises à jour sont plus pénibles. Docker reste le choix conseillé.

## Étape 4 — Ouvrir l'interface depuis ton PC

Le réseau `10.0.0.0/24` n'est pas joignable directement depuis le PC : on fait passer la page web dans un **tunnel SSH** par `pve01`.

=== "Windows"
    ```powershell
    ssh -L 3001:10.0.0.60:3001 root@192.168.1.20
    # ou, si l'alias existe dans ~/.ssh/config :
    ssh -L 3001:10.0.0.60:3001 pve01
    ```

=== "Linux"
    ```bash
    ssh -L 3001:10.0.0.60:3001 root@192.168.1.20
    # ou : ssh -L 3001:10.0.0.60:3001 pve01
    ```

Laisser le terminal ouvert, puis aller sur **http://localhost:3001** :

1. Choisir la langue (**Français** disponible) et la base de données **SQLite** (intégrée, parfaite pour un homelab).
2. Créer le compte administrateur : `admin` + un mot de passe solide.
3. **Paramètres → Général** : régler le fuseau horaire sur `Europe/Paris`.

## Étape 5 — Ajouter les sondes

Bouton **+ Ajouter une sonde**. Pour chaque sonde, les réglages communs :

| Réglage | Valeur conseillée | Pourquoi |
|---|---|---|
| *Intervalle de vérification* | `60` s | Assez réactif sans charger le réseau |
| *Nouvelles tentatives* | `2` | 3 échecs d'affilée avant l'alerte : évite les fausses alertes |
| *Intervalle des nouvelles tentatives* | `30` s | Confirme vite une vraie panne |
| *Étiquettes* | `lab`, `web`, `infra`… | Trier et filtrer le tableau de bord |

### 5a. Ping : la machine est-elle joignable ?

*Type* **Ping** · *Nom* `srv-dc01 (ping)` · *Nom d'hôte* `10.0.0.10`. Refaire pour `srv-files01` (`10.0.0.20`) et les autres VM.

Windows Server **bloque le ping** par défaut : la sonde de `srv-dc01` reste rouge tant qu'on ne l'autorise pas.

=== "Windows Server (PowerShell)"
    ```powershell
    # Autoriser la réponse au ping (ICMPv4) sur srv-dc01
    Enable-NetFirewallRule -Name FPS-ICMP4-ERQ-In
    # Ou créer la règle explicitement :
    New-NetFirewallRule -DisplayName "Ping depuis srv-mon01" -Protocol ICMPv4 -IcmpType 8 `
      -RemoteAddress 10.0.0.60 -Direction Inbound -Action Allow
    ```

=== "Linux (nftables)"
    ```bash
    # Debian répond au ping par défaut. Si un pare-feu nftables bloque tout :
    nft add rule inet filter input ip saddr 10.0.0.60 icmp type echo-request accept
    ```

### 5b. HTTP : le site répond-il vraiment ?

*Type* **HTTP(s)** · *Nom* `Site (via srv-lb01)` · *URL* `http://10.0.0.30`. Le site est **UP** si le serveur renvoie un code `2xx`.

Refaire pour `http://10.0.0.31` (`srv-web01`) et `http://10.0.0.32` (`srv-web02`) : si un seul serveur web tombe, le répartiteur continue de servir le site, et seule la sonde du serveur concerné passe au rouge.

!!! tip "Vérifier le contenu, pas seulement la réponse"
    Le type **HTTP(s) - Mot-clé** vérifie qu'un texte précis est dans la page (ex. le titre du site). Une page d'erreur qui renvoie `200` sera alors détectée.

Pour l'interface Proxmox : *Type* **HTTP(s)** · *URL* `https://10.0.0.1:8006` · cocher **Ignorer les erreurs TLS/SSL** (certificat auto-signé).

### 5c. Port TCP : le service écoute-t-il ?

*Type* **Port TCP** · *Nom* `srv-files01 (SSH)` · *Nom d'hôte* `10.0.0.20` · *Port* `22`.

Autres ports utiles du lab :

| Service | Machine | Port |
|---|---|---|
| SSH | toutes les VM Linux | `22` |
| LDAP (Active Directory) | `srv-dc01` | `389` |
| Partage SMB | `srv-files01` | `445` |
| NFS | `srv-files01` | `2049` |

### 5d. DNS : le contrôleur de domaine résout-il les noms ?

*Type* **DNS** · *Nom* `srv-dc01 (DNS)` · *Nom d'hôte à résoudre* `srv-web01.lab.local` · *Serveur DNS* `10.0.0.10` · *Port* `53` · *Type d'enregistrement* `A`.

Si la résolution échoue, la sonde passe au rouge même si `srv-dc01` répond au ping : c'est exactement la différence entre « la machine est là » et « le service fonctionne ».

## Étape 6 — Recevoir les alertes

**Paramètres → Notifications → Configurer une notification**. Choisir un canal, cocher **Activé par défaut** et **Appliquer à toutes les sondes existantes**, puis **Tester** avant d'enregistrer.

=== "Discord"
    1. Sur ton serveur Discord : salon → **Modifier le salon → Intégrations → Webhooks → Nouveau webhook** → **Copier l'URL**.
    2. Uptime Kuma : *Type* **Discord** · coller l'*URL du webhook* → **Tester**.

=== "Mail (SMTP)"
    *Type* **Email (SMTP)** :

    - *Serveur SMTP* : celui de ta messagerie (ex. `smtp.gmail.com`, port `587`, sécurité **STARTTLS**)
    - *Nom d'utilisateur* : l'adresse d'envoi · *Mot de passe* : un **mot de passe d'application** (pas celui du compte)
    - *De* / *À* : l'adresse d'envoi et celle qui reçoit les alertes → **Tester**

=== "Telegram"
    1. Dans Telegram, parler à **@BotFather** → `/newbot` → récupérer le **jeton** du bot.
    2. Envoyer un message à ton bot, puis dans Uptime Kuma : *Type* **Telegram** · coller le jeton → **Récupérer automatiquement l'ID de chat** → **Tester**.

!!! tip "Tester pour de vrai"
    Arrêter Apache sur `srv-web02` (`systemctl stop apache2`), attendre l'alerte 🔴 (environ 2 min avec les réglages de l'étape 5), puis le relancer (`systemctl start apache2`) et attendre le message ✅ de retour.

## Étape 7 — Vérifier la sauvegarde de la nuit (sonde Push)

Une sauvegarde ne s'interroge pas avec un ping : c'est elle qui doit **signaler** qu'elle a terminé. Si le signal n'arrive pas dans les temps, Uptime Kuma alerte.

1. **+ Ajouter une sonde** → *Type* **Push** · *Nom* `Sauvegarde nocturne pve01`.
2. *Intervalle de vérification* : `90000` s (25 h : une sauvegarde par jour, avec une marge).
3. Copier l'**URL Push** affichée, du type `http://10.0.0.60:3001/api/push/AbC123xYz?status=up&msg=OK&ping=` → **Enregistrer**.

Sur `pve01`, un **script crochet** (*hook*) de vzdump appelle cette URL à la fin de chaque tâche de sauvegarde :

```bash
cat > /usr/local/bin/vzdump-kuma.sh <<'EOF'
#!/bin/bash
# Appelé par vzdump à chaque phase ; $1 = nom de la phase
URL="http://10.0.0.60:3001/api/push/AbC123xYz"   # ← ton URL, sans les paramètres
case "$1" in
  job-end)   curl -fsS -m 10 "$URL?status=up&msg=Sauvegarde%20OK" >/dev/null ;;
  job-abort) curl -fsS -m 10 "$URL?status=down&msg=Sauvegarde%20en%20echec" >/dev/null ;;
esac
exit 0
EOF
chmod +x /usr/local/bin/vzdump-kuma.sh
echo "script: /usr/local/bin/vzdump-kuma.sh" >> /etc/vzdump.conf
```

Lancer la tâche planifiée à la main (**Datacenter → Backup → Run now**) : la sonde passe au vert dès la fin de la sauvegarde.

| Ce qui se passe la nuit | Ce que fait Uptime Kuma |
|---|---|
| La sauvegarde se termine | Reçoit `status=up` → ✅ |
| La sauvegarde échoue | Reçoit `status=down` → 🔴 alerte immédiate |
| La tâche ne se lance pas (désactivée, `pve01` éteint) | Aucun signal pendant 25 h → 🔴 alerte |

!!! note "Même principe pour n'importe quelle tâche planifiée"
    Un script `cron` sous Linux ou une tâche planifiée Windows peut appeler l'URL Push à la fin de son travail : `curl -fsS "<url>?status=up"` sous Linux, `Invoke-RestMethod "<url>?status=up"` en PowerShell.

## Étape 8 — Publier une page de statut

**Pages de statut → + Nouvelle page de statut** · *Nom* `Lab` · *Slug* `lab`. Ajouter des groupes (ex. *Infrastructure* : `pve01`, `srv-dc01` ; *Web* : site, `srv-web01`, `srv-web02`), puis **Enregistrer**.

La page est visible sans compte sur `http://10.0.0.60:3001/status/lab` : pratique sur un écran, ou pour montrer la disponibilité du lab en soutenance.

!!! tip "Prévoir les coupures"
    Avant une mise à jour, déclarer une **Maintenance** (menu **Maintenance**) sur les sondes concernées : elles passent en bleu et n'envoient pas d'alerte pendant la fenêtre.

---

## ✅ Vérification

```bash
# Sur srv-mon01
docker compose -f /opt/uptime-kuma/compose.yaml ps   # uptime-kuma  Up (healthy)
ss -tlnp | grep 3001                                  # le port 3001 écoute
# Sur pve01
grep script /etc/vzdump.conf                          # le crochet est déclaré
```

Dans l'interface : toutes les sondes en vert, une notification de test reçue, la sonde Push verte après une sauvegarde, la page `/status/lab` accessible.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| Toutes les sondes vertes, % de disponibilité qui monte | Tout fonctionne | Refaire le test d'alerte de temps en temps |
| Sonde **Ping** rouge sur `srv-dc01` mais le DNS est vert | Le pare-feu Windows bloque l'ICMP | Autoriser le ping (étape 5a) |
| Sonde **HTTP** rouge, **Ping** vert | La VM tourne mais le serveur web est arrêté | `systemctl status apache2` (ou `nginx`) sur la VM |
| `ECONNREFUSED` sur une sonde TCP | Rien n'écoute sur ce port | Vérifier le service et le port (`ss -tlnp`) |
| `ETIMEDOUT` / `timeout of 48000ms exceeded` | Paquets bloqués en route | Pare-feu de la VM ou case *Firewall* de la carte réseau dans Proxmox |
| Sonde **Push** rouge le matin | La sauvegarde a échoué ou ne s'est pas lancée | Journal des tâches de `pve01` (**Datacenter → Backup → Job Detail**) |
| Sondes en **bleu** | Une maintenance est en cours | Normal, ou terminer la maintenance |

## 🔧 Dépannage courant

!!! question "`http://localhost:3001` ne s'ouvre pas ?"
    Le tunnel SSH de l'étape 4 doit rester ouvert. Vérifier aussi sur `srv-mon01` que le conteneur tourne (`docker compose ps`) et que le port écoute (`ss -tlnp | grep 3001`).

!!! question "Le conteneur redémarre en boucle ?"
    Lire `docker compose logs`. Le plus souvent : le dossier `data` n'est pas accessible en écriture ou le disque de la VM est plein (`df -h`).

!!! question "Aucune alerte reçue alors que la sonde est rouge ?"
    Ouvrir la sonde → **Modifier** → section *Notifications* : la case du canal doit être cochée. Le bouton **Tester** de la notification doit fonctionner. Pour le mail, regarder les spams et utiliser un mot de passe d'application.

!!! question "La sonde Push ne passe jamais au vert ?"
    Depuis `pve01`, tester l'URL à la main : `curl -v "http://10.0.0.60:3001/api/push/AbC123xYz?status=up"`. Une réponse `{"ok":true}` veut dire que le lien fonctionne : vérifier alors la ligne `script:` dans `/etc/vzdump.conf` et que le script est exécutable.

!!! question "Comment mettre à jour Uptime Kuma ?"
    Sauvegarder d'abord le dossier de données, puis :
    ```bash
    cd /opt/uptime-kuma
    tar czf ~/uptime-kuma-data-$(date +%F).tar.gz data
    docker compose pull && docker compose up -d
    ```

## Voir aussi

- 🧠 [La supervision : métriques, logs et alertes](concept-supervision.md)
- 🛠️ [Superviser ses VM avec Zabbix](tuto-zabbix.md) : l'étape suivante, pour voir CPU, RAM et disques
- 🛠️ [Superviser ses VM avec Centreon](tuto-centreon.md) : la supervision par SNMP, très répandue en entreprise
- 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](tuto-proxmox-backup.md)
- 🛠️ [Diagnostiquer une connexion](../reseau/tuto-diagnostic-connexion.md)
- 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)

??? quote "Sources"
    *Uptime Kuma n'a pas de site de documentation séparé : la référence est le dépôt GitHub du projet et son wiki.*

    - [Uptime Kuma : dépôt officiel](https://github.com/louislam/uptime-kuma)
    - [Uptime Kuma : wiki officiel](https://github.com/louislam/uptime-kuma/wiki)
