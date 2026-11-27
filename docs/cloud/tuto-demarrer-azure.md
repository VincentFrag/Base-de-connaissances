---
icon: material/wrench-outline
title: Démarrer sur Azure sans exploser son crédit
tags: [cloud, tuto, azure, azure cli, finops, budget, tags, azure for students]
---

# 🛠️ Démarrer sur Azure sans exploser son crédit

!!! abstract "Objectif"
    Ouvrir un compte **Azure for Students**, poser les garde-fous avant la moindre VM (**budget avec alertes**, **convention de tags**), installer **Azure CLI** et prendre les bons réflexes : un groupe de ressources par projet, désallouer, tout supprimer en fin de séance.

**Niveau :** 🟢 Bases · **Durée :** 45 min

## Prérequis

- Comprendre IaaS, régions et modèle de coût → 🧠 [Le cloud computing](concept-cloud-computing.md)
- Une adresse e-mail d'établissement scolaire (pour l'offre étudiante)
- Un PC sous Windows (avec WSL de préférence) ou Linux

## Le plan

| Élément | Valeur dans ce tuto |
|---|---|
| Abonnement | Azure for Students (crédit gratuit, sans carte bancaire) |
| Région | `francecentral` |
| Groupe de ressources | `rg-demo` (un groupe par projet ou par séance) |
| Tags obligatoires | `projet`, `owner`, `env` |
| Budget | 10 $ par mois, alertes à 50 %, 80 % et 100 % |

---

## Étape 1 — Activer l'offre étudiante

1. Aller sur la page **Azure for Students** de Microsoft et se connecter avec l'adresse de l'école.
2. Valider le compte : l'offre donne un **crédit** (100 $ à ce jour) valable 12 mois, plus des services gratuits.
3. Ouvrir le **portail** `https://portal.azure.com` et vérifier dans **Abonnements** qu'un abonnement *Azure for Students* est actif.

!!! info "Pas de carte bancaire, mais un crédit qui fond"
    Quand le crédit est épuisé, Azure **désactive** tes ressources au lieu de te facturer. Le risque n'est donc pas une facture, c'est de perdre ton terrain de jeu pour le reste de l'année à cause d'une ressource oubliée.

## Étape 2 — Installer Azure CLI et se connecter

=== "Linux / WSL"

    ```bash
    curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
    az version
    az login            # ouvre le navigateur ; sinon : az login --use-device-code
    az account show -o table
    ```

=== "Windows (PowerShell)"

    ```powershell
    winget install --exact --id Microsoft.AzureCLI
    # rouvrir le terminal, puis :
    az version
    az login
    az account show -o table
    ```

Fixer la région par défaut pour ne plus la répéter :

```bash
az config set defaults.location=francecentral
```

!!! tip "Cloud Shell"
    Le portail propose aussi un terminal intégré (icône `>_`), avec Azure CLI déjà connecté. Pratique pour dépanner, mais son adresse IP n'est **pas** celle de ton PC : une règle de pare-feu « mon IP » ne marchera pas depuis Cloud Shell.

## Étape 3 — Créer un budget avec alertes

=== "Portail"

    1. **Cost Management + Billing** → **Budgets** → **Ajouter**.
    2. Portée : ton abonnement. Montant : **10 $**, période **mensuelle**.
    3. Alertes : **50 %**, **80 %** et **100 %** du réel, plus **100 % de la prévision**, envoyées à ton adresse.

=== "Azure CLI"

    ```bash
    az consumption budget create --budget-name budget-mensuel \
      --amount 10 --category cost --time-grain monthly \
      --start-date 2026-12-01 --end-date 2027-11-30
    ```

    Les seuils d'alerte et les destinataires se règlent ensuite dans le portail (Budgets → `budget-mensuel` → Modifier).

!!! note "Si les budgets sont indisponibles"
    Certaines offres éducation ne proposent pas les budgets. Dans ce cas, suis ton solde dans **Education Hub** (rechercher « Education » dans le portail) au moins une fois par semaine.

## Étape 4 — Imposer une convention de tags

Les **tags** sont des étiquettes `clé=valeur` posées sur les ressources. Ils permettent ensuite de filtrer les coûts par projet dans *Cost analysis*.

| Tag | Rôle | Exemple |
|---|---|---|
| `projet` | À quoi sert la ressource | `demo`, `site-vitrine` |
| `owner` | Qui l'a créée | `alice` |
| `env` | Environnement | `lab`, `test`, `prod` |

Pour que personne n'oublie (toi compris), on fait refuser par Azure tout groupe de ressources sans tag `projet`, avec une **Azure Policy** intégrée :

```bash
SUB=$(az account show --query id -o tsv)
DEF=$(az policy definition list \
  --query "[?displayName=='Require a tag on resource groups'].name" -o tsv)

az policy assignment create -n exiger-tag-projet \
  --policy "$DEF" --scope /subscriptions/$SUB \
  --params '{"tagName":{"value":"projet"}}'
```

Tester (attendre quelques minutes que la règle s'applique) :

```bash
az group create -n rg-sans-tag                       # doit échouer : RequestDisallowedByPolicy
az group create -n rg-demo --tags projet=demo owner=alice env=lab
```

## Étape 5 — Les réflexes de chaque séance

| Moment | Commande | Pourquoi |
|---|---|---|
| Début | `az group create -n rg-<projet> --tags projet=<projet> …` | Tout ce que tu crées ira dans ce groupe |
| Pause | `az vm deallocate -g rg-demo -n <vm>` | Une VM arrêtée depuis l'OS reste facturée ; désallouée, non |
| Fin | `az group delete -n rg-demo --yes --no-wait` | Supprime **tout** le contenu du groupe d'un coup |
| Contrôle | `az group list -o table` | Vérifier qu'il ne reste rien d'oublié |
| Quota | `az vm list-usage -l francecentral -o table` | Les comptes étudiants ont peu de vCPU |

!!! tip "Arrêt automatique"
    Sur chaque VM : `az vm auto-shutdown -g rg-demo -n <vm> --time 1900` (heure UTC). Une VM oubliée s'éteindra toute seule le soir.

## Ce que tu vois / ce que ça veut dire / action

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `RequestDisallowedByPolicy` | Une Azure Policy bloque la création (tag manquant, région interdite) | Lire le nom de la policy dans le message ; ajouter le tag ou changer de région |
| `RequestDisallowedByAzure` sur la région | L'abonnement étudiant est limité à certaines régions | `az account list-locations -o table`, essayer `westeurope` ou une région proposée |
| `QuotaExceeded` / `OperationNotAllowed` sur une VM | Quota de vCPU atteint | Désallouer ou supprimer d'autres VM, choisir une taille plus petite |
| `SkuNotAvailable` | Cette taille de VM n'est pas disponible dans la région ou la zone | `az vm list-skus -l francecentral --size Standard_B -o table` |
| `AuthorizationFailed` | Ton compte n'a pas le rôle nécessaire sur cette portée | Vérifier l'abonnement actif : `az account show` |

## Vérification finale

- [ ] `az account show` affiche l'abonnement Azure for Students
- [ ] Le budget apparaît dans Cost Management avec ses seuils
- [ ] `az group create -n rg-sans-tag` est refusé, `rg-demo` avec ses tags est accepté
- [ ] `az group delete -n rg-demo --yes` puis `az group list -o table` : plus rien d'inutile

## Dépannage courant

- **`az login` ne s'ouvre pas dans WSL** : utiliser `az login --use-device-code` et saisir le code dans un navigateur Windows.
- **Plusieurs abonnements** (école, perso) : `az account list -o table` puis `az account set -s "<nom ou id>"`.
- **Le groupe `NetworkWatcherRG` est refusé par la policy** : Azure le crée tout seul au premier réseau virtuel. Le créer toi-même avant, avec le tag : `az group create -n NetworkWatcherRG --tags projet=infra`.

## Pour aller plus loin

- 🧠 [Le cloud computing](concept-cloud-computing.md) : modèles, régions, SLA, coûts
- 🧠 [Le réseau dans Azure](concept-reseau-azure.md) : l'étape suivante, avant ta première VM

??? quote "Sources"
    - [Compte gratuit Azure](https://azure.microsoft.com/fr-fr/pricing/free-services/)
    - [Microsoft Learn : créer et gérer des budgets](https://learn.microsoft.com/fr-fr/azure/cost-management-billing/costs/tutorial-acm-create-budgets)
    - [Microsoft Learn : groupes de ressources](https://learn.microsoft.com/fr-fr/azure/azure-resource-manager/management/manage-resource-groups-portal)
    - [Documentation Azure CLI](https://learn.microsoft.com/fr-fr/cli/azure/)
