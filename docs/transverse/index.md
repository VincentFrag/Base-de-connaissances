---
hide:
  - toc
---

# Transverse

Ce qui concerne toute l'infrastructure, quel que soit le système ou le réseau : sauvegarde, sécurité, supervision, scripting (PowerShell, Bash), cloud et Azure.

<p class="kb-legend" markdown>
:material-book-open-variant: Comprendre &nbsp;·&nbsp;
:material-wrench-outline: Pratiquer &nbsp;·&nbsp;
:material-flask-outline: TP &nbsp;·&nbsp;
:material-clipboard-text-outline: Mémo &nbsp;&nbsp;
<span class="kb-badge kb-bases">Bases</span>
<span class="kb-badge kb-inter">Intermédiaire</span>
</p>

<div class="grid cards" markdown>

-   :material-numeric-1-circle-outline:{ .lg .middle } __Sauvegarde__

    ---

    Ne jamais perdre une VM ni des données : la règle 3-2-1 et sa mise en place.

    - :material-book-open-variant: [Comprendre les sauvegardes et la règle 3-2-1](concept-sauvegardes.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Sauvegarder et restaurer ses VM avec Proxmox](tuto-proxmox-backup.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-2-circle-outline:{ .lg .middle } __Supervision__

    ---

    Être prévenu d'une panne avant les utilisateurs : métriques, logs et alertes.

    - :material-book-open-variant: [Comprendre la supervision](concept-supervision.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Superviser ses VM avec Uptime Kuma](tuto-uptime-kuma.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Superviser ses VM avec Zabbix](tuto-zabbix.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-wrench-outline: [Superviser ses VM avec Centreon](tuto-centreon.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-3-circle-outline:{ .lg .middle } __Scripting et automatisation__

    ---

    Écrire de petits scripts d'admin, les faire tourner tout seuls, puis configurer toutes les VM d'un coup avec Ansible.

    - :material-clipboard-text-outline: [Mémo scripting Bash & PowerShell](memo-bash-powershell.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Automatiser une tâche avec cron et le Planificateur de tâches](tuto-taches-planifiees.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-book-open-variant: [Comprendre Ansible : inventaire, playbook, idempotence](concept-ansible.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Automatiser ses VM avec Ansible](tuto-ansible-homelab.md) <span class="kb-badge kb-inter">Intermédiaire</span>

</div>
