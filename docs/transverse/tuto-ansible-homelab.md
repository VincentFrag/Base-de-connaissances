---
icon: material/wrench-outline
title: Automatiser ses VM avec Ansible
tags: [transverse, tuto, ansible, automatisation, playbook, nginx, proxmox, debian, ssh]
---

# 🛠️ Automatiser ses VM avec Ansible

!!! abstract "Objectif"
    Installer **Ansible** sur un nœud de contrôle du lab, lui donner un accès **SSH par clé** aux VM Debian, écrire un **inventaire**, lancer des **commandes ad hoc**, puis deux **playbooks** : un socle commun à toutes les VM (paquets, fuseau horaire, SSH durci) et un déploiement **nginx** identique sur `srv-web01` et `srv-web02`. Pour finir : prouver l'idempotence et la remise en conformité.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 h

## Prérequis

- Comprendre inventaire, playbook, module et idempotence → 🧠 [Ansible](concept-ansible.md)
- Savoir se connecter en SSH et utiliser une clé → 🧠 [SSH](../reseau/concept-ssh.md)
- Des VM Debian clonées depuis `tpl-debian01`, avec le compte `admin` membre de `sudo` → 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md)
- Un hôte Proxmox `pve01` avec le réseau du lab `10.0.0.0/24` sur le pont `vmbr2`, et un accès Internet pour les paquets

## Le plan qu'on va monter

| Élément | Valeur |
|---|---|
| Nœud de contrôle | `srv-ansible01` · `10.0.0.50` · 1 vCPU, 1 Go RAM · Debian (ou WSL sur ton PC, voir étape 1) |
| Nœuds gérés | `srv-web01` `10.0.0.31` · `srv-web02` `10.0.0.32` (groupe `web`) · `srv-lb01` `10.0.0.30` (groupe `repartiteurs`) |
| Compte | `admin` partout, avec `sudo` |
| Dossier du projet | `~/ansible-lab` sur le nœud de contrôle |
| Résultat | Une page `http://10.0.0.31` et `http://10.0.0.32` déployée par Ansible |

```text
~/ansible-lab/
├── ansible.cfg            # réglages du projet
├── inventory.ini          # les machines et leurs groupes
├── group_vars/
│   └── web.yml            # variables du groupe web
├── templates/
│   └── index.html.j2      # modèle de la page d'accueil
├── socle.yml              # playbook commun à toutes les VM
└── web.yml                # playbook des serveurs web
```

!!! info "Rien à installer sur les VM gérées"
    Les nœuds gérés n'ont besoin que de **SSH** et de **Python 3**, déjà présents sur Debian. Seul le nœud de contrôle reçoit Ansible.

---

## Étape 1 — Préparer le nœud de contrôle

=== "VM srv-ansible01 (recommandé)"

    1. Cloner `tpl-debian01` en **Full Clone**, nom `srv-ansible01`, et le personnaliser (nom, IP `10.0.0.50`, machine-id, clés d'hôte SSH) comme dans 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md).
    2. Se connecter : `ssh admin@10.0.0.50`, puis installer Ansible :

    ```bash
    sudo apt update
    sudo apt install -y ansible
    ansible --version
    ```

    La VM est **dans** le réseau du lab : elle joint directement toutes les autres.

=== "WSL sur ton PC"

    Ansible ne tourne pas nativement sous Windows, mais très bien dans **WSL** (Ubuntu ou Debian) :

    ```bash
    sudo apt update
    sudo apt install -y ansible
    ansible --version
    ```

    Les VM sont derrière `pve01` : ajoute dans `~/.ssh/config` **de WSL** un rebond pour tout le réseau du lab (même principe que 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)) :

    ```text
    Host pve01
        HostName 192.168.1.20
        User root

    Host 10.0.0.*
        User admin
        ProxyJump pve01
    ```

    Ansible utilise le client `ssh` du système : il profite donc automatiquement de ce rebond. Crée le projet dans ton dossier personnel WSL (`~/ansible-lab`), **pas** sous `/mnt/c/` (voir dépannage).

!!! tip "Version"
    `ansible --version` affiche la version d'`ansible-core` installée. Le paquet `ansible` de la distribution suffit pour le lab. Selon la documentation, `ansible-core` seul **ne contient pas** les collections comme `community.general` (utilisée à l'étape 5). Si un module est introuvable, voir le dépannage.

## Étape 2 — Donner l'accès SSH par clé

Ansible se connecte comme toi en SSH : sans clé, il faudrait taper le mot de passe pour chaque machine. Sur le nœud de contrôle :

```bash
ssh-keygen -t ed25519 -C "admin@srv-ansible01"      # Entrée pour accepter le chemin par défaut
ssh-copy-id admin@10.0.0.31
ssh-copy-id admin@10.0.0.32
ssh-copy-id admin@10.0.0.30
```

`ssh-copy-id` demande d'accepter l'empreinte de chaque machine (`yes`) puis le mot de passe d'`admin` une dernière fois. Vérifier qu'aucun mot de passe n'est plus demandé :

```bash
ssh admin@10.0.0.31 hostname     # doit afficher srv-web01, sans mot de passe
```

!!! warning "Une VM sans empreinte connue bloque Ansible"
    Par défaut, Ansible refuse une machine dont l'empreinte SSH n'est pas dans `~/.ssh/known_hosts`. Se connecter une première fois à la main (ou par `ssh-copy-id`) règle la question proprement. Ne désactive pas la vérification (`host_key_checking = False`) ailleurs que dans un lab jetable.

## Étape 3 — Écrire la configuration et l'inventaire

```bash
mkdir -p ~/ansible-lab/{group_vars,templates}
cd ~/ansible-lab
```

`ansible.cfg` : les réglages du projet, lus automatiquement quand on lance Ansible **depuis ce dossier**.

```ini
[defaults]
inventory = inventory.ini
remote_user = admin
interpreter_python = auto_silent
```

`inventory.ini` : les machines, rangées par groupe. `ansible_host` donne l'IP quand le nom n'est pas résolu par le DNS.

```ini
[web]
srv-web01 ansible_host=10.0.0.31
srv-web02 ansible_host=10.0.0.32

[repartiteurs]
srv-lb01 ansible_host=10.0.0.30

# Groupe de groupes : toutes les VM Debian du lab
[debian:children]
web
repartiteurs
```

Vérifier ce qu'Ansible a compris :

```bash
ansible-inventory --graph
```

```text
@all:
  |--@ungrouped:
  |--@debian:
  |  |--@web:
  |  |  |--srv-web01
  |  |  |--srv-web02
  |  |--@repartiteurs:
  |  |  |--srv-lb01
```

!!! tip "Avec le DNS du lab"
    Si `srv-dc01` (`10.0.0.10`) résout les noms (`srv-web01.lab.local`), tu peux écrire directement `srv-web01.lab.local` dans l'inventaire et te passer de `ansible_host`.

## Étape 4 — Premières commandes ad hoc

Une commande ad hoc lance **un module, une fois**, sans playbook. Idéal pour tester ou interroger.

```bash
ansible all -m ping                        # Ansible arrive-t-il à se connecter ?
ansible web -a "uptime"                    # une commande sur le groupe web
ansible srv-web01 -m setup -a "filter=ansible_default_ipv4"   # un fact
ansible debian -m apt -a "name=htop state=present" -b -K      # installer un paquet en root
```

- `-m` choisit le module (`command` par défaut, d'où `-a "uptime"` tout seul) ;
- `-b` (*become*) passe root avec `sudo` ;
- `-K` demande le **mot de passe sudo** d'`admin` (une fois pour toutes les machines).

Résultat attendu du `ping` (ce n'est pas un ping ICMP, c'est un test de connexion SSH + Python) :

```text
srv-web01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `SUCCESS` … `"ping": "pong"` | SSH et Python fonctionnent | Passer à la suite |
| `UNREACHABLE` … `Permission denied (publickey,password)` | La clé n'est pas installée pour `admin` | Refaire `ssh-copy-id admin@<IP>` |
| `UNREACHABLE` … `Connection timed out` | La machine ne répond pas en SSH | VM éteinte, mauvaise IP ou rebond absent (WSL) : 🛠️ [Diagnostiquer une connexion](../reseau/tuto-diagnostic-connexion.md) |
| `UNREACHABLE` … `Host key verification failed` | Empreinte inconnue ou changée (VM recréée) | `ssh-keygen -R 10.0.0.31` puis `ssh admin@10.0.0.31` une fois |
| `FAILED` … `Missing sudo password` | Tâche en root sans mot de passe sudo | Ajouter `-K` |
| `CHANGED` en orange | Le module a modifié la machine | Normal au premier passage |

## Étape 5 — Le playbook « socle » commun à toutes les VM

`socle.yml` : ce que **chaque** VM Debian du lab doit avoir.

```yaml
- name: Socle commun des VM Debian
  hosts: debian
  become: true

  tasks:
    - name: Paquets de base installés
      ansible.builtin.apt:
        name: [curl, vim, htop, tree, chrony]
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Fuseau horaire Europe/Paris
      community.general.timezone:
        name: Europe/Paris

    - name: Connexion SSH directe en root interdite
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin no'
        validate: /usr/sbin/sshd -t -f %s
      notify: Redémarrer SSH

  handlers:
    - name: Redémarrer SSH
      ansible.builtin.service:
        name: ssh
        state: restarted
```

| Élément | Rôle |
|---|---|
| `hosts: debian` | Le groupe ciblé (défini dans l'inventaire) |
| `become: true` | Toutes les tâches passent root avec `sudo` |
| `cache_valid_time: 3600` | Ne refait `apt update` que si le cache a plus d'une heure : le second passage reste `ok` |
| `validate:` | Teste le fichier modifié avec `sshd -t` **avant** de le mettre en place : une erreur ne coupe pas SSH |
| `notify` / `handlers` | SSH n'est redémarré **que si** la ligne a réellement changé |

## Étape 6 — Simuler, lancer, relancer

**1. Vérifier la syntaxe puis simuler** (rien n'est modifié) :

```bash
ansible-playbook socle.yml --syntax-check
ansible-playbook socle.yml --check --diff -K
```

`--diff` affiche, pour chaque fichier, les lignes qui **seraient** changées.

**2. Lancer pour de bon :**

```bash
ansible-playbook socle.yml -K
```

```text
PLAY RECAP *********************************************************************
srv-lb01   : ok=5  changed=4  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
srv-web01  : ok=5  changed=4  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
srv-web02  : ok=5  changed=4  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

**3. Relancer exactement la même commande** : c'est le test d'idempotence.

```text
srv-lb01   : ok=4  changed=0  unreachable=0  failed=0  ...
srv-web01  : ok=4  changed=0  unreachable=0  failed=0  ...
srv-web02  : ok=4  changed=0  unreachable=0  failed=0  ...
```

Au premier passage, `changed=4` = trois tâches + le redémarrage de SSH par le handler. Au second, tout est déjà conforme : aucune modification, et le handler ne se déclenche pas (d'où un `ok` de moins).

| Ce que tu vois dans le récapitulatif | Ce que ça veut dire | Action |
|---|---|---|
| `changed=0` au second passage | Playbook idempotent | ✅ Rien à faire |
| `changed=1` à chaque passage, toujours la même tâche | Tâche non idempotente (souvent `command` ou `shell`) | Remplacer par un module dédié, ou ajouter `creates:` / `changed_when:` |
| `failed=1` | Une tâche a échoué : Ansible **arrête** cette machine, continue les autres | Lire le message rouge au-dessus du récapitulatif |
| `unreachable=1` | Connexion SSH perdue | Voir le tableau de l'étape 4 |
| `skipped=1` | Tâche sautée par une condition `when:` | Normal si voulu |

## Étape 7 — Déployer nginx sur les serveurs web

**Les variables du groupe** `web`, dans `group_vars/web.yml` (Ansible les charge seul, d'après le nom du fichier) :

```yaml
site_titre: "Lab - serveur web"
site_couleur: "#2e7d32"
```

**Le modèle de page** `templates/index.html.j2` (Jinja2 : `{{ … }}` est remplacé par la valeur de la variable) :

```html
<!doctype html>
<html lang="fr">
<head><meta charset="utf-8"><title>{{ site_titre }}</title></head>
<body style="font-family: sans-serif; border-top: 8px solid {{ site_couleur }};">
  <h1>{{ site_titre }}</h1>
  <p>Servi par <strong>{{ inventory_hostname }}</strong> ({{ ansible_default_ipv4.address }})</p>
  <p>Page déployée par Ansible.</p>
</body>
</html>
```

**Le playbook** `web.yml` :

```yaml
- name: Serveurs web nginx
  hosts: web
  become: true

  tasks:
    - name: Nginx installé
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
        cache_valid_time: 3600

    - name: Page d'accueil générée depuis le modèle
      ansible.builtin.template:
        src: templates/index.html.j2
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: "0644"

    - name: En-tête qui indique le serveur
      ansible.builtin.copy:
        dest: /etc/nginx/conf.d/serveur.conf
        content: |
          add_header X-Serveur {{ inventory_hostname }} always;
        mode: "0644"
      notify: Recharger nginx

    - name: Nginx démarré et activé au démarrage
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Recharger nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

```bash
ansible-playbook web.yml -K
curl -I http://10.0.0.31            # doit contenir : X-Serveur: srv-web01
curl -s http://10.0.0.32 | grep Servi
```

Ouvrir `http://10.0.0.31` et `http://10.0.0.32` dans un navigateur : même page, même couleur, seul le nom du serveur change.

!!! tip "Changer une seule valeur, partout"
    Modifie `site_couleur` dans `group_vars/web.yml` et relance `ansible-playbook web.yml -K` : seule la tâche *Page d'accueil* passe en `changed`, sur les deux serveurs à la fois. Avec 2 serveurs c'est pratique ; avec 20, c'est indispensable.

## Étape 8 — Prouver la remise en conformité

Simuler un « bricolage » à la main sur `srv-web01` :

```bash
ssh admin@10.0.0.31 "echo 'modifie a la main' | sudo tee /var/www/html/index.html"
```

Puis demander à Ansible ce qui a dérivé, sans rien toucher :

```bash
ansible-playbook web.yml --check --diff -K
```

Le `--diff` montre la page modifiée sur `srv-web01` seulement, et `srv-web02` reste à `changed=0`. Relancer sans `--check` remet la page d'origine. C'est la règle d'or : **sur une machine gérée par Ansible, on corrige le playbook, pas le serveur.**

---

## ✅ Vérification finale

- [ ] `ansible all -m ping` répond `pong` pour les trois VM
- [ ] `ansible-inventory --graph` montre les groupes `web`, `repartiteurs` et `debian`
- [ ] `ansible-playbook socle.yml -K` lancé deux fois : `changed=0` au second passage
- [ ] `ansible debian -a "timedatectl show -p Timezone"` affiche `Timezone=Europe/Paris` partout
- [ ] `http://10.0.0.31` et `http://10.0.0.32` affichent la page avec le bon nom de serveur
- [ ] Après une modification à la main, `--check --diff` la détecte et un nouveau passage la corrige

## 🔧 Dépannage courant

!!! question "Sous WSL, Ansible ignore `ansible.cfg` et ne trouve plus l'inventaire ?"
    Un projet placé dans `/mnt/c/…` hérite des droits Windows : le dossier est accessible en écriture à tous. Par sécurité, Ansible ne charge **pas** un `ansible.cfg` situé dans un tel dossier. Déplace le projet dans ton dossier WSL : `mv /mnt/c/…/ansible-lab ~/`.

!!! question "`[WARNING]: provided hosts list is empty` ou `Could not match supplied host pattern` ?"
    Ansible ne trouve pas l'inventaire ou le groupe. Vérifier que tu lances la commande **depuis `~/ansible-lab`** (là où est `ansible.cfg`), l'orthographe du groupe, puis `ansible-inventory --graph`.

!!! question "`mapping values are not allowed here` ou `did not find expected key` ?"
    Erreur d'indentation YAML : un espace de trop ou de moins, ou une tabulation. La ligne indiquée est souvent celle **d'après** l'erreur. `ansible-playbook <fichier> --syntax-check` la localise ; un éditeur qui affiche les espaces (VS Code) aide beaucoup.

!!! question "`couldn't resolve module/action 'community.general.timezone'` ?"
    `ansible-core` a été installé seul (par exemple avec `pip install ansible-core`) : il n'a pas les collections. Installer le paquet complet (`sudo apt install ansible`), ou seulement la collection : `ansible-galaxy collection install community.general`.

!!! question "Une tâche reste en `changed` à chaque passage ?"
    Elle n'est pas idempotente. Avec `command` ou `shell`, indiquer quand elle a déjà été faite : `creates: /chemin/du/fichier/produit`, ou `changed_when: false` pour une commande qui ne fait que lire.

!!! question "Le playbook bloque sans rien afficher ?"
    Une tâche attend une saisie (mot de passe sudo sans `-K`, question d'un paquet). Interrompre avec ++ctrl+c++, relancer avec `-K` et, pour voir le détail de la connexion, avec `-vvv`.

## 📋 Mémo des commandes

| Besoin | Commande |
|---|---|
| Voir les groupes de l'inventaire | `ansible-inventory --graph` |
| Tester la connexion | `ansible all -m ping` |
| Commande ponctuelle sur un groupe | `ansible web -a "df -h"` |
| Module ponctuel en root | `ansible web -m apt -a "name=htop state=present" -b -K` |
| Lister les facts d'une machine | `ansible srv-web01 -m setup` |
| Vérifier la syntaxe d'un playbook | `ansible-playbook web.yml --syntax-check` |
| Simuler et voir les différences | `ansible-playbook web.yml --check --diff -K` |
| Lancer un playbook | `ansible-playbook web.yml -K` |
| Sur une seule machine | `ansible-playbook web.yml -K --limit srv-web01` |
| Lister les tâches sans les lancer | `ansible-playbook web.yml --list-tasks` |
| Plus de détails en cas d'erreur | `ansible-playbook web.yml -K -vvv` |
| Chiffrer un fichier de variables secrètes | `ansible-vault encrypt group_vars/web_secrets.yml` |
| Lancer avec des variables chiffrées | `ansible-playbook web.yml -K --ask-vault-pass` |
| Documentation d'un module | `ansible-doc ansible.builtin.template` |

## Voir aussi

- 🧠 [Ansible : inventaire, playbook, idempotence](concept-ansible.md)
- 🛠️ [Déployer une infrastructure Azure avec Terraform](../cloud/tuto-terraform-azure.md) : créer la VM avec Terraform, puis la configurer avec ce playbook
- 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md) : piloter le lab depuis le PC
- 🛠️ [Automatiser une tâche avec cron et le Planificateur de tâches](tuto-taches-planifiees.md) : relancer un playbook chaque nuit pour garder les VM conformes
- 🧠 [Supervision](concept-supervision.md) : vérifier ensuite que les services déployés répondent

??? quote "Sources"
    Documentation officielle d'Ansible, consultée le 27 novembre 2026 :

    - [Commandes ad hoc](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html) : module `command` par défaut, `-K`, 5 processus en parallèle
    - [Connexion et vérification des clés d'hôte](https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html)
    - [Handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
    - [Configuration (`ansible.cfg`, `interpreter_python`, dossier accessible à tous)](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
    - Modules : [apt](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html) · [lineinfile](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html) · [community.general.timezone](https://docs.ansible.com/ansible/latest/collections/community/general/timezone_module.html)
