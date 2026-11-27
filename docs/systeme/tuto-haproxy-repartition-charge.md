---
icon: material/wrench-outline
title: Répartir la charge entre deux serveurs web avec HAProxy
tags: [système, tuto, haproxy, load balancing, apache, nginx, debian, proxmox]
---

# 🛠️ Répartir la charge entre deux serveurs web avec HAProxy

!!! abstract "Objectif"
    Placer un répartiteur **HAProxy** devant deux serveurs web (**Apache** et **Nginx**), en **roundrobin**, puis **prouver** que les requêtes sont réparties à 50/50 et que le service tient la charge.

**Niveau :** 🟡 Intermédiaire · **Durée :** 30 min

## Prérequis

- Comprendre le principe → 🧠 [La répartition de charge](concept-repartition-charge.md)
- Trois VM Debian préparées (nom, IP, machine-id, clés SSH) → 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)
- Un accès SSH aux VM, ici par rebond via Proxmox → 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)

## Le lab

| VM | IP | Rôle |
|---|---|---|
| `srv-lb01` | `10.0.0.30` | Répartiteur HAProxy |
| `srv-web01` | `10.0.0.31` | Backend Apache |
| `srv-web02` | `10.0.0.32` | Backend Nginx |

Réseau `10.0.0.0/24` sur le pont `vmbr2` · passerelle `10.0.0.1` (Proxmox) · DNS `10.0.0.10` (`srv-dc01`, domaine `lab.local`)

```
                                ┌──▶ srv-web01 .31 (Apache)
PC / client ──▶ srv-lb01 .30:80 ┤
                                └──▶ srv-web02 .32 (Nginx)
```

Toutes les commandes se font en **root** (`ssh srv-lb01` puis `su -`).

---

## Étape 1 — Installer les deux serveurs web

Chaque backend affiche une page **différente**, pour voir à l'œil nu qui a répondu.

=== "srv-web01"
    ```bash
    apt install -y apache2
    echo "<h1>Backend APACHE</h1>" > /var/www/html/index.html
    ```

=== "srv-web02"
    ```bash
    apt install -y nginx
    echo "<h1>Backend NGINX</h1>" > /var/www/html/index.html
    ```

Vérifie chaque backend **seul**, depuis `srv-lb01` :

```bash
curl -s http://10.0.0.31     # <h1>Backend APACHE</h1>
curl -s http://10.0.0.32     # <h1>Backend NGINX</h1>
```

!!! tip "Toujours tester les backends avant le répartiteur"
    Si un backend ne répond pas en direct, HAProxy ne fera pas mieux. On élimine ce suspect d'abord.

## Étape 2 — Installer et configurer HAProxy

Sur `srv-lb01` :

```bash
apt install -y haproxy
nano /etc/haproxy/haproxy.cfg
```

Le fichier livré par Debian contient déjà les sections `global` et `defaults` (dont `mode http`) : on **ajoute** à la fin :

```text
frontend http_front
    bind *:80
    default_backend web_back

backend web_back
    balance roundrobin
    option httpchk GET /
    http-response set-header X-Backend %s
    server apache 10.0.0.31:80 check
    server nginx  10.0.0.32:80 check

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 5s
```

| Ligne | Rôle |
|---|---|
| `bind *:80` | HAProxy écoute sur le port 80 de toutes ses interfaces |
| `balance roundrobin` | Chaque serveur à tour de rôle |
| `option httpchk GET /` | Le test de santé demande la page `/` (et pas seulement « le port est ouvert ») |
| `http-response set-header X-Backend %s` | Ajoute à chaque réponse un en-tête avec le **nom du serveur** qui a répondu (`%s`) : pratique pour compter |
| `server … check` | Déclare un backend et active son test de santé |
| `listen stats` | Page de supervision sur le port 8404 |

Vérifie la syntaxe **avant** de redémarrer :

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl restart haproxy
systemctl status haproxy --no-pager
```

!!! info "`haproxy -c` peut ne rien afficher"
    Selon la version, une configuration valide affiche `Configuration file is valid`… ou **rien du tout** (cas des versions récentes). Ce qui compte, c'est l'absence d'erreur : grâce au `&&`, le redémarrage ne se fait que si la vérification réussit.

## Étape 3 — Tester la répartition

**Alternance** : 4 requêtes doivent alterner APACHE / NGINX.

```bash
for i in 1 2 3 4; do curl -s http://10.0.0.30; done
```

**Comptage 50/50** sur 100 requêtes, grâce à l'en-tête `X-Backend` :

```bash
for i in $(seq 1 100); do curl -sI http://10.0.0.30/ | grep -i x-backend; done | sort | uniq -c
```

Résultat attendu :

```text
     50 x-backend: apache
     50 x-backend: nginx
```

**Tenue en charge** avec ApacheBench : 1 000 requêtes, 10 en parallèle.

```bash
apt install -y apache2-utils
ab -n 1000 -c 10 http://10.0.0.30/
```

Dans le résultat, regarde `Failed requests: 0` et `Requests per second`.

!!! note "Pourquoi « Length » peut apparaître dans les échecs de `ab`"
    `ab` compare la taille de chaque réponse à la première. Comme les deux pages n'ont pas la même longueur, il peut compter des `Failed requests (Length: …)`. Ce ne sont pas de vraies erreurs : elles disparaissent quand les deux backends servent le **même** contenu (voir 🛠️ [NFS](tuto-partage-nfs.md)).

**Page de stats** : depuis un navigateur qui joint le lab, `http://10.0.0.30:8404/stats`. Chaque serveur doit être **vert** (*UP*).

## Étape 4 — Simuler une panne

```bash
# Sur srv-web02
systemctl stop nginx
```

```bash
# Sur srv-lb01 : après quelques secondes, toutes les réponses viennent d'Apache
for i in 1 2 3 4; do curl -s http://10.0.0.30; done
```

Sur la page de stats, `nginx` passe au **rouge** (*DOWN*). Relance-le (`systemctl start nginx`) : il revient tout seul dans la boucle.

---

## ✅ Vérification

| Test | Résultat attendu |
|---|---|
| `curl` ×4 sur `10.0.0.30` | Alternance APACHE / NGINX |
| Comptage `X-Backend` sur 100 requêtes | 50 / 50 |
| `ab -n 1000 -c 10` | `Failed requests: 0` (hors écarts de longueur) |
| Page `:8404/stats` | Les deux serveurs UP |
| Arrêt d'un backend | Le service continue sur l'autre |

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `503 Service Unavailable` | Aucun backend n'est jugé en bonne santé | Tester chaque backend en direct avec `curl` ; regarder la page de stats |
| Toujours le même backend | L'autre est DOWN, ou `balance` n'est pas `roundrobin` | Page de stats ; relire la section `backend` |
| `Connection refused` sur `.30` | HAProxy ne tourne pas ou n'écoute pas sur 80 | `systemctl status haproxy` ; `ss -tlnp \| grep :80` |
| HAProxy refuse de démarrer | Erreur dans le fichier | `haproxy -c -f /etc/haproxy/haproxy.cfg` affiche la ligne fautive |
| Pas de ligne `X-Backend` dans le comptage | HAProxy envoie les en-têtes **en minuscules** (`x-backend`) et `grep` respecte la casse ; ou `http-response set-header` absent | `curl -sI http://10.0.0.30/` pour voir les en-têtes |

## 🔧 Dépannage courant

!!! question "Tout tourne dans les VM, mais rien ne répond depuis une autre machine ?"
    Si la carte réseau des VM a la case **Firewall** cochée dans Proxmox (*Hardware → Network Device*) et que le pare-feu Proxmox est actif avec des règles restrictives, il filtre le trafic : décoche la case pour tester.

!!! question "Le test de santé met un serveur DOWN alors que le site s'affiche ?"
    `option httpchk GET /` attend une réponse **2xx ou 3xx**. Si `/` renvoie 403 ou 404 (page absente, mauvais droits), le serveur est jugé en panne. Teste avec `curl -I http://10.0.0.31/`.

!!! question "Je n'ai pas de navigateur qui joint le réseau du lab pour la page de stats ?"
    Fais passer le port dans le rebond SSH depuis ton PC : `ssh -L 8404:10.0.0.30:8404 pve01`, puis ouvre `http://localhost:8404/stats`.

## Pour aller plus loin

- 🛠️ [Partager un dossier en NFS et le monter sur les serveurs web](tuto-partage-nfs.md) : servir le même contenu sur les deux backends
- 🛠️ [Protéger une page web par un groupe AD](tuto-apache-auth-ad.md) : router `/admin` vers un seul backend
- 🧪 [TP : Load balancer HAProxy et pages web sur NFS](tp-load-balancer-haproxy-nfs.md)

## Sources

- [HAProxy : manuel de configuration](https://docs.haproxy.org/) : `frontend`, `backend`, `balance roundrobin`, `option httpchk`, `http-response set-header`, `stats`, format de log `%s` (nom du serveur), option `h1-case-adjust` (casse des en-têtes)
- [ApacheBench (`ab`)](https://httpd.apache.org/docs/2.4/programs/ab.html) : documentation Apache
- [Paquet Debian `haproxy`](https://packages.debian.org/stable/haproxy)
