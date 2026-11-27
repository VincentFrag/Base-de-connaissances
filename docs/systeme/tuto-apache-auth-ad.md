---
icon: material/wrench-outline
title: Protéger une page web par un groupe Active Directory (Apache + LDAPS)
tags: [système, tuto, active directory, samba, apache, ldap, ldaps, haproxy, authentification]
---

# 🛠️ Protéger une page web par un groupe Active Directory (Apache + LDAPS)

!!! abstract "Objectif"
    Réserver la zone `/admin` d'un site aux **membres d'un groupe AD**. Apache vérifie l'identifiant et le mot de passe auprès du contrôleur de domaine en **LDAPS**, puis contrôle l'appartenance au groupe. Derrière un répartiteur HAProxy, on fait en sorte que `/admin` passe **toujours** par le serveur protégé.

**Niveau :** 🟡 Intermédiaire · **Durée :** 45 min

## Prérequis

- Comprendre domaine, utilisateurs et groupes → 🧠 [Active Directory](concept-active-directory.md)
- Un contrôleur de domaine `lab.local` (Samba AD ou Windows Server) → 🛠️ [Monter un AD sur une VM Proxmox](tuto-monter-ad-proxmox.md)
- Le répartiteur et les serveurs web en place → 🛠️ [Répartir la charge avec HAProxy](tuto-haproxy-repartition-charge.md) et 🛠️ [Pages servies en NFS](tuto-partage-nfs.md)

## Le lab

| Machine | IP | Rôle ici |
|---|---|---|
| `srv-dc01` | `10.0.0.10` | Contrôleur de domaine `lab.local`, LDAPS sur le port 636 |
| `srv-files01` | `10.0.0.20` | Partage NFS des pages (`/srv/web`) |
| `srv-lb01` | `10.0.0.30` | HAProxy |
| `srv-web01` | `10.0.0.31` | **Apache** : c'est lui qui protège `/admin` |
| `srv-web02` | `10.0.0.32` | Nginx (pas de module LDAP intégré) |

| Compte / groupe AD | Rôle |
|---|---|
| `svc-web` | **Compte de service** : Apache s'en sert pour chercher les utilisateurs dans l'annuaire |
| `web-admins` | Groupe autorisé (`CN=web-admins,CN=Users,DC=lab,DC=local`) |
| `alice` | Membre de `web-admins` : doit passer |
| `bruno` | Hors du groupe : doit être refusé |

!!! warning "Le piège du roundrobin"
    Nginx n'a pas d'authentification LDAP intégrée : seul Apache protège `/admin`. Si HAProxy alterne entre les deux, une requête sur deux arrive sur Nginx… qui sert la page **sans mot de passe**. L'étape 5 règle ça en envoyant `/admin` vers Apache seul.

---

## Étape 1 — Créer le compte de service et le groupe

=== "Samba AD (Linux)"
    Sur `srv-dc01`, **en root** (sinon `Permission denied` sur `sam.ldb`) :
    ```bash
    samba-tool user create svc-web '<mot-de-passe-svc-web>'
    samba-tool user setexpiry svc-web --noexpiry
    samba-tool group add web-admins
    samba-tool group addmembers web-admins alice
    samba-tool group show web-admins | grep -E "^dn|^member"
    ```

=== "Windows Server (PowerShell)"
    ```powershell
    New-ADUser -Name svc-web -SamAccountName svc-web -AccountPassword (Read-Host -AsSecureString) `
        -Enabled $true -PasswordNeverExpires $true -Path "CN=Users,DC=lab,DC=local"
    New-ADGroup -Name web-admins -GroupScope Global -Path "CN=Users,DC=lab,DC=local"
    Add-ADGroupMember -Identity web-admins -Members alice
    Get-ADGroup web-admins | Select DistinguishedName
    ```

Note le **DN exact** du groupe affiché : c'est lui qu'on écrira dans Apache.

!!! tip "`samba-tool` ne marche que sur le contrôleur de domaine"
    Lancé sur un serveur web, il répond `could not open file /var/lib/samba/private/sam.ldb: No such file or directory` : la base de l'annuaire n'existe que sur le DC.

## Étape 2 — Tester LDAPS depuis le serveur web, sans Apache

Sur `srv-web01` :

```bash
apt install -y ldap-utils

# Le compte de service peut-il se connecter ?
LDAPTLS_REQCERT=never ldapwhoami -H ldaps://10.0.0.10 -D "svc-web@lab.local" -w '<mot-de-passe-svc-web>'

# Trouve-t-il alice et son groupe ?
LDAPTLS_REQCERT=never ldapsearch -H ldaps://10.0.0.10 -D "svc-web@lab.local" -w '<mot-de-passe-svc-web>' \
    -b "DC=lab,DC=local" "(sAMAccountName=alice)" memberOf
```

| Ce que tu vois | Ce que ça veut dire |
|---|---|
| `u:LAB\svc-web` | Le compte de service et LDAPS fonctionnent |
| `memberOf: CN=web-admins,CN=Users,DC=lab,DC=local` | alice est bien dans le groupe |
| `Invalid credentials (49)` | Mauvais mot de passe ou mauvais compte |
| `Can't contact LDAP server (-1)` | Port 636 injoignable ou LDAPS non actif sur le DC |

!!! info "Pourquoi LDAPS et pas LDAP simple ?"
    Le mot de passe de l'utilisateur est transmis au DC pour vérification. Sans chiffrement, il passerait en clair sur le réseau, et un DC Samba **refuse** d'ailleurs par défaut ce type de connexion non chiffrée (`ldap server require strong auth`). `LDAPTLS_REQCERT=never` sert seulement au test : le certificat généré par Samba est auto-signé.

**Si ce test échoue, Apache échouera aussi** : on corrige ici d'abord.

## Étape 3 — Créer la page protégée

Les pages sont sur le partage NFS, monté **en lecture seule** sur les serveurs web : la page se crée donc sur `srv-files01`.

```bash
mkdir -p /srv/web/admin
echo "<h1>Zone admin</h1>" > /srv/web/admin/index.html
chown -R 33:33 /srv/web/admin && chmod -R 755 /srv/web/admin
```

## Étape 4 — Configurer Apache

Sur `srv-web01` :

```bash
a2enmod ldap authnz_ldap
nano /etc/apache2/conf-available/auth-ad.conf
```

```apache
# Hors <Location> : directive globale (certificat auto-signé du DC)
LDAPVerifyServerCert Off

<Location /admin>
    # Directive « par dossier » : elle doit être ICI, pas au-dessus
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

| Directive | Rôle |
|---|---|
| `AuthLDAPURL` | Où chercher : serveur, base de recherche, attribut d'identifiant (`sAMAccountName`), filtre |
| `AuthLDAPBindDN` / `AuthLDAPBindPassword` | Le compte de service utilisé pour la recherche |
| `AuthLDAPGroupAttribute member` + `IsDN on` | Dans AD, un groupe liste ses membres par leur DN dans l'attribut `member` |
| `Require ldap-group` | Seuls les membres de ce groupe passent |
| `LDAPReferrals Off` | Évite qu'Apache suive des renvois d'annuaire (recherche depuis la racine du domaine) |
| `LDAPVerifyServerCert Off` | Accepte le certificat auto-signé : **acceptable en lab seulement** |

```bash
chmod 640 /etc/apache2/conf-available/auth-ad.conf   # le fichier contient un mot de passe
a2enconf auth-ad
apache2ctl configtest && systemctl restart apache2
```

!!! tip "Toujours `apache2ctl configtest` avant le redémarrage"
    Il affiche l'erreur **en entier**, alors que `systemctl status` coupe les lignes. Et tant que tu n'as pas redémarré, le site reste en ligne.

Test **en direct** sur Apache :

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://10.0.0.31/admin/                       # 401
curl -s -u alice:'<mdp-alice>' http://10.0.0.31/admin/                                 # <h1>Zone admin</h1>
curl -s -o /dev/null -w "%{http_code}\n" -u bruno:'<mdp-bruno>' http://10.0.0.31/admin/ # 401
```

## Étape 5 — Envoyer `/admin` vers Apache seul dans HAProxy

Sur `srv-lb01`, dans `/etc/haproxy/haproxy.cfg`, complète le frontend :

```text
frontend http_front
    bind *:80
    acl is_admin path_beg /admin
    use_backend admin_back if is_admin
    default_backend web_back
```

et ajoute un backend qui ne contient qu'Apache :

```text
backend admin_back
    option httpchk GET /
    http-response set-header X-Backend %s
    server apache 10.0.0.31:80 check
```

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl restart haproxy
```

| Ligne | Rôle |
|---|---|
| `acl is_admin path_beg /admin` | Vrai si l'URL commence par `/admin` |
| `use_backend admin_back if is_admin` | Dans ce cas, utiliser le backend Apache seul |
| `default_backend web_back` | Tout le reste continue en roundrobin |

---

## ✅ Vérification

Depuis `srv-lb01` :

```bash
# 1. Sans mot de passe : 4 fois 401, jamais 200
for i in 1 2 3 4; do curl -s -o /dev/null -w "%{http_code} " http://10.0.0.30/admin/; done; echo

# 2. Membre du groupe : la page
curl -s -u alice:'<mdp-alice>' http://10.0.0.30/admin/

# 3. /admin passe toujours par Apache
curl -sI -u alice:'<mdp-alice>' http://10.0.0.30/admin/ | grep -i x-backend

# 4. Le reste du site est toujours réparti
for i in $(seq 1 20); do curl -sI http://10.0.0.30/ | grep -i x-backend; done | sort | uniq -c
```

| Test | Résultat attendu |
|---|---|
| `/admin` sans identifiants, ×4 | `401 401 401 401` |
| `/admin` avec alice | `<h1>Zone admin</h1>` |
| `/admin` avec bruno | `401` |
| En-tête sur `/admin` | `x-backend: apache` |
| Le site `/` | ~10 apache / ~10 nginx |

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `200` sans mot de passe sur `/admin` | Modules ou config non actifs | `apache2ctl -M \| grep ldap` ; `ls /etc/apache2/conf-enabled/` |
| Apache ne démarre plus : `LDAPReferrals not allowed here` | Directive placée hors du `<Location>` | La déplacer **dans** le bloc |
| `500 Internal Server Error` | Apache ne joint pas le DC en LDAPS | `tail /var/log/apache2/error.log` ; refaire l'étape 2 |
| `401` avec le bon mot de passe, log `Password Mismatch` | Le bind du compte de service **ou** de l'utilisateur échoue | Voir le dépannage ci-dessous |
| `401` pour un membre du groupe, bind OK | Mauvais DN de groupe dans `Require ldap-group` | Comparer avec le DN de l'étape 1 |
| Une requête `/admin` sur deux répond `200` via le LB | `/admin` n'est pas routé vers Apache seul | Revoir l'étape 5 |
| Le comptage `x-backend` est vide | HAProxy envoie l'en-tête en **minuscules** | `grep -i` |

## 🔧 Dépannage courant

!!! question "`Password Mismatch` alors que le mot de passe de l'utilisateur est bon ?"
    Ce message d'Apache est générique : il ne dit pas **quel** bind a échoué. Vérifie d'abord l'utilisateur seul :
    ```bash
    LDAPTLS_REQCERT=never ldapwhoami -H ldaps://10.0.0.10 -D "alice@lab.local" -w '<mdp-alice>'
    ```
    Si ça répond `u:LAB\alice`, le problème vient du **compte de service** : le mot de passe écrit dans `AuthLDAPBindPassword` ne correspond pas à celui de `svc-web`. Pour en avoir le cœur net, ajoute dans le `<Location>` :
    ```apache
    LogLevel authnz_ldap:trace1 ldap:debug
    ```
    puis `grep -i ldap /var/log/apache2/error.log | tail` : une ligne `ldap_simple_bind() failed [Invalid credentials]` juste après la construction du filtre de recherche confirme un mauvais mot de passe de bind. Retire ce `LogLevel` une fois réglé.

!!! question "`samba-tool` répond `Permission denied` sur `sam.ldb` ?"
    Tu n'es pas root. `su -` puis relance.

!!! question "Et Nginx, il ne peut vraiment pas faire la même chose ?"
    Pas avec les modules livrés par défaut. Il existe un module tiers (`nginx-auth-ldap`) ou la délégation `auth_request` vers un petit service d'authentification, mais c'est plus lourd : router `/admin` vers Apache reste la solution la plus simple.

!!! warning "En production"
    - Servir le site en **HTTPS** : en HTTP, l'authentification *Basic* envoie l'identifiant et le mot de passe quasiment en clair entre le navigateur et le serveur.
    - Importer le certificat de l'autorité du DC sur le serveur web et remplacer `LDAPVerifyServerCert Off` par une vérification réelle (`LDAPTrustedGlobalCert`).
    - Donner au compte de service **le minimum de droits** (lecture de l'annuaire) et un mot de passe long.

## Pour aller plus loin

- 🧠 [Active Directory](concept-active-directory.md)
- 🧪 [TP : Load balancer HAProxy et pages web sur NFS](tp-load-balancer-haproxy-nfs.md)

## Sources

- [Apache : mod_authnz_ldap](https://httpd.apache.org/docs/2.4/mod/mod_authnz_ldap.html) : `AuthLDAPURL`, `AuthLDAPBindDN`, `AuthLDAPGroupAttribute`, `Require ldap-group`
- [Apache : mod_ldap](https://httpd.apache.org/docs/2.4/mod/mod_ldap.html) : `LDAPVerifyServerCert`, `LDAPReferrals` (contexte « directory »), `LDAPTrustedGlobalCert`
- [Apache : LogLevel par module](https://httpd.apache.org/docs/2.4/mod/core.html#loglevel)
- [Samba : page de manuel `samba-tool(8)`](https://www.samba.org/samba/docs/current/man-html/samba-tool.8.html)
- [Samba : paramètre `ldap server require strong auth` (`smb.conf(5)`)](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html)
- [Microsoft : New-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/new-aduser) et [Add-ADGroupMember](https://learn.microsoft.com/powershell/module/activedirectory/add-adgroupmember)
- [HAProxy : manuel de configuration](https://docs.haproxy.org/) : `acl`, `path_beg`, `use_backend`
- [OpenLDAP : `ldapwhoami(1)`](https://www.openldap.org/software/man.cgi?query=ldapwhoami) et [`ldap.conf(5)`](https://www.openldap.org/software/man.cgi?query=ldap.conf) (`TLS_REQCERT`)
