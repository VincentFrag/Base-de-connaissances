---
icon: material/clipboard-text-outline
title: "Démarrer le parcours Azure"
tags: [cloud, azure, azure for students, finops, parcours]
---

# 📋 Démarrer le parcours Azure

Ce cahier transforme tous les exercices et TP du cours « Design infrastructures virtuelles & Cloud » (DENV-R, janvier 2026) en un parcours faisable seul avec Azure for Students : 7 exercices papier, 19 TP et un projet final, pour environ 20 à 30 $ de crédit si tu supprimes tout en fin de séance. Chaque page de TP donne les consignes, et le corrigé est replié sous chaque exercice : essaie d'abord sans.

## Avant de commencer

Ton crédit Azure for Students vaut 100 $ pour 12 mois, sans carte bancaire : quand il est épuisé, Azure désactive tes ressources au lieu de te facturer. Le vrai risque n'est donc pas une facture, c'est de griller le crédit avec une ressource oubliée.

### Les 5 règles du parcours

1. **Un groupe de ressources par TP** (`rg-tp03`, `rg-tp04`…), tagué `tp=<numéro>`. En fin de séance : `az group delete -n rg-tpXX --yes --no-wait`, puis vérifie avec `az group list -o table` qu'il ne reste rien.
2. **Fais le TP1 (budget et alertes) en premier.** C'est à la fois l'exercice FinOps du cours et ton filet de sécurité.
3. **Petites tailles uniquement** : `Standard_B1s` par défaut (incluse dans les 750 h/mois de VM gratuites la première année), `Standard_B2s` quand il faut 4 Go de RAM (Keycloak, Zabbix, Windows). Une VM *arrêtée depuis l'OS* reste facturée : il faut la **désallouer** (`az vm deallocate`).
4. **Ressources à l'heure = créer, tester, supprimer le jour même** : VPN Gateway, Azure Bastion Basic, Private Endpoint, Load Balancer, IP publiques. N'utilise jamais Azure Firewall (plusieurs centaines de dollars par mois) : le TP4 le remplace par une VM Linux passerelle, ce qui est exactement l'exercice réseau du cours.
5. **Ouvre les ports à ton IP seulement** (`<ton IP>/32`), jamais à `*`. Attention : `az vm create` crée par défaut un NSG qui ouvre SSH à tout Internet.

### Limites du compte étudiant

- **Régions** : l'abonnement peut être limité à quelques régions par une stratégie Azure Policy. Si `francecentral` est refusée (`RequestDisallowedByAzure`), liste les régions permises avec `az policy assignment list -o table` et prends la plus proche.
- **Quota vCPU** : il est faible. Vérifie avec `az vm list-usage -l francecentral -o table` et désalloue les VM des TP précédents avant d'en créer d'autres.
- **Tenant de l'école** : tu es propriétaire de ton abonnement, mais pas administrateur Entra ID. Inviter des utilisateurs, créer des inscriptions d'application ou installer Entra Connect peut être bloqué ; les TP concernés (TP6, TP13, TP18) proposent un contournement.
- **Changement depuis le cours** : depuis le 31 mars 2026, les sous-réseaux des nouveaux VNet sont *privés par défaut*. Une VM sans IP publique n'a plus Internet (fin du SNAT implicite décrit slide 73) : il lui faut une IP publique, une NAT Gateway, une règle sortante de Load Balancer ou une VM passerelle. Plusieurs TP s'appuient là-dessus.

### Outils à installer sur ton PC

- **WSL (Ubuntu)** : tous les corrigés sont en bash. Dans WSL : Azure CLI (`curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`), Terraform, Ansible, AWS CLI.
- **Postman**, le client **WireGuard** pour Windows, **Docker** (dans WSL ou Docker Desktop).
- Une clé SSH dédiée : `ssh-keygen -t ed25519 -f ~/.ssh/azure_lab`. Les corrigés utilisent `--generate-ssh-keys` (clé RSA `~/.ssh/id_rsa`) pour rester simples ; tu peux la remplacer par `--ssh-key-values ~/.ssh/azure_lab.pub`.
- Cloud Shell (portail) dépanne, mais son IP n'est pas la tienne : les règles « mon IP » ne marcheront pas depuis Cloud Shell.

### Format de chaque TP

Objectif, slides du cours concernées, coût estimé, consignes, ce que tu dois observer pour valider, questions de réflexion, nettoyage. Fais chaque TP d'abord au portail pour comprendre les écrans, puis refais-le en CLI à partir du corrigé : c'est la CLI qui prépare au bloc Automatisation et à l'AZ-104.

### Variables des corrigés

Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

## Le parcours

L'ordre suit le cours, regroupé pour réutiliser les mêmes briques d'un TP à l'autre ; total estimé ≈ 15 $ de crédit pour les TP, 3 à 5 $ pour le projet final. La colonne « Slides » renvoie aux pages du support de cours.

| # | Exercice / TP | Slides | Durée | Coût estimé |
| --- | --- | --- | --- | --- |
| E1 | Comprendre l'offre OVHcloud | 38 | 1 h | 0 $ |
| E2 | Dimensionnement cloud privé vs public | 25-28, 130-132 | 45 min | 0 $ |
| E3 | SLA composites | 141 | 30 min | 0 $ |
| E4 | Plan d'adressage VNet | 61-63 | 45 min | 0 $ |
| E5 | Lecture de règles NSG | 61, 133 | 30 min | 0 $ |
| E6 | Migration : les 7R | 104-107 | 45 min | 0 $ |
| E7 | Plan de sauvegarde, RPO/RTO | 112-114 | 1 h | 0 $ |
| TP1 | Budget, alertes et tags (FinOps) | 39-42 | 30 min | 0 $ |
| TP2 | Première VM + cloud-init | 57-59, 64-65 | 1 h | < 0,20 $ |
| TP3 | VNet, sous-réseaux, NSG, Network Watcher | 61-63, 74 | 1 h 30 | < 0,30 $ |
| TP4 | Hub & spoke avec VM passerelle | 62, 69, 76-79 | 2 h | < 0,50 $ |
| TP5 | Load Balancer L4 | 65-68, 72 | 1 h 30 | < 0,50 $ |
| TP6 | RBAC et identité managée | 46, 51-56 | 1 h | < 0,10 $ |
| TP7 | Keycloak : VM Docker puis Container Instance | 56, 75, 134 | 1 h 30 | < 0,50 $ |
| TP8 | Azure Bastion puis Teleport | 82-83 | 2 h | < 1 $ |
| TP9 | WireGuard entre deux VNet | 49, 81 | 1 h 30 | < 0,30 $ |
| TP10 | VPN Gateway point-à-site | 49, 81 | 2 h (dont 45 min d'attente) | 0,50 à 2 $ |
| TP11 | Compte de stockage, SAS, immuabilité | 85-100, 127-129 | 1 h 30 | < 0,10 $ |
| TP12 | Service Endpoint vs Private Endpoint | 48, 70, 101 | 1 h 30 | < 0,50 $ |
| TP13 | Postman et API REST Azure | 110, 136 | 1 h 30 | 0 $ |
| TP14 | AWS CLI et S3 Object Lock | 110, 127-129 | 1 h | 0 $ (offre gratuite AWS) |
| TP15 | Supervision Zabbix + Azure Monitor | 108, 122 | 2 h | < 0,50 $ |
| TP16 | Azure Backup et restauration | 111-115 | 1 h 30 | < 0,50 $ |
| TP17 | IaC : script CLI, Terraform, Ansible, autoscaling | 117-120, 122 | 3 h | < 1 $ |
| TP18 | Windows Server + AD DS sur Azure | 51, 121 | 2 h | < 1 $ |
| TP19 | Durcissement d'une image et d'une VM | 121 | 1 h 30 | 0 $ |
| Final | Architecture de référence en Terraform | 48 | 1 journée | 3 à 5 $ |

Les coûts sont des ordres de grandeur pour une séance suivie d'une suppression immédiate ; vérifie les prix sur la calculatrice Azure avant les TP marqués « à l'heure » (TP8, TP10, TP12).

## Sources

Cours « Design infrastructures virtuelles & Cloud », DENV-R, 19-22 janvier 2026 (support de cours). Documentation consultée le 13 octobre 2026 :

- [Azure for Students](https://azure.microsoft.com/en-us/free/students) — crédit de 100 $ sur 12 mois, sans carte bancaire, services gratuits inclus.
- [Default outbound access in Azure](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access) — sous-réseaux privés par défaut depuis le 31 mars 2026.
- [az network vnet subnet](https://learn.microsoft.com/en-us/cli/azure/network/vnet/subnet) — paramètre `--default-outbound-access`.
- [Azure Bastion SKU comparison](https://learn.microsoft.com/azure/bastion/bastion-sku-comparison) — Developer gratuit, sans appairage ; Basic minimum pour l'appairage.
- [VPN Gateway SKU consolidation](https://docs.azure.cn/en-us/vpn-gateway/gateway-sku-consolidation) — plus de création de passerelles non zonales depuis novembre 2025 ; Basic maintenue avec IP publique Standard.
- [MinIO CE in 2026: retired upstream, source-only](https://glukhov.org/data-infrastructure/object-storage/minio-dead/) — fin de la distribution binaire de MinIO communautaire.
