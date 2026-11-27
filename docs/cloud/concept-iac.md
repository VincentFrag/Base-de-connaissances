---
icon: material/book-open-variant
title: L'Infrastructure as Code (Terraform, état, impératif vs déclaratif)
tags: [cloud, intermédiaire, concept, iac, terraform, ansible, automatisation, azure]
---

# 🧠 L'Infrastructure as Code : décrire son infra dans des fichiers

!!! abstract "En une phrase"
    L'**Infrastructure as Code** (IaC) consiste à décrire réseaux, VM et stockages dans des **fichiers texte versionnés**, qu'un outil comme **Terraform** compare à la réalité pour créer, modifier ou détruire exactement ce qu'il faut ; l'infrastructure devient reproductible, relue et historisée comme du code.

## L'analogie

Une recette de cuisine contre un cuisinier qui improvise :

- **cliquer dans le portail**, c'est cuisiner de tête : le plat est bon, mais personne ne sait le refaire à l'identique ;
- **un script**, c'est une liste d'actions (« ajoute 200 g de farine ») : relancé deux fois, il met 400 g ;
- **Terraform**, c'est la photo du plat fini (« il doit y avoir 200 g de farine ») : il regarde ce qui est déjà dans le bol et n'ajoute que ce qui manque.

---

## Impératif ou déclaratif

| | **Impératif** (script) | **Déclaratif** (IaC) |
|---|---|---|
| On écrit | **Les étapes** à exécuter | **L'état final** voulu |
| Exemple | Script Bash avec `az vm create` | Fichier Terraform `azurerm_linux_virtual_machine` |
| Relancer | Peut planter ou créer des doublons, sauf si on teste tout à la main | Ne change **rien** si tout est déjà conforme |
| Modifier | Écrire une nouvelle commande `update` | Modifier le fichier ; l'outil calcule la différence |
| Supprimer | Écrire les commandes de suppression dans le bon ordre | `terraform destroy` |
| Voir ce qui va changer | Non | **Oui** : `terraform plan` |

Un script qui vérifie avant d'agir (« le groupe existe-t-il déjà ? ») est dit **idempotent** : c'est faisable, mais c'est à toi de l'écrire pour chaque ressource.

## Les outils

| Outil | Rôle | Particularité |
|---|---|---|
| **Terraform** / **OpenTofu** | Créer l'**infrastructure** (réseau, VM, stockage) | Multi-fournisseur (Azure, AWS, Proxmox…) ; OpenTofu en est le fork libre |
| **Bicep** / modèles ARM | Créer l'infrastructure, Azure uniquement | Natif Azure, pas de fichier d'état à gérer |
| **Ansible** | **Configurer** l'intérieur des machines (paquets, fichiers, services) | Sans agent, par SSH |
| **cloud-init** | Configurer une VM **au premier démarrage** | Intégré aux images Linux cloud |

On les **combine** : Terraform crée la VM, cloud-init ou Ansible la configure.

## Le cycle Terraform

| Commande | Ce qu'elle fait |
|---|---|
| `terraform init` | Télécharge les fournisseurs (*providers*), prépare le dossier |
| `terraform fmt` / `validate` | Met en forme et vérifie la syntaxe |
| `terraform plan` | Compare le code, l'état et la réalité ; affiche `+` créer, `~` modifier, `-` détruire, `-/+` recréer |
| `terraform apply` | Applique le plan après confirmation |
| `terraform destroy` | Supprime tout ce que ce code a créé |

## Le fichier d'état

Terraform garde dans `terraform.tfstate` la correspondance entre chaque bloc du code et la ressource réelle (son identifiant Azure).

| Question | Réponse |
|---|---|
| Pourquoi c'est important ? | Sans lui, Terraform ne sait plus ce qu'il a créé et voudra tout recréer |
| Pourquoi c'est sensible ? | Il contient en clair des valeurs comme les mots de passe ou les clés générés |
| Où le mettre en équipe ? | Dans un **backend distant** (un compte de stockage Azure), avec verrouillage, jamais dans Git |

## La dérive

Quelqu'un modifie une règle de pare-feu à la main dans le portail : la réalité ne correspond plus au code, c'est une **dérive** (*drift*). Le prochain `terraform plan` la détecte et propose de remettre la règle comme dans le code. D'où la règle d'équipe : **une fois en IaC, on ne touche plus au portail**.

---

## À quoi ça sert concrètement

- **Recréer un lab complet en 5 minutes** et le détruire le soir, sans rien oublier ni rien payer pour rien.
- **Dupliquer** un environnement : même code, variables différentes pour test et production.
- **Relire** une modification d'infrastructure dans une pull request avant de l'appliquer.
- **Documenter** : le code *est* la documentation à jour de l'infrastructure.

!!! warning "Pièges fréquents"
    - **Committer `terraform.tfstate`** ou un fichier `*.tfvars` contenant des secrets : les ajouter au `.gitignore`.
    - Modifier à la main ce que Terraform gère : le prochain `apply` annulera la modification.
    - Lire `terraform plan` en diagonale : un `-/+` (recréation) sur une VM ou un disque signifie **perte des données**.
    - Ne pas fixer les versions des fournisseurs (`version = "~> 4.0"`) : une mise à jour majeure peut casser le code.
    - Supprimer le fichier d'état « pour repartir de zéro » : les ressources existent toujours dans Azure, mais Terraform ne les connaît plus.

## Pour aller plus loin

- 🛠️ **[Déployer une infrastructure Azure avec Terraform](tuto-terraform-azure.md)** : la mise en pratique (VNet, NSG, VM, plan, dérive, destroy)
- 🧠 [Ansible](../transverse/concept-ansible.md) : configurer les machines une fois créées
- 🧠 [Le réseau dans Azure](concept-reseau-azure.md) : ce qu'on va décrire en code

??? quote "Sources"
    - [Terraform : introduction](https://developer.hashicorp.com/terraform/intro)
    - [Terraform : l'état (state)](https://developer.hashicorp.com/terraform/language/state)
    - [Provider Terraform azurerm](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
