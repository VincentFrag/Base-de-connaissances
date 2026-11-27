---
icon: material/book-open-variant
title: Conteneurs Docker (image, conteneur, volume, réseau)
tags: [système, bases, concept, docker, conteneur, compose]
---

# 🧠 Conteneurs Docker : image, conteneur, volume, réseau

!!! abstract "En une phrase"
    **Docker** emballe une application avec tout ce dont elle a besoin (bibliothèques, configuration) dans une **image**, qu'on lance en **conteneur** isolé : elle tourne pareil sur ton PC, sur une VM du lab ou sur un serveur de production.

## L'analogie

Le transport maritime :

- l'**image** = le plan de chargement d'un conteneur, figé et réutilisable ;
- le **conteneur** = la boîte réelle, remplie d'après le plan, qu'on pose sur n'importe quel navire ;
- le **moteur Docker** = la grue et le navire, qui savent manipuler toutes les boîtes de la même façon ;
- le **volume** = la cale réfrigérée où l'on range ce qu'on veut garder quand la boîte est jetée ;
- le **réseau** = les passerelles entre boîtes d'un même navire ;
- le **registre** (Docker Hub) = l'entrepôt où l'on va chercher les plans.

---

## Les briques à connaître

| Brique | Ce que c'est | Exemple |
|---|---|---|
| **Image** | Modèle en lecture seule, en couches, identifié par `nom:tag` | `nginx:1.27`, `mariadb:11` |
| **Conteneur** | Une instance en cours d'exécution d'une image, avec sa couche inscriptible | `docker run -d --name web nginx:1.27` |
| **Registre** | Serveur qui stocke et distribue les images | Docker Hub, `ghcr.io`, registre privé |
| **Dockerfile** | Recette texte pour **construire** sa propre image | `FROM debian:12` puis `RUN apt install …` |
| **Volume** | Stockage géré par Docker, **survit** à la suppression du conteneur | `db_data:/var/lib/mysql` |
| **Bind mount** | Un dossier de l'hôte monté dans le conteneur | `./html:/usr/share/nginx/html` |
| **Réseau** | Réseau virtuel où les conteneurs se joignent **par leur nom** | `db` résolu en interne par le DNS de Docker |
| **Publication de port** | Expose un port du conteneur sur l'hôte | `-p 8080:80` → `http://10.0.0.40:8080` |
| **Docker Compose** | Fichier `compose.yaml` qui décrit plusieurs conteneurs, volumes et réseaux et les lance ensemble | `docker compose up -d` |

### Conteneur, LXC ou VM ?

| | **VM** (KVM) | **Conteneur LXC** (Proxmox) | **Conteneur Docker** |
|---|---|---|---|
| Ce qu'on isole | Une machine entière, avec son noyau | Un système Linux complet, noyau partagé | **Une application** (un processus principal) |
| Démarrage | 30 s à 1 min | Quelques secondes | Moins d'une seconde |
| Taille typique | Plusieurs Go | Quelques centaines de Mo | Quelques dizaines de Mo |
| On s'y connecte en SSH ? | Oui | Oui | Non : `docker exec`, `docker logs` |
| Durée de vie | Longue, on la met à jour | Longue, on la met à jour | **Jetable** : on remplace par une image plus récente |

!!! info "Les trois s'emboîtent"
    Dans un homelab Proxmox, on fait tourner **Docker dans une VM** (ou un LXC) : la VM isole du reste du lab, Docker isole les applications entre elles. Voir 🧠 [Virtualisation et hyperviseurs](concept-virtualisation.md).

### Ce qui est perdu, ce qui reste

| Où est la donnée | Après `docker rm` du conteneur |
|---|---|
| Écrite dans le conteneur (couche inscriptible) | ❌ **Perdue** |
| Dans un **volume** nommé | ✅ Conservée (supprimée seulement par `docker volume rm` ou `down -v`) |
| Dans un **bind mount** (dossier de l'hôte) | ✅ Conservée sur l'hôte |
| L'image elle-même | ✅ Conservée (supprimée par `docker image rm`) |

### Les réseaux Docker

| Type | Usage |
|---|---|
| `bridge` par défaut | Conteneurs lancés à la main ; ils ne se trouvent **pas** par leur nom |
| `bridge` **défini par l'utilisateur** (créé par Compose) | Chaque pile a son réseau ; les services se joignent par leur nom (`db`, `app`) |
| `host` | Le conteneur utilise directement la carte de l'hôte, sans isolation réseau |
| `none` | Aucun réseau |

---

## À quoi ça sert concrètement

- **Déployer un service en une commande** (wiki, supervision, gestionnaire de mots de passe) sans installer ses dépendances sur le système.
- **Mettre à jour sans risque** : `docker compose pull` puis `up -d` ; si ça casse, on revient au tag précédent.
- **Faire tourner plusieurs versions** d'un même logiciel côte à côte (PHP 8.1 et 8.3, MariaDB 10 et 11).
- **Reproduire un environnement** : le `compose.yaml` versionné dans Git décrit toute la pile.

!!! warning "Pièges fréquents"
    - Stocker les données **dans le conteneur** sans volume : elles disparaissent à la première mise à jour.
    - Utiliser le tag `latest` : une mise à jour peut changer de version majeure sans prévenir. **Fixer la version** (`mariadb:11`).
    - Publier une base de données sur l'hôte (`-p 3306:3306`) alors que seule l'application doit la joindre : le réseau interne de Compose suffit.
    - Mettre les mots de passe en clair dans `compose.yaml` versionné : les mettre dans un fichier `.env` **exclu de Git**.
    - Croire que le pare-feu de l'hôte protège les ports publiés : Docker écrit ses propres règles et **contourne** `ufw`. Publier seulement ce qui est nécessaire, ou sur `127.0.0.1:8080:80`.
    - Ajouter un utilisateur au groupe `docker` revient à lui donner les **droits root** sur l'hôte.

## Pour aller plus loin

- 🛠️ **[Déployer un service avec Docker Compose sur Proxmox](tuto-docker-compose-proxmox.md)** : la mise en pratique (installation, WordPress + MariaDB, volumes, mise à jour)
- 🧠 [Virtualisation et hyperviseurs](concept-virtualisation.md) : VM et LXC, où faire tourner Docker
- 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md) : créer la VM `srv-docker01`
- 🧠 [Sauvegardes](../transverse/concept-sauvegardes.md) : ne pas oublier les volumes

??? quote "Sources"
    - [Docker : vue d'ensemble](https://docs.docker.com/get-started/docker-overview/)
    - [Docker : volumes](https://docs.docker.com/engine/storage/volumes/)
    - [Docker : réseau](https://docs.docker.com/engine/network/)
