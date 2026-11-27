---
icon: material/wrench-outline
title: Prouver le RBAC avec une identité managée
tags: [cloud, tuto, azure, azure cli, powershell, rbac, identité managée, stockage]
---

# 🛠️ Prouver le RBAC avec une identité managée

!!! abstract "Objectif"
    Donner une **identité managée** à une VM, puis vérifier par l'expérience ce que chaque rôle autorise : d'abord **Lecteur** sur un groupe de ressources (lecture oui, modification non), ensuite **Contributeur de données Blob** sur un compte de stockage, pour qu'un script de la VM y dépose une sauvegarde **sans aucun mot de passe ni clé**.

**Niveau :** 🟡 Intermédiaire · **Durée :** 45 min · **Coût :** quelques centimes

## Prérequis

- Comprendre principal, rôle, portée et identité managée → 🧠 [Identités et droits dans Azure](concept-identite-azure.md)
- Azure CLI installé et connecté → 🛠️ [Démarrer sur Azure](tuto-demarrer-azure.md)
- Être Propriétaire de son abonnement (c'est le cas avec Azure for Students)

## Le plan

| Élément | Valeur |
|---|---|
| Groupe de ressources | `rg-rbac` |
| VM | `vm-ops01` · Ubuntu 24.04 · `Standard_B1s` · identité managée **système** |
| Compte de stockage | `stsauvegardes<nombre>` · conteneur `backups` |
| Rôle 1 | **Reader** sur `rg-rbac` |
| Rôle 2 | **Storage Blob Data Contributor** sur le compte de stockage |

---

## Étape 1 — La VM et son identité

```bash
RG=rg-rbac; LOC=francecentral
az group create -n $RG -l $LOC --tags projet=rbac owner=alice env=lab

az vm create -g $RG -n vm-ops01 --image Ubuntu2404 --size Standard_B1s \
  --assign-identity '[system]' --public-ip-sku Standard --nsg-rule NONE \
  --admin-username azureuser --generate-ssh-keys

# L'identifiant de l'identité (principalId) : c'est lui qui recevra les rôles
PID=$(az vm show -g $RG -n vm-ops01 --query identity.principalId -o tsv)
echo $PID
```

Ouvrir SSH à ton IP seulement :

```bash
MYIP=$(curl -s https://ifconfig.me)
az network nsg rule create -g $RG --nsg-name vm-ops01NSG -n ssh-admin --priority 100 \
  --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow
```

## Étape 2 — Donner le rôle Lecteur sur le groupe

=== "Azure CLI"

    ```bash
    RGID=$(az group show -n $RG --query id -o tsv)
    az role assignment create --assignee-object-id $PID \
      --assignee-principal-type ServicePrincipal --role Reader --scope $RGID
    ```

=== "PowerShell (module Az)"

    ```powershell
    $vm = Get-AzVM -ResourceGroupName rg-rbac -Name vm-ops01
    $rg = Get-AzResourceGroup -Name rg-rbac
    New-AzRoleAssignment -ObjectId $vm.Identity.PrincipalId `
      -RoleDefinitionName Reader -Scope $rg.ResourceId
    ```

=== "Portail"

    `rg-rbac` → **Contrôle d'accès (IAM)** → **Ajouter une attribution de rôle** → rôle **Lecteur** → *Attribuer l'accès à* : **Identité managée** → sélectionner `vm-ops01`.

## Étape 3 — Tester depuis la VM

```bash
IP=$(az vm show -d -g $RG -n vm-ops01 --query publicIps -o tsv)
ssh azureuser@$IP
```

Sur la VM :

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login --identity                       # pas de mot de passe : c'est l'identité de la VM

az resource list -g rg-rbac -o table      # ✅ fonctionne : lire est permis
az vm update -g rg-rbac -n vm-ops01 --set tags.test=1
                                          # ❌ AuthorizationFailed : modifier est interdit
az group create -n rg-pirate -l francecentral --tags projet=x
                                          # ❌ AuthorizationFailed : rien hors de la portée
```

Le message d'erreur indique l'**action** refusée (`Microsoft.Compute/virtualMachines/write`) et la **portée** : c'est exactement ce que le rôle Reader ne contient pas.

## Étape 4 — Le compte de stockage et le rôle sur les données

Depuis ton PC :

```bash
SA=stsauvegardes$RANDOM
az storage account create -g $RG -n $SA --sku Standard_LRS --kind StorageV2 \
  --min-tls-version TLS1_2 --allow-blob-public-access false
SAID=$(az storage account show -g $RG -n $SA --query id -o tsv)

az role assignment create --assignee-object-id $PID \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope $SAID
echo $SA
```

!!! info "Toi aussi, tu as besoin de ce rôle"
    Même Propriétaire de l'abonnement, tu ne peux pas lire les blobs avec `--auth-mode login` sans un rôle « … Data … ». C'est la séparation entre **plan de contrôle** et **plan de données**.

## Étape 5 — Un script de sauvegarde sans secret

Sur la VM (attendre 2 à 5 minutes que le rôle se propage), remplacer `<SA>` par le nom affiché :

```bash
SA=<SA>
az login --identity
az storage container create --account-name $SA -n backups --auth-mode login

tar czf /tmp/etc-$(hostname)-$(date +%F).tar.gz -C / etc 2>/dev/null
az storage blob upload --account-name $SA -c backups --auth-mode login \
  -f /tmp/etc-$(hostname)-$(date +%F).tar.gz -n $(hostname)/etc-$(date +%F).tar.gz

az storage blob list --account-name $SA -c backups --auth-mode login -o table
```

Aucune clé, aucun mot de passe n'apparaît dans le script : il peut être versionné dans Git sans risque. Il ne reste qu'à le planifier, voir 🛠️ [Automatiser une tâche avec cron](../transverse/tuto-taches-planifiees.md).

## Étape 6 — Lire les attributions

```bash
az role assignment list --assignee $PID --all -o table
```

Côté portail : sur la ressource → **Contrôle d'accès (IAM)** → **Vérifier l'accès**, choisir l'identité de la VM : on voit tous ses rôles, hérités compris.

## Ce que tu vois / ce que ça veut dire / action

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `AuthorizationFailed … does not have authorization to perform action '…/write'` | Le rôle ne contient pas cette action, ou pas sur cette portée | Normal avec Reader ; sinon vérifier rôle et portée |
| `az login --identity` : `No managed identity` | La VM n'a pas d'identité | `az vm identity assign -g rg-rbac -n vm-ops01` |
| `You do not have the required permissions needed to perform this operation` sur un blob | Pas de rôle sur le **plan de données** | Ajouter *Storage Blob Data Contributor* ou *Reader* |
| L'upload échoue juste après l'attribution | Propagation du RBAC en cours | Attendre quelques minutes, refaire `az login --identity` |
| `Principal … does not exist in the directory` | Le `principalId` est faux ou l'identité vient d'être créée | Revérifier `$PID`, réessayer une minute plus tard |

## Vérification finale

- [ ] Avec Reader, `az resource list` fonctionne et toute modification renvoie `AuthorizationFailed`
- [ ] Le fichier `etc-….tar.gz` apparaît dans le conteneur `backups`
- [ ] Aucun secret dans le script ni dans `~/.azure` autre que le jeton de l'identité
- [ ] `az role assignment list` montre exactement deux rôles pour l'identité

## Nettoyage

```bash
az group delete -n rg-rbac --yes --no-wait
```

La suppression du groupe supprime la VM, donc son identité système et ses attributions de rôle.

## Dépannage courant

- **Le portail ne propose pas « Identité managée »** dans l'attribution : la VM a été créée sans `--assign-identity`. L'ajouter (onglet **Identité** de la VM → *Attribuée par le système* → Activé).
- **Plusieurs abonnements** : sur la VM, `az account show` doit indiquer le même abonnement que `rg-rbac`.
- **Le compte de l'école interdit d'inviter des utilisateurs** : c'est justement l'intérêt de l'identité managée, qui ne demande aucun droit d'administration sur le tenant.

## Pour aller plus loin

- 🧠 [Identités et droits dans Azure](concept-identite-azure.md) : rôles personnalisés, ABAC, accès conditionnel
- 🛠️ [Stocker et partager des fichiers dans Azure](tuto-stockage-azure-sas.md) : SAS, cycle de vie et immuabilité
- 🧠 [Comptes et droits Linux](../systeme/concept-permissions-linux.md) : le moindre privilège, version Linux

??? quote "Sources"
    - [Microsoft Learn : attribuer des rôles avec Azure CLI](https://learn.microsoft.com/fr-fr/azure/role-based-access-control/role-assignments-cli)
    - [Microsoft Learn : rôles intégrés Azure](https://learn.microsoft.com/fr-fr/azure/role-based-access-control/built-in-roles)
    - [Microsoft Learn : identités managées](https://learn.microsoft.com/fr-fr/entra/identity/managed-identities-azure-resources/overview)
