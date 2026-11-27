---
hide:
  - toc
---

# Système

Windows Server, Active Directory, GPO, Linux, virtualisation, sauvegarde. Suis les thèmes dans l'ordre : chacun commence par **comprendre**, puis **pratiquer**.

<p class="kb-legend" markdown>
:material-book-open-variant: Comprendre &nbsp;·&nbsp;
:material-wrench-outline: Pratiquer &nbsp;·&nbsp;
:material-flask-outline: TP &nbsp;·&nbsp;
:material-clipboard-text-outline: Mémo &nbsp;&nbsp;
<span class="kb-badge kb-bases">Bases</span>
<span class="kb-badge kb-inter">Intermédiaire</span>
</p>

<div class="grid cards" markdown>

-   :material-numeric-1-circle-outline:{ .lg .middle } __Virtualisation__

    ---

    Créer et gérer des machines virtuelles avec Proxmox.

    - :material-book-open-variant: [Virtualisation et hyperviseurs](concept-virtualisation.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md) <span class="kb-badge kb-bases">Bases</span>

-   :material-numeric-2-circle-outline:{ .lg .middle } __Comptes et droits Linux__

    ---

    Gérer utilisateurs, groupes, permissions (chmod, chown) et sudo.

    - :material-book-open-variant: [Comptes, droits et permissions Linux](concept-permissions-linux.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Gérer utilisateurs, groupes et permissions](tuto-permissions-linux.md) <span class="kb-badge kb-bases">Bases</span>

-   :material-numeric-3-circle-outline:{ .lg .middle } __Active Directory__

    ---

    Centraliser comptes, postes et règles avec un domaine Windows Server.

    - :material-book-open-variant: [Domaine, OU et GPO](concept-active-directory.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Monter un AD sur une VM Proxmox](tuto-monter-ad-proxmox.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-wrench-outline: [Protéger une page web par un groupe AD](tuto-apache-auth-ad.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-4-circle-outline:{ .lg .middle } __Conteneurs Docker__

    ---

    Emballer et déployer des services avec Docker et Docker Compose.

    - :material-book-open-variant: [Image, conteneur, volume, réseau](concept-docker.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Déployer un service avec Docker Compose](tuto-docker-compose-proxmox.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-5-circle-outline:{ .lg .middle } __Répartition de charge__

    ---

    Répartir le trafic web entre plusieurs serveurs avec HAProxy.

    - :material-book-open-variant: [La répartition de charge](concept-repartition-charge.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Répartir la charge avec HAProxy](tuto-haproxy-repartition-charge.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-6-circle-outline:{ .lg .middle } __Partage de fichiers NFS__

    ---

    Partager un dossier sur le réseau entre serveurs Linux.

    - :material-book-open-variant: [Le partage de fichiers NFS](concept-nfs.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Partager un dossier en NFS](tuto-partage-nfs.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-flask-outline:{ .lg .middle } __S'entraîner__

    ---

    Mettre tout bout à bout sur un cas réel, avec indices et solutions.

    - :material-flask-outline: [TP : monter l'infra d'une PME sur Proxmox](tp-infra-pme-proxmox.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-flask-outline: [TP : load balancer HAProxy, NFS et accès admin AD](tp-load-balancer-haproxy-nfs.md) <span class="kb-badge kb-inter">Intermédiaire</span>

</div>
