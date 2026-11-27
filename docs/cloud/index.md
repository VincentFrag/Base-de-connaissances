---
hide:
  - toc
---

# Cloud Azure

Un parcours complet de cloud computing sur Azure, faisable seul avec un compte **Azure for Students** : 7 exercices papier, 19 TP et un projet final, issus du cours « Design infrastructures virtuelles & Cloud ». Compte environ 20 à 30 $ de crédit si tu supprimes tout en fin de séance.

!!! warning "Avant le premier TP"
    Lis [Démarrer le parcours Azure](memo-parcours-azure.md) et fais le **TP1 (budget et alertes)** en premier : c'est ton filet de sécurité contre une ressource oubliée qui viderait le crédit.

<p class="kb-legend" markdown>
:material-book-open-variant: Comprendre &nbsp;·&nbsp;
:material-wrench-outline: Pratiquer &nbsp;·&nbsp;
:material-flask-outline: TP &nbsp;·&nbsp;
:material-clipboard-text-outline: Mémo &nbsp;&nbsp;
<span class="kb-badge kb-bases">Bases</span>
<span class="kb-badge kb-inter">Intermédiaire</span>
</p>

<div class="grid cards" markdown>

-   :material-numeric-1-circle-outline:{ .lg .middle } __Démarrer__

    ---

    Les règles pour ne pas griller son crédit, les outils à installer, et sept exercices papier pour poser les notions sans dépenser un centime.

    - :material-clipboard-text-outline: [Démarrer le parcours Azure](memo-parcours-azure.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-flask-outline: [E1 à E7 : penser cloud avant de cliquer](tp-exercices-papier.md) <span class="kb-badge kb-bases">Bases</span>

-   :material-numeric-2-circle-outline:{ .lg .middle } __Fondations__

    ---

    Budget et tags FinOps, première VM avec cloud-init, VNet, sous-réseaux et NSG.

    - :material-flask-outline: [TP1 à TP3 : budget, première VM, VNet et NSG](tp-azure-fondations.md) <span class="kb-badge kb-bases">Bases</span>

-   :material-numeric-3-circle-outline:{ .lg .middle } __Réseau avancé__

    ---

    Hub & spoke avec une VM passerelle, puis un Load Balancer sur deux zones.

    - :material-flask-outline: [TP4 et TP5 : hub & spoke, Load Balancer](tp-azure-reseau-avance.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-4-circle-outline:{ .lg .middle } __Identité et conteneurs__

    ---

    RBAC et identité managée, Keycloak sur VM Docker puis en Container Instance.

    - :material-flask-outline: [TP6 et TP7 : RBAC, Keycloak](tp-azure-identite-conteneurs.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-5-circle-outline:{ .lg .middle } __Accès sécurisé__

    ---

    Atteindre une VM privée : Azure Bastion et Teleport, WireGuard, VPN Gateway.

    - :material-flask-outline: [TP8 à TP10 : Bastion, WireGuard, VPN Gateway](tp-azure-acces-securise.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-6-circle-outline:{ .lg .middle } __Stockage et API__

    ---

    Compte de stockage, SAS, immuabilité et accès privé, puis piloter le cloud par son API (Postman, AWS CLI et S3).

    - :material-flask-outline: [TP11 et TP12 : stockage, SAS, Private Endpoint](tp-azure-stockage.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-flask-outline: [TP13 et TP14 : API REST Azure, AWS CLI et S3](tp-azure-api.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-7-circle-outline:{ .lg .middle } __Exploitation__

    ---

    Supervision Zabbix et Azure Monitor, Azure Backup et restauration.

    - :material-flask-outline: [TP15 et TP16 : supervision, Azure Backup](tp-azure-exploitation.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-8-circle-outline:{ .lg .middle } __Automatisation__

    ---

    La même infrastructure par script CLI, Terraform et Ansible, puis l'autoscaling avec un VMSS.

    - :material-flask-outline: [TP17 : script, Terraform, Ansible, VMSS](tp-azure-automatisation.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-9-circle-outline:{ .lg .middle } __S'entraîner__

    ---

    Les hands-on du cours, puis le projet final : l'architecture de référence en Terraform, documentée et détruite.

    - :material-flask-outline: [TP18 et TP19 : AD DS sur Azure, durcissement](tp-azure-hands-on.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-flask-outline: [Projet final : l'architecture de référence](tp-azure-projet-final.md) <span class="kb-badge kb-inter">Intermédiaire</span>

</div>
