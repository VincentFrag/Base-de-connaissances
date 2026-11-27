---
icon: material/book-open-variant
title: Ansible (inventaire, playbook, idempotence)
tags: [transverse, bases, concept, ansible, automatisation, iac, yaml, ssh]
---

# 🧠 Ansible : inventaire, playbook, idempotence

!!! abstract "En une phrase"
    **Ansible** configure des dizaines de machines d'un coup, **par SSH et sans rien installer dessus** : on décrit dans un fichier YAML **l'état voulu** (« nginx installé et démarré »), et Ansible ne touche que ce qui n'est pas encore conforme.

## L'analogie

Une **check-list de chef de chantier** plutôt qu'une liste d'ordres :

- le **script Bash** dit « coule une dalle » : relancé deux fois, il coule deux dalles ;
- le **playbook Ansible** dit « il doit y avoir une dalle » : le chef passe, voit qu'elle existe, coche et passe à la suite ;
- l'**inventaire** = la liste des chantiers et des équipes (quels serveurs, rangés par groupe) ;
- les **modules** = les corps de métier spécialisés (plombier `apt`, électricien `service`, peintre `template`) ;
- le **nœud de contrôle** = le bureau du chef, le seul endroit où Ansible est installé.

---

## Les briques à connaître

| Brique | Ce que c'est | Exemple |
|---|---|---|
| **Nœud de contrôle** | La machine où Ansible est installé et d'où on lance les commandes | `srv-ansible01`, ou WSL sur ton PC |
| **Nœuds gérés** | Les machines configurées. Il leur faut seulement **SSH** et **Python** | `srv-web01`, `srv-web02` |
| **Inventaire** | Fichier qui liste les machines et les range en **groupes** | `[web]` → `srv-web01`, `srv-web02` |
| **Module** | Une action prête à l'emploi, qui sait vérifier avant d'agir | `apt`, `service`, `copy`, `template`, `user` |
| **Tâche** (*task*) | Un appel à un module avec ses paramètres | « installer nginx » |
| **Playbook** | Fichier YAML qui associe des **groupes** à une liste de **tâches** | `web.yml` |
| **Handler** | Tâche lancée **seulement si** une autre a changé quelque chose | redémarrer nginx si sa config a changé |
| **Variables** | Valeurs par groupe ou par machine, réutilisées dans les tâches | `group_vars/web.yml` |
| **Template** | Fichier modèle (Jinja2) où Ansible remplace les variables | `index.html.j2` avec `{{ inventory_hostname }}` |
| **Facts** | Infos collectées sur chaque machine au début (OS, IP, RAM) | `ansible_default_ipv4.address` |
| **Rôle** | Dossier standard qui regroupe tâches, handlers, templates d'un service | rôle `nginx` réutilisable d'un projet à l'autre |
| **Commande ad hoc** | Un module lancé une fois, sans playbook | `ansible web -m ping` |

### Comment ça se déroule

1. Tu lances `ansible-playbook web.yml` depuis le nœud de contrôle.
2. Ansible lit l'inventaire et ouvre une connexion **SSH** vers chaque machine du groupe, **en parallèle** (5 à la fois par défaut).
3. Il collecte les **facts**, puis exécute les tâches **dans l'ordre**, toutes les machines avançant ensemble.
4. Chaque module compare l'état réel à l'état voulu et répond `ok` (déjà conforme), `changed` (corrigé) ou `failed`.
5. À la fin, le **récapitulatif** (*PLAY RECAP*) donne le compte par machine. Les handlers notifiés s'exécutent juste avant.

### L'idempotence, le cœur d'Ansible

| | Script Bash | Playbook Ansible |
|---|---|---|
| Style | **Impératif** : « fais ceci, puis cela » | **Déclaratif** : « voici l'état voulu » |
| Relancé une 2ᵉ fois | Peut dupliquer une ligne, planter sur « existe déjà » | Ne change rien : `changed=0` |
| Machine à moitié configurée | À toi de gérer chaque cas avec des `if` | Ansible ne complète que ce qui manque |
| Simulation | Rien de prévu | `--check --diff` montre ce qui **changerait** |

!!! tip "Le test qui ne ment pas"
    Un playbook bien écrit, lancé **deux fois de suite**, doit finir la seconde fois sur `changed=0` partout. Sinon, une tâche n'est pas idempotente (souvent un `command` ou un `shell`).

### Ansible, Terraform, script ou Docker ?

| Outil | Son travail | Exemple dans le lab |
|---|---|---|
| **Script Bash / PowerShell** | Une suite de commandes sur **une** machine | Sauvegarde planifiée sur `srv-files01` |
| **Ansible** | **Configurer l'intérieur** de machines existantes : paquets, fichiers, services, comptes | Installer nginx sur `srv-web01` et `srv-web02` |
| **Terraform** | **Créer l'infrastructure** elle-même : VM, réseaux, règles de pare-feu | Créer les VM dans Azure ou Proxmox |
| **Docker Compose** | Lancer des applications emballées **sur une** machine Docker | La pile WordPress de `srv-docker01` |

Ils se complètent : **Terraform crée** les VM, **Ansible les configure**, et Ansible peut même installer Docker et déposer le `compose.yaml`.

!!! info "Et Windows ?"
    Ansible gère aussi Windows (`srv-dc01`), avec les modules `ansible.windows.*` (`win_feature`, `win_service`…). La connexion passe par **WinRM** ou par OpenSSH pour Windows ; le nœud de contrôle, lui, reste toujours un Linux (ou WSL).

---

## À quoi ça sert concrètement

- **Préparer toutes les VM d'un coup** après un clone : paquets de base, fuseau horaire, compte `admin`, durcissement SSH.
- **Déployer un service identique** sur plusieurs serveurs (deux serveurs web derrière `srv-lb01`) sans oublier une étape sur l'un des deux.
- **Remettre en conformité** : si quelqu'un a modifié une config à la main, relancer le playbook la remet comme prévu.
- **Documenter l'infra** : le playbook versionné dans Git *est* la procédure d'installation, toujours à jour.
- **Reconstruire après une panne** : une VM neuve + le playbook = le serveur d'avant en quelques minutes.

!!! warning "Pièges fréquents"
    - **L'indentation YAML** : deux espaces, jamais de tabulation. Un décalage d'un espace et le playbook ne se lit plus (`mapping values are not allowed here`).
    - **Abuser de `command` et `shell`** : ils relancent la commande à chaque passage et affichent toujours `changed`. Chercher d'abord le module dédié (`apt`, `user`, `lineinfile`…).
    - **Oublier `become: true`** : sans lui, les tâches tournent avec le compte SSH et échouent sur `Permission denied` dès qu'il faut être root.
    - **Mettre des mots de passe en clair** dans les variables versionnées : les chiffrer avec `ansible-vault`.
    - **Lancer sur tout l'inventaire par erreur** : tester d'abord avec `--check --diff` et limiter avec `--limit srv-web01`.
    - **Corriger à la main un serveur géré par Ansible** : la modification sera écrasée au prochain passage. On corrige le playbook, puis on le relance.

## Sources

- [Documentation officielle d'Ansible](https://docs.ansible.com/ansible/latest/) : [commandes ad hoc](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html), [handlers](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html), [connexion SSH](https://docs.ansible.com/ansible/latest/inventory_guide/connection_details.html) (consultée le 27 novembre 2026)

## Pour aller plus loin

- 🛠️ **[Automatiser ses VM avec Ansible](tuto-ansible-homelab.md)** : la mise en pratique (inventaire, commandes ad hoc, playbook de base et serveurs web nginx sur le lab Proxmox)
- 🧪 [TP Azure : Automatisation (TP17)](https://vincentfrag.github.io/knowledge-base/cloud/tp-azure-automatisation/) : Ansible après Terraform, sur une VM Azure (partie C)
- 🧠 [SSH : se connecter à distance](../reseau/concept-ssh.md) : le transport utilisé par Ansible
- 📋 [Mémo scripting Bash & PowerShell](memo-bash-powershell.md) : quand un simple script suffit
- 🧠 [Conteneurs Docker](../systeme/concept-docker.md) : l'autre façon de rendre un déploiement reproductible
