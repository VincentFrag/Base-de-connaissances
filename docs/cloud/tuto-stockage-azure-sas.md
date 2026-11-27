---
icon: material/wrench-outline
title: Stocker et partager des fichiers dans Azure
tags: [cloud, tuto, azure, azure cli, powershell, stockage, blob, sas, azcopy, cycle de vie, immuabilité, sauvegarde]
---

# 🛠️ Stocker et partager des fichiers dans Azure

!!! abstract "Objectif"
    Utiliser Azure comme **copie hors site** des sauvegardes du homelab : créer un compte de stockage sécurisé, y envoyer une sauvegarde depuis `pve01` avec **AzCopy**, partager un fichier par un lien **SAS** qui expire tout seul, faire descendre les vieux fichiers en niveau froid avec une **règle de cycle de vie**, et les protéger contre la suppression avec l'**immuabilité**.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 h · **Coût :** quelques centimes par Go et par mois

## Prérequis

- Comprendre redondance, niveaux et SAS → 🧠 [Le stockage dans Azure](concept-stockage-azure.md)
- Azure CLI installé et connecté → 🛠️ [Démarrer sur Azure](tuto-demarrer-azure.md)
- Une sauvegarde à envoyer, par exemple un fichier `vzdump` de `pve01` → 🛠️ [Sauvegarder ses VM avec Proxmox](../transverse/tuto-proxmox-backup.md)

## Le plan

| Élément | Valeur |
|---|---|
| Groupe de ressources | `rg-horssite` |
| Compte de stockage | `sthorssite<nombre>` · StorageV2 · **LRS** (lab) · TLS 1.2 · pas d'accès public anonyme |
| Conteneur | `pve01` (un conteneur par machine sauvegardée) |
| Partage ponctuel | SAS de délégation d'utilisateur, lecture seule, 2 h, HTTPS |
| Cycle de vie | Froid à 30 jours, très froid à 90 jours, suppression à 365 jours |
| Immuabilité | 7 jours, **non verrouillée** (lab) |

---

## Étape 1 — Créer le compte de stockage

=== "Azure CLI"

    ```bash
    RG=rg-horssite; LOC=francecentral; SA=sthorssite$RANDOM
    az group create -n $RG -l $LOC --tags projet=horssite owner=alice env=lab

    az storage account create -g $RG -n $SA --sku Standard_LRS --kind StorageV2 \
      --access-tier Hot --min-tls-version TLS1_2 \
      --allow-blob-public-access false --allow-shared-key-access false
    echo $SA
    ```

=== "PowerShell (module Az)"

    ```powershell
    $rg = "rg-horssite"; $sa = "sthorssite$(Get-Random -Maximum 99999)"
    New-AzResourceGroup -Name $rg -Location francecentral -Tag @{projet="horssite"; owner="alice"; env="lab"}
    New-AzStorageAccount -ResourceGroupName $rg -Name $sa -Location francecentral `
      -SkuName Standard_LRS -Kind StorageV2 -AccessTier Hot -MinimumTlsVersion TLS1_2 `
      -AllowBlobPublicAccess $false -AllowSharedKeyAccess $false
    ```

`--allow-shared-key-access false` désactive les **clés du compte** : tout passera par Entra ID (ton compte, une identité managée) et les SAS de délégation d'utilisateur, qui sont révocables et traçables.

## Étape 2 — Te donner le rôle sur les données

Propriétaire de l'abonnement ne suffit pas pour lire et écrire les blobs (plan de données) :

```bash
ME=$(az ad signed-in-user show --query id -o tsv)
SAID=$(az storage account show -g $RG -n $SA --query id -o tsv)
az role assignment create --assignee $ME --role "Storage Blob Data Contributor" --scope $SAID
```

Attendre 2 à 5 minutes, puis :

```bash
az storage container create --account-name $SA -n pve01 --auth-mode login
```

## Étape 3 — Envoyer une sauvegarde avec AzCopy

**AzCopy** est l'outil de transfert de Microsoft : parallèle, avec reprise, idéal pour les gros fichiers.

=== "Linux (pve01)"

    ```bash
    wget -O azcopy.tar.gz https://aka.ms/downloadazcopy-v10-linux
    tar xzf azcopy.tar.gz --strip-components=1 --wildcards '*/azcopy'
    sudo mv azcopy /usr/local/bin/

    azcopy login --login-type=device      # code à saisir dans un navigateur
    azcopy copy "/var/lib/vz/dump/vzdump-qemu-110-*.vma.zst" \
      "https://<SA>.blob.core.windows.net/pve01/" --put-md5
    ```

=== "Windows (PowerShell)"

    ```powershell
    winget install --exact --id Microsoft.Azure.AZCopy.10
    azcopy login
    azcopy copy "D:\Sauvegardes\*" "https://<SA>.blob.core.windows.net/pve01/" --recursive --put-md5
    ```

Vérifier :

```bash
az storage blob list --account-name $SA -c pve01 --auth-mode login \
  --query "[].{nom:name, taille:properties.contentLength, niveau:properties.blobTier}" -o table
```

!!! tip "Synchroniser plutôt que copier"
    `azcopy sync /var/lib/vz/dump "https://<SA>.blob.core.windows.net/pve01/"` n'envoie que les fichiers nouveaux ou modifiés. À planifier chaque nuit après le job `vzdump`, voir 🛠️ [Automatiser une tâche avec cron](../transverse/tuto-taches-planifiees.md).

## Étape 4 — Partager un fichier par un lien SAS

Un SAS **de délégation d'utilisateur** est signé par ton identité Entra ID, pas par une clé : il est limité dans le temps et révocable.

=== "Azure CLI"

    ```bash
    FIN=$(date -u -d '+2 hours' +%Y-%m-%dT%H:%MZ)
    BLOB=$(az storage blob list --account-name $SA -c pve01 --auth-mode login --query "[0].name" -o tsv)

    az storage blob generate-sas --account-name $SA -c pve01 -n "$BLOB" \
      --permissions r --expiry $FIN --https-only \
      --auth-mode login --as-user --full-uri -o tsv
    ```

=== "PowerShell (module Az)"

    ```powershell
    $ctx  = New-AzStorageContext -StorageAccountName $sa -UseConnectedAccount
    $blob = (Get-AzStorageBlob -Container pve01 -Context $ctx | Select-Object -First 1).Name
    New-AzStorageBlobSASToken -Container pve01 -Blob $blob -Permission r `
      -ExpiryTime (Get-Date).AddHours(2) -Protocol HttpsOnly -Context $ctx -FullUri
    ```

Ouvrir l'URL obtenue dans un navigateur privé : le téléchargement démarre. Retirer `sig=…` de l'URL : `AuthenticationFailed`. Après 2 h : `AuthenticationFailed` aussi.

| Paramètre de l'URL | Signification |
|---|---|
| `sp=r` | Droits : lecture seulement |
| `se=…` | Date et heure d'expiration (UTC) |
| `spr=https` | HTTPS obligatoire |
| `skoid=…` | Identité qui a signé (délégation d'utilisateur) |
| `sig=…` | Signature : sans elle, rien ne marche |

!!! warning "Révoquer avant l'expiration"
    Un SAS de délégation d'utilisateur se révoque en invalidant **toutes** les clés de délégation du compte : `az storage account revoke-delegation-keys -g $RG -n $SA`.

## Étape 5 — Règle de cycle de vie

Créer `cycle.json` :

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "sauvegardes-vieillissantes",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["pve01/"] },
        "actions": {
          "baseBlob": {
            "tierToCool":  { "daysAfterModificationGreaterThan": 30 },
            "tierToCold":  { "daysAfterModificationGreaterThan": 90 },
            "delete":      { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

```bash
az storage account management-policy create -g $RG --account-name $SA --policy @cycle.json
```

Le moteur passe environ une fois par jour : les changements de niveau ne sont pas immédiats. Pour tester tout de suite un changement manuel :

```bash
az storage blob set-tier --account-name $SA -c pve01 -n "$BLOB" --tier Cool --auth-mode login
```

## Étape 6 — Protéger contre la suppression

Activer la suppression réversible et le versionnage :

```bash
az storage account blob-service-properties update -g $RG --account-name $SA \
  --enable-delete-retention true --delete-retention-days 7 --enable-versioning true
```

Puis une **règle d'immuabilité** de 7 jours sur le conteneur, **sans la verrouiller** (sur un lab, une règle verrouillée empêcherait de supprimer le compte) :

```bash
az storage container immutability-policy create -g $RG --account-name $SA \
  -c pve01 --period 7
```

Essayer de supprimer un blob :

```bash
az storage blob delete --account-name $SA -c pve01 -n "$BLOB" --auth-mode login
# → This operation is not permitted as the blob is immutable due to a policy.
```

## Ce que tu vois / ce que ça veut dire / action

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `AuthorizationPermissionMismatch` | Pas de rôle « … Data … » sur le compte | Étape 2, puis attendre la propagation |
| `KeyBasedAuthenticationNotPermitted` | Les clés du compte sont désactivées et la commande les utilise | Ajouter `--auth-mode login` |
| `The specified account name is already taken` | Nom déjà pris quelque part dans le monde | Changer le suffixe numérique |
| `AuthenticationFailed` sur un lien SAS | Lien expiré, tronqué ou clés de délégation révoquées | Régénérer un SAS |
| `This operation is not permitted as the blob is immutable` | La règle d'immuabilité fait son travail | Attendre la fin de la période, ou supprimer la règle tant qu'elle n'est pas verrouillée |

## Vérification finale

- [ ] La sauvegarde est listée dans le conteneur `pve01`, avec sa taille
- [ ] Le lien SAS télécharge le fichier, puis cesse de fonctionner après expiration
- [ ] `az storage account management-policy show` affiche la règle de cycle de vie
- [ ] La suppression d'un blob est refusée tant que la règle d'immuabilité court

## Nettoyage

```bash
az storage container immutability-policy delete -g $RG --account-name $SA -c pve01 \
  --if-match $(az storage container immutability-policy show -g $RG --account-name $SA -c pve01 --query etag -o tsv)
az group delete -n rg-horssite --yes --no-wait
```

Si des blobs sont encore sous immuabilité, le compte ne pourra être supprimé qu'à la fin de la période.

## Dépannage courant

- **`azcopy login` refusé depuis `pve01`** : utiliser `--login-type=device` ; si le tenant de l'école le bloque, générer depuis ton PC un SAS de conteneur en écriture (`--permissions acw`, quelques heures) et l'ajouter à l'URL de destination.
- **Transfert très lent** : `azcopy` utilise beaucoup de connexions parallèles ; sur une petite connexion, limiter avec `--cap-mbps 50`.
- **Coût surprise** : vérifier dans *Cost analysis* le poste « Data Transfer » : rapatrier les données (egress) est facturé, pas leur envoi.

## Pour aller plus loin

- 🧠 [Le stockage dans Azure](concept-stockage-azure.md) : redondance, niveaux, accès privé
- 🧠 [Sauvegardes](../transverse/concept-sauvegardes.md) : où cette copie s'inscrit dans la règle 3-2-1
- 🛠️ [Déployer une infrastructure Azure avec Terraform](tuto-terraform-azure.md) : créer ce compte en code
