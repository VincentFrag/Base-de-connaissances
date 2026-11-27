---
icon: material/flask-outline
title: "TP : Load balancer HAProxy, pages web sur NFS et accès admin par AD"
tags: [système, tp, haproxy, load balancing, nfs, apache, nginx, active directory, ldap, proxmox]
---

# 🧪 TP : Load balancer HAProxy, pages web sur NFS et accès admin par AD

!!! abstract "Ce que tu vas faire"
    Mettre un site web derrière un **répartiteur de charge**, servir ses pages depuis un **partage NFS** commun, puis réserver une zone `/admin` aux membres d'un **groupe Active Directory**.

**Niveau :** 🟡 Intermédiaire · **Durée :** 3 h environ · **Outil :** Proxmox (VM Debian)

## Avant de commencer

Ce TP mobilise :

- 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)
- 🛠️ [SSH par rebond vers un réseau interne](../reseau/tuto-ssh-rebond.md)
- 🧠 [La répartition de charge](concept-repartition-charge.md) et 🧠 [NFS](concept-nfs.md)
- 🛠️ [Monter un AD sur une VM Proxmox](tuto-monter-ad-proxmox.md) (un DC Samba AD convient aussi)

!!! tip "Comment utiliser ce TP"
    Pour chaque tâche, essaie d'abord **seul**. Bloqué ? Ouvre l'**💡 Indice**. Toujours bloqué, ou pour vérifier ? Ouvre la **✅ Solution**.

---

## Le contexte

La société **WebLab** héberge son site vitrine sur un seul serveur, qui sature aux heures de pointe et coupe le site à chaque mise à jour. La DSI veut :

- **deux** serveurs web de technologies différentes (pour ne pas dépendre d'un seul logiciel), derrière **une seule adresse** ;
- que le contenu soit publié **une seule fois**, à un seul endroit ;
- une zone `/admin` réservée à l'équipe web, avec les **comptes du domaine** existants.

## Le cahier des charges

**Domaine :** `lab.local` · **Réseau :** `10.0.0.0/24` sur le pont `vmbr2` · passerelle `10.0.0.1` · DNS `10.0.0.10`

| VM | IP | Rôle |
|---|---|---|
| `srv-dc01` | `10.0.0.10` | Contrôleur de domaine (déjà en place) |
| `srv-files01` | `10.0.0.20` | Serveur NFS : pages du site dans `/srv/web` |
| `srv-lb01` | `10.0.0.30` | HAProxy, port 80, page de stats sur 8404 |
| `srv-web01` | `10.0.0.31` | Apache |
| `srv-web02` | `10.0.0.32` | Nginx |

**Règles :**

- Répartition **roundrobin** avec test de santé HTTP ; le serveur qui a répondu doit être identifiable.
- Le partage NFS est en **lecture seule** et réservé aux deux serveurs web.
- `/admin` : accessible aux seuls membres du groupe AD `web-admins`, via **LDAPS** et un compte de service dédié `svc-web`. Aucune requête `/admin` ne doit aboutir sans mot de passe, même via le répartiteur.

---

## Tâche 1 — Préparer les quatre VM

Clone `tpl-debian01` quatre fois et donne à chaque clone son nom, son IP, un nouveau `machine-id` et de nouvelles clés SSH.

??? tip "💡 Indice"
    Démarre les clones **un par un** : tant qu'ils ne sont pas personnalisés, ils ont tous l'IP du template. Vérifie aussi que la carte réseau n'est pas « Disconnect » (symptôme : `NO-CARRIER` dans `ip a`).

??? success "✅ Solution"
    Dans la console Proxmox **de la VM** (l'invite doit afficher le nom de la VM, pas `pve01`) :
    ```bash
    hostnamectl set-hostname srv-web01
    sed -i 's/tpl-debian01/srv-web01/g' /etc/hosts
    nano /etc/network/interfaces        # address 10.0.0.31/24
    rm -f /etc/machine-id /var/lib/dbus/machine-id && systemd-machine-id-setup
    rm -f /etc/ssh/ssh_host_* && ssh-keygen -A
    reboot
    ```
    Détails : 🛠️ [Préparer une VM clonée](tuto-preparer-clone-vm.md).

## Tâche 2 — Deux serveurs web distincts

Installe Apache sur `srv-web01` et Nginx sur `srv-web02`, chacun avec une page d'accueil qui indique son nom.

??? tip "💡 Indice"
    Les deux servent par défaut `/var/www/html/index.html` sur Debian. Teste chacun **en direct** avec `curl` avant de passer au répartiteur.

??? success "✅ Solution"
    ```bash
    # srv-web01
    apt install -y apache2
    echo "<h1>Backend APACHE</h1>" > /var/www/html/index.html
    ```
    ```bash
    # srv-web02
    apt install -y nginx
    echo "<h1>Backend NGINX</h1>" > /var/www/html/index.html
    ```
    ```bash
    curl -s http://10.0.0.31 ; curl -s http://10.0.0.32
    ```

## Tâche 3 — Le répartiteur HAProxy

Configure `srv-lb01` selon le cahier des charges et prouve la répartition 50/50.

??? tip "💡 Indice"
    Il faut un `frontend` sur le port 80, un `backend` en `balance roundrobin` avec `option httpchk`, et un en-tête de réponse contenant le nom du serveur (`%s`) pour compter. Valide toujours avec `haproxy -c` avant de redémarrer.

??? success "✅ Solution"
    À ajouter à `/etc/haproxy/haproxy.cfg` :
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
    ```bash
    haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl restart haproxy
    for i in $(seq 1 100); do curl -sI http://10.0.0.30/ | grep -i x-backend; done | sort | uniq -c
    apt install -y apache2-utils && ab -n 1000 -c 10 http://10.0.0.30/
    ```
    Détails : 🛠️ [Répartir la charge avec HAProxy](tuto-haproxy-repartition-charge.md).

## Tâche 4 — Un seul contenu, servi en NFS

Publie les pages sur `srv-files01` et monte-les sur les deux serveurs web, de façon permanente.

??? tip "💡 Indice"
    Côté serveur : `nfs-kernel-server`, une ligne dans `/etc/exports` par client autorisé, puis `exportfs -ra`. Côté clients : `nfs-common` et une ligne `nfs` dans `/etc/fstab` avec `_netdev`. Pense au propriétaire des fichiers : `www-data` a l'UID 33.

??? success "✅ Solution"
    ```bash
    # srv-files01
    apt install -y nfs-kernel-server
    mkdir -p /srv/web
    echo "<h1>Site servi depuis srv-files01 (NFS)</h1>" > /srv/web/index.html
    chown -R 33:33 /srv/web && chmod -R 755 /srv/web
    echo "/srv/web 10.0.0.31(ro,sync,no_subtree_check) 10.0.0.32(ro,sync,no_subtree_check)" >> /etc/exports
    exportfs -ra && exportfs -v
    ```
    ```bash
    # srv-web01 et srv-web02
    apt install -y nfs-common
    showmount -e 10.0.0.20
    rm -rf /var/www/html/*
    echo "10.0.0.20:/srv/web /var/www/html nfs defaults,_netdev 0 0" >> /etc/fstab
    systemctl daemon-reload && mount -a
    df -h | grep 10.0.0.20
    ```
    Détails : 🛠️ [Partager un dossier en NFS](tuto-partage-nfs.md).

## Tâche 5 — Une zone `/admin` réservée à un groupe AD

Crée le compte de service et le groupe dans l'AD, protège `/admin` sur Apache, puis fais en sorte que le répartiteur ne laisse jamais passer `/admin` sans authentification.

??? tip "💡 Indice"
    - Nginx n'a pas d'authentification LDAP intégrée : seul Apache peut protéger `/admin`. Avec le roundrobin, une requête sur deux irait donc sur une page **non protégée**. Il faut une `acl` HAProxy qui envoie `/admin` vers Apache seul.
    - Le partage NFS est en lecture seule sur les serveurs web : la page `/admin` se crée sur `srv-files01`.
    - Teste LDAPS avec `ldapwhoami` / `ldapsearch` **avant** de toucher Apache.

??? success "✅ Solution"
    ```bash
    # srv-dc01 (Samba AD), en root
    samba-tool user create svc-web '<mot-de-passe-svc-web>'
    samba-tool user setexpiry svc-web --noexpiry
    samba-tool group add web-admins
    samba-tool group addmembers web-admins alice
    ```
    ```bash
    # srv-files01
    mkdir -p /srv/web/admin && echo "<h1>Zone admin</h1>" > /srv/web/admin/index.html
    chown -R 33:33 /srv/web/admin
    ```
    ```bash
    # srv-web01
    apt install -y ldap-utils
    LDAPTLS_REQCERT=never ldapwhoami -H ldaps://10.0.0.10 -D "svc-web@lab.local" -w '<mot-de-passe-svc-web>'
    a2enmod ldap authnz_ldap
    ```
    `/etc/apache2/conf-available/auth-ad.conf` :
    ```apache
    LDAPVerifyServerCert Off

    <Location /admin>
        LDAPReferrals Off
        AuthType Basic
        AuthName "Zone admin"
        AuthBasicProvider ldap
        AuthLDAPURL "ldaps://10.0.0.10:636/DC=lab,DC=local?sAMAccountName?sub?(objectClass=user)"
        AuthLDAPBindDN "svc-web@lab.local"
        AuthLDAPBindPassword "<mot-de-passe-svc-web>"
        AuthLDAPGroupAttribute member
        AuthLDAPGroupAttributeIsDN on
        Require ldap-group CN=web-admins,CN=Users,DC=lab,DC=local
    </Location>
    ```
    ```bash
    chmod 640 /etc/apache2/conf-available/auth-ad.conf
    a2enconf auth-ad && apache2ctl configtest && systemctl restart apache2
    ```
    Sur `srv-lb01`, dans le frontend, avant `default_backend` :
    ```text
        acl is_admin path_beg /admin
        use_backend admin_back if is_admin
    ```
    et à la fin du fichier :
    ```text
    backend admin_back
        option httpchk GET /
        http-response set-header X-Backend %s
        server apache 10.0.0.31:80 check
    ```
    ```bash
    haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl restart haproxy
    ```
    Détails : 🛠️ [Protéger une page web par un groupe AD](tuto-apache-auth-ad.md).

---

## ✅ Grille de validation

Depuis `srv-lb01` :

| # | Test | Résultat attendu |
|---|---|---|
| 1 | `curl -s http://10.0.0.31` et `http://10.0.0.32` | Chaque backend répond |
| 2 | 100 requêtes sur `10.0.0.30`, comptage `x-backend` | 50 apache / 50 nginx |
| 3 | `ab -n 1000 -c 10 http://10.0.0.30/` | `Failed requests: 0` |
| 4 | Page `http://10.0.0.30:8404/stats` | Deux serveurs UP |
| 5 | `systemctl stop nginx` sur `srv-web02` | Le site répond toujours (Apache seul) |
| 6 | Modification de `/srv/web/index.html` sur `srv-files01` | Visible sur 4 requêtes de suite |
| 7 | `/admin` sans identifiants, 4 fois | `401 401 401 401` |
| 8 | `/admin` avec `alice` (membre) | `<h1>Zone admin</h1>`, `x-backend: apache` |
| 9 | `/admin` avec `bruno` (hors groupe) | `401` |

## Si un test échoue

| Test | Piste |
|---|---|
| 1 | Service web arrêté, mauvaise IP, case **Firewall** de la carte réseau dans Proxmox |
| 2 | Comptage vide : l'en-tête arrive en minuscules, utiliser `grep -i` ; un serveur DOWN sur la page de stats |
| 3 | `Failed requests (Length: …)` : les deux pages n'ont pas la même taille ; disparaît après la tâche 4 |
| 6 | Une réponse sur deux est l'ancienne page : un seul backend a monté le partage (`df -h`) ; quelques secondes de retard : cache NFS |
| 7 | `200` : modules LDAP non activés, `auth-ad.conf` non activé, ou `/admin` pas routé vers Apache seul |
| 8 | `Password Mismatch` dans `/var/log/apache2/error.log` : tester l'utilisateur avec `ldapwhoami` ; s'il passe, c'est le mot de passe de `svc-web` dans la config qui est faux |
| Apache ne démarre plus | `LDAPReferrals not allowed here` : la directive doit être **dans** le `<Location>` |

!!! note "Retour d'expérience"
    Les blocages rencontrés en montant ce lab : un PC Windows qui ne joignait pas le réseau interne à cause d'une ancienne carte VMware sur la même plage (résolu par le [rebond SSH](../reseau/tuto-ssh-rebond.md)), `samba-tool` lancé sans être root, `LDAPReferrals` placé hors du `<Location>`, et un mauvais mot de passe de compte de service, trouvé grâce à `LogLevel authnz_ldap:trace1`.

## Sources

Voir les sections « Sources » des tutos liés : [HAProxy](tuto-haproxy-repartition-charge.md#sources), [NFS](tuto-partage-nfs.md#sources), [Apache + AD](tuto-apache-auth-ad.md#sources).
