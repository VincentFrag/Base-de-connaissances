---
icon: material/wrench-outline
title: Déployer un service avec Docker Compose sur Proxmox
tags: [système, tuto, docker, compose, proxmox, lxc, debian]
---

# 🛠️ Déployer un service avec Docker Compose sur Proxmox

!!! abstract "Objectif"
    Préparer une VM (ou un conteneur LXC) Debian sur Proxmox, y installer **Docker** et le plugin **Compose**, puis déployer une pile **WordPress + MariaDB** avec un **volume** pour les données, un **réseau** interne et les mots de passe dans un fichier `.env`. Pour finir : mise à jour et sauvegarde des données.

**Niveau :** 🟡 Intermédiaire · **Durée :** 45 min

## Prérequis

- Comprendre image, conteneur, volume et réseau → 🧠 [Conteneurs Docker](concept-docker.md)
- Savoir cloner et personnaliser une VM → 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)
- Un hôte Proxmox `pve01` avec le réseau du lab `10.0.0.0/24` sur le pont `vmbr2`, et un accès Internet pour télécharger les images

## Le plan qu'on va monter

| Élément | Valeur |
|---|---|
| Machine Docker | `srv-docker01` · `10.0.0.40` · 2 vCPU, 2 Go RAM, 20 Go · Debian 12 |
| Passerelle / DNS | `10.0.0.1` / `10.0.0.10` |
| Compte | `admin` (membre du groupe `docker`) |
| Dossier de la pile | `/opt/wordpress` |
| Services | `db` (`mariadb:11`) et `wordpress` (`wordpress:6-apache`) |
| Accès | `http://10.0.0.40:8080` |

---

## Étape 1 — Préparer la machine

Docker tourne aussi bien dans une VM que dans un LXC. La **VM est plus simple** et plus isolée ; le LXC est plus léger mais demande deux options.

=== "VM Debian (recommandé)"

    1. Cloner le template `tpl-debian01` en **Full Clone**, nom `srv-docker01` (2 Go RAM, disque agrandi à 20 Go).
    2. Personnaliser le clone (nom, IP `10.0.0.40`, machine-id, clés SSH) comme dans 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md).
    3. Se connecter : `ssh admin@10.0.0.40`.

=== "Conteneur LXC"

    Sur l'hôte `pve01` (shell en root), télécharger un modèle Debian puis créer un conteneur **non privilégié** avec **nesting** activé :

    ```bash
    pveam update
    pveam available --section system | grep debian-12   # repérer le nom exact
    pveam download local debian-12-standard_12.7-1_amd64.tar.zst

    pct create 140 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
      --hostname srv-docker01 --cores 2 --memory 2048 --rootfs local-lvm:20 \
      --net0 name=eth0,bridge=vmbr2,ip=10.0.0.40/24,gw=10.0.0.1 \
      --nameserver 10.0.0.10 --unprivileged 1 --features nesting=1,keyctl=1 \
      --password
    pct start 140
    pct enter 140
    ```

    Dans le conteneur, créer le compte `admin` : `apt install -y sudo && adduser admin && usermod -aG sudo admin`.

    !!! warning "nesting=1 est indispensable"
        Sans lui, Docker refuse de démarrer ses conteneurs (`permission denied` sur `/proc` ou `/sys`). Sur un LXC existant : **Options → Features → Nesting** (et *keyctl*), puis redémarrer le LXC.

## Étape 2 — Installer Docker depuis le dépôt officiel

Le paquet `docker.io` de Debian est souvent ancien et n'inclut pas `docker compose`. On ajoute le dépôt de Docker :

```bash
sudo apt update && sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Autoriser `admin` à utiliser Docker sans `sudo`, puis **se déconnecter et se reconnecter** pour que le groupe soit pris en compte :

```bash
sudo usermod -aG docker admin
exit
```

!!! warning "Groupe docker = root"
    Un membre du groupe `docker` peut monter `/` dans un conteneur et tout modifier. Ne l'accorder qu'aux administrateurs.

## Étape 3 — Premier test

```bash
docker version                 # client ET serveur doivent répondre
docker compose version         # Docker Compose version v2.x
docker run --rm hello-world    # télécharge l'image, lance, affiche « Hello from Docker! », supprime
```

## Étape 4 — Écrire la pile Compose

Un dossier par pile, appartenant à `admin` :

```bash
sudo mkdir -p /opt/wordpress && sudo chown admin: /opt/wordpress
cd /opt/wordpress
```

Les secrets dans `.env` (lu automatiquement par Compose) :

```bash title="/opt/wordpress/.env"
MARIADB_ROOT_PASSWORD=ChangeMoi-Root
MARIADB_DATABASE=wordpress
MARIADB_USER=wp
MARIADB_PASSWORD=ChangeMoi-WP
```

```bash
chmod 600 .env
```

La description de la pile :

```yaml title="/opt/wordpress/compose.yaml"
services:
  db:
    image: mariadb:11
    restart: unless-stopped
    environment:
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
      MARIADB_DATABASE: ${MARIADB_DATABASE}
      MARIADB_USER: ${MARIADB_USER}
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql          # les données de la base survivent au conteneur
    networks:
      - interne
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      retries: 5

  wordpress:
    image: wordpress:6-apache
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy      # attendre que la base soit prête
    environment:
      WORDPRESS_DB_HOST: db             # le NOM du service, résolu par le DNS de Docker
      WORDPRESS_DB_NAME: ${MARIADB_DATABASE}
      WORDPRESS_DB_USER: ${MARIADB_USER}
      WORDPRESS_DB_PASSWORD: ${MARIADB_PASSWORD}
    volumes:
      - wp_data:/var/www/html
    ports:
      - "8080:80"                       # seul port publié sur l'hôte
    networks:
      - interne

volumes:
  db_data:
  wp_data:

networks:
  interne:
```

| Clé | Rôle |
|---|---|
| `image` | Image et **version fixée** à utiliser |
| `restart: unless-stopped` | Redémarre le conteneur après un plantage ou un reboot de la VM |
| `environment` | Variables passées au conteneur ; `${…}` vient de `.env` |
| `volumes` | Monte un volume nommé (déclaré en bas) dans le conteneur |
| `ports` | `hôte:conteneur` ; la base n'en a **pas** : elle n'est joignable que depuis `interne` |
| `depends_on` + `healthcheck` | Démarre WordPress seulement quand MariaDB répond |
| `networks` | Les deux services sont sur le même réseau et se voient par leur nom |

!!! tip "Valider avant de lancer"
    `docker compose config` affiche le fichier final avec les variables remplacées, ou l'erreur de syntaxe YAML (indentation, tabulation).

## Étape 5 — Lancer et observer

```bash
docker compose up -d           # télécharge les images, crée réseau, volumes et conteneurs
docker compose ps              # état : db « healthy », wordpress « running »
docker compose logs -f wordpress   # Ctrl+C pour quitter
```

Ce que Compose a créé (préfixé par le nom du dossier, `wordpress_`) :

```bash
docker network ls              # wordpress_interne
docker volume ls               # wordpress_db_data, wordpress_wp_data
docker exec -it wordpress-db-1 mariadb -u wp -p wordpress -e "SHOW TABLES;"
```

Ouvrir `http://10.0.0.40:8080` depuis un poste du lab (ou par rebond SSH, voir 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)) et terminer l'assistant d'installation de WordPress.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `db` en `starting` puis `healthy` | MariaDB initialise sa base au premier lancement | Patienter 20 à 30 s |
| `wordpress` en `Restarting` | Le conteneur plante en boucle | `docker compose logs wordpress` |
| « Error establishing a database connection » | Mauvais mot de passe ou base pas prête | Vérifier `.env` ; voir dépannage |
| `bind: address already in use` | Le port 8080 est déjà pris sur l'hôte | `sudo ss -tlnp` pour voir qui l’occupe, changer le port publié |
| `pull access denied` / `manifest unknown` | Nom ou tag d'image faux | Vérifier sur Docker Hub |

## Étape 6 — Prouver que les données survivent

```bash
docker compose down            # supprime conteneurs et réseau, GARDE les volumes
docker compose up -d
```

Le site WordPress est toujours là : les données étaient dans les volumes. À l'inverse, `docker compose down -v` supprime aussi les volumes, donc **tout le site**.

## Étape 7 — Mettre à jour

```bash
cd /opt/wordpress
docker compose pull            # télécharge les nouvelles versions des tags fixés (ex. mariadb:11.x)
docker compose up -d           # recrée uniquement les conteneurs dont l'image a changé
docker image prune             # supprime les anciennes images inutilisées
```

Pour une version majeure (ex. `mariadb:11` → `12`) : **sauvegarder d'abord**, modifier le tag dans `compose.yaml`, puis `up -d`.

## Étape 8 — Sauvegarder les données

Une sauvegarde de la VM par Proxmox couvre tout (voir 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](../transverse/tuto-proxmox-backup.md)). Pour une sauvegarde applicative :

```bash
cd /opt/wordpress
# Export logique de la base (cohérent même conteneur démarré)
docker compose exec db sh -c 'mariadb-dump -u root -p"$MARIADB_ROOT_PASSWORD" --all-databases' \
  > ~/wp-db-$(date +%F).sql

# Archive du volume des fichiers WordPress via un conteneur jetable
docker run --rm -v wordpress_wp_data:/data -v ~/:/backup debian:12 \
  tar czf /backup/wp-files-$(date +%F).tar.gz -C /data .
```

---

## ✅ Vérification finale

```bash
docker compose ps                                  # 2 services, db healthy
curl -sI http://localhost:8080 | head -1           # HTTP/1.1 200 OK (ou 302 vers l'installation)
docker volume ls | grep wordpress                  # 2 volumes
sudo reboot                                        # puis, une fois revenu :
docker compose -f /opt/wordpress/compose.yaml ps   # tout est reparti seul (restart: unless-stopped)
```

Depuis un autre poste du lab : `http://10.0.0.40:8080` affiche le site. Et le port de la base n'est **pas** joignable : `nc -zv 10.0.0.40 3306` doit échouer.

## 🔧 Dépannage courant

!!! question "`permission denied while trying to connect to the Docker daemon socket` ?"
    `admin` n'est pas (encore) dans le groupe `docker` pour cette session. Vérifier avec `id`, puis se déconnecter et se reconnecter après le `usermod -aG docker admin`.

!!! question "WordPress n'arrive pas à joindre la base alors que j'ai changé le mot de passe dans `.env` ?"
    MariaDB ne lit `MARIADB_*` qu'à la **première initialisation** du volume. Changer `.env` ensuite ne change rien dans la base. En lab : `docker compose down -v` puis `up -d` (⚠️ efface les données) ; sinon changer le mot de passe avec `ALTER USER` dans MariaDB.

!!! question "Dans un LXC : `failed to create shim task` ou `operation not permitted` ?"
    Il manque `nesting=1` (et `keyctl=1`) dans les *Features* du conteneur. L'activer dans Proxmox puis redémarrer le LXC. Si ça persiste, passer sur une VM.

!!! question "Le site répond en local (`curl localhost:8080`) mais pas depuis un autre poste ?"
    Vérifier la carte réseau de la VM dans Proxmox (pont `vmbr2`, case *Firewall*) et la route depuis le poste : 🛠️ [Diagnostiquer une connexion](../reseau/tuto-diagnostic-connexion.md).

!!! question "`docker compose` : commande inconnue, mais `docker-compose` existe ?"
    C'est l'ancienne version 1 (en Python, abandonnée). Installer `docker-compose-plugin` depuis le dépôt officiel (étape 2) et utiliser `docker compose`, avec un espace.

## 📋 Mémo des commandes

| Besoin | Commande |
|---|---|
| Lancer / recréer la pile | `docker compose up -d` |
| Arrêter et supprimer (garder les données) | `docker compose down` |
| État des services | `docker compose ps` |
| Journaux en direct | `docker compose logs -f <service>` |
| Ouvrir un shell dans un conteneur | `docker compose exec <service> sh` |
| Mettre à jour les images | `docker compose pull && docker compose up -d` |
| Lister images / volumes / réseaux | `docker image ls` · `docker volume ls` · `docker network ls` |
| Place utilisée par Docker | `docker system df` |
| Nettoyer images et conteneurs inutilisés | `docker system prune` |

## Voir aussi

- 🧠 [Conteneurs Docker : image, conteneur, volume, réseau](concept-docker.md)
- 🧠 [Virtualisation et hyperviseurs](concept-virtualisation.md) (VM ou LXC)
- 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](../transverse/tuto-proxmox-backup.md)
