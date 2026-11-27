---
icon: material/book-open-variant
title: La répartition de charge (load balancing)
tags: [système, bases, concept, load balancing, haproxy, haute disponibilité, web]
---

# 🧠 La répartition de charge (load balancing)

!!! abstract "En une phrase"
    Un **répartiteur de charge** (*load balancer*) reçoit toutes les requêtes des clients à une **adresse unique** et les distribue entre **plusieurs serveurs** identiques, pour encaisser plus de trafic et continuer à répondre si l'un d'eux tombe.

## L'analogie

C'est l'**hôtesse d'accueil d'un restaurant** : les clients entrent tous par la même porte, et elle les place à tour de rôle chez chaque serveur de salle. Si un serveur s'absente, elle arrête simplement de lui envoyer du monde. Les clients, eux, ne voient que la porte d'entrée.

## Comment ça marche

```
                                 ┌──▶ srv-web01  10.0.0.31
[Clients] ──▶ [srv-lb01 HAProxy] ┤
                 10.0.0.30       └──▶ srv-web02  10.0.0.32
```

| Élément | Rôle | Dans HAProxy |
|---|---|---|
| **Frontend** | La « porte d'entrée » : IP et port où arrivent les clients | `frontend` + `bind *:80` |
| **Backend** | Le groupe de serveurs qui font le vrai travail | `backend` + lignes `server` |
| **Algorithme** | La règle de choix du serveur | `balance roundrobin` |
| **Health check** | Test régulier de chaque serveur ; un serveur KO est sorti du lot | `check` + `option httpchk` |
| **Page de stats** | Tableau de bord : état des serveurs, nombre de requêtes | `stats enable` |

## Les algorithmes courants

| Algorithme | Principe | Quand l'utiliser |
|---|---|---|
| `roundrobin` | Chacun son tour : 1, 2, 1, 2… | Serveurs équivalents, requêtes courtes (sites web) |
| `leastconn` | Le serveur qui a **le moins de connexions** en cours | Connexions longues (bases de données, WebSocket) |
| `source` | Même IP client → toujours le même serveur | Applis qui gardent une **session** en mémoire locale |

On peut aussi donner un **poids** (`weight`) à chaque serveur : un serveur `weight 2` reçoit deux fois plus de requêtes.

## Couche 4 ou couche 7 ?

| Mode | Ce que le répartiteur regarde | Possibilités |
|---|---|---|
| **TCP** (couche 4) | IP et port seulement | Répartit n'importe quel service (SSH, SQL, mail) |
| **HTTP** (couche 7) | Le contenu de la requête : URL, en-têtes, cookies | Router `/admin` vers un serveur précis, ajouter des en-têtes, vérifier une page |

HAProxy sait faire les deux ; pour un site web, on travaille en **HTTP**.

## À quoi ça sert concrètement

- **Monter en charge** : ajouter un serveur web suffit pour absorber plus de visiteurs.
- **Haute disponibilité** : un serveur en panne ou en maintenance est contourné automatiquement.
- **Maintenance sans coupure** : on met à jour les serveurs un par un.
- **Point d'entrée unique** : un seul endroit pour le certificat HTTPS, les règles d'accès, les logs.

!!! warning "Pièges fréquents"
    - **Contenu différent sur chaque serveur** : le client voit une page différente à chaque rafraîchissement. Les backends doivent servir **le même contenu**, d'où l'intérêt d'un **partage commun** (voir 🧠 [NFS](concept-nfs.md)).
    - **Sessions perdues** : si l'appli garde la connexion de l'utilisateur en mémoire, le roundrobin l'envoie sur un autre serveur qui ne le connaît pas. Utiliser `balance source` ou un cookie de persistance.
    - **Le répartiteur devient le point unique de panne** : s'il tombe, plus rien ne répond. En production, on en met deux avec une IP virtuelle partagée (keepalived/VRRP).
    - **Health check trop naïf** : un serveur qui répond au ping mais dont le site plante reste dans le lot. Préférer un test HTTP (`option httpchk`).

## Pour aller plus loin

- 🛠️ **[Répartir la charge entre deux serveurs web avec HAProxy](tuto-haproxy-repartition-charge.md)** : la mise en pratique
- 🧠 [Le partage de fichiers NFS](concept-nfs.md) (servir le même contenu sur tous les backends)
- 🧪 [TP : Load balancer HAProxy et pages web sur NFS](tp-load-balancer-haproxy-nfs.md)

??? quote "Sources"
    - [HAProxy : documentation officielle (manuel de configuration)](https://docs.haproxy.org/) : sections `balance`, `server`, `option httpchk`
    - [HAProxy : The Four Essential Sections of an HAProxy Configuration](https://www.haproxy.com/blog/the-four-essential-sections-of-an-haproxy-configuration) (blog de l'éditeur)
