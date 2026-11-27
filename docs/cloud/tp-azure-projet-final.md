---
icon: material/flask-outline
title: "Projet final : l'architecture de référence en Terraform"
tags: [cloud, azure, tp, projet, terraform, ansible, hub and spoke, architecture]
---

# 🧪 Projet final : l'architecture de référence en Terraform

!!! abstract "Ce que tu vas faire"
    Reconstruire en Terraform, à l'échelle d'un budget étudiant, l'architecture de référence du cours (hub, spoke applicatif, stockage privé), la documenter comme pour un vrai client, puis tout détruire.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 journée · **Coût :** 3 à 5 $

!!! info "Prérequis"
    Ce projet réutilise tous les blocs du parcours : [Fondations](tp-azure-fondations.md), [Réseau avancé](tp-azure-reseau-avance.md), [Accès sécurisé](tp-azure-acces-securise.md), [Stockage](tp-azure-stockage.md), [Exploitation](tp-azure-exploitation.md) et [Automatisation](tp-azure-automatisation.md). Pas de corrigé : c'est la synthèse de leurs commandes.

Le schéma de la p. 48 (hub avec pare-feu, Bastion et passerelle VPN ; spoke applicatif en trois niveaux ; stockage privé) résume tout le cours. Le projet le reconstruit en Terraform, à l'échelle d'un budget étudiant, en une journée, puis le détruit.

**Ce que tu déploies, uniquement en Terraform** :

- un hub : VM passerelle (TP4) et point d'entrée d'administration WireGuard ou Teleport (TP8-9) ;
- un spoke web : deux VM en zones différentes derrière un Load Balancer (TP5), sorties Internet via la passerelle du hub ;
- un compte de stockage atteint uniquement par Private Endpoint (TP12) ;
- supervision (TP15) et sauvegarde d'une VM (TP16) ;
- tags, budget et arrêt automatique (TP1) ; Ansible pour la configuration (TP17).

**Ce que tu rends, comme pour un vrai client** :

1. Un schéma d'architecture (draw.io ou Excalidraw) avec le plan d'adressage (E4).
2. Un dossier d'architecture technique de 3 à 5 pages : choix et alternatives écartées (pourquoi pas Azure Firewall, pourquoi pas App Gateway), matrice des flux et NSG (E5), SLA composite (E3), RPO/RTO (E7).
3. Une estimation de coût mensuel « production » avec la calculatrice Azure, puis la version optimisée FinOps (réservations, tailles, arrêts planifiés).
4. Le dépôt Git : Terraform, playbooks, collection Postman, README de déploiement et de destruction.

**Validation** : `terraform apply` depuis zéro reconstruit tout sans intervention manuelle, et `terraform destroy` ne laisse aucune ressource. C'est aussi une bonne pièce de portfolio pour un poste d'administrateur systèmes multisite.
