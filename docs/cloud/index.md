---
hide:
  - toc
---

# Cloud computing

Louer serveurs, réseau et stockage à la demande au lieu de les héberger soi-même. Les notions générales d'abord, puis leur mise en pratique sur **Microsoft Azure** (avec un compte Azure for Students, sans carte bancaire).

!!! warning "Avant de créer quoi que ce soit"
    Commence par [Démarrer sur Azure sans exploser son crédit](tuto-demarrer-azure.md) : budget, tags et réflexes de nettoyage. Une ressource oubliée peut vider le crédit en quelques jours.

<p class="kb-legend" markdown>
:material-book-open-variant: Comprendre &nbsp;·&nbsp;
:material-wrench-outline: Pratiquer &nbsp;·&nbsp;
:material-flask-outline: TP &nbsp;·&nbsp;
:material-clipboard-text-outline: Mémo &nbsp;&nbsp;
<span class="kb-badge kb-bases">Bases</span>
<span class="kb-badge kb-inter">Intermédiaire</span>
</p>

<div class="grid cards" markdown>

-   :material-numeric-1-circle-outline:{ .lg .middle } __Fondamentaux du cloud__

    ---

    IaaS, PaaS, SaaS, régions et zones, SLA, modèle de coût, puis un compte Azure bien protégé.

    - :material-book-open-variant: [Comprendre le cloud computing](concept-cloud-computing.md) <span class="kb-badge kb-bases">Bases</span>
    - :material-wrench-outline: [Démarrer sur Azure sans exploser son crédit](tuto-demarrer-azure.md) <span class="kb-badge kb-bases">Bases</span>

-   :material-numeric-2-circle-outline:{ .lg .middle } __Azure : réseau__

    ---

    VNet, sous-réseaux, NSG, sortie Internet et hub & spoke.

    - :material-book-open-variant: [Comprendre le réseau dans Azure](concept-reseau-azure.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-wrench-outline: [Monter un VNet avec une VM publique et une VM privée](tuto-vnet-nsg-azure.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-3-circle-outline:{ .lg .middle } __Azure : identités et droits__

    ---

    Entra ID, RBAC et identités managées : le moindre privilège, sans mot de passe dans les scripts.

    - :material-book-open-variant: [Comprendre les identités et droits dans Azure](concept-identite-azure.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-wrench-outline: [Prouver le RBAC avec une identité managée](tuto-rbac-identite-managee.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-4-circle-outline:{ .lg .middle } __Azure : stockage__

    ---

    Comptes de stockage, redondance, niveaux d'accès, liens SAS et immuabilité.

    - :material-book-open-variant: [Comprendre le stockage dans Azure](concept-stockage-azure.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-wrench-outline: [Stocker et partager des fichiers dans Azure](tuto-stockage-azure-sas.md) <span class="kb-badge kb-inter">Intermédiaire</span>

-   :material-numeric-5-circle-outline:{ .lg .middle } __Infrastructure as Code__

    ---

    Décrire son infrastructure dans des fichiers et la recréer en une commande.

    - :material-book-open-variant: [Comprendre l'Infrastructure as Code](concept-iac.md) <span class="kb-badge kb-inter">Intermédiaire</span>
    - :material-wrench-outline: [Déployer une infrastructure Azure avec Terraform](tuto-terraform-azure.md) <span class="kb-badge kb-inter">Intermédiaire</span>

</div>
