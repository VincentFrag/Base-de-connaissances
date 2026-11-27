---
icon: material/flask-outline
title: "TP Azure : Stockage (TP11 et TP12)"
tags: [cloud, azure, tp, stockage, blob, sas, immuabilité, private endpoint, service endpoint]
---

# 🧪 TP Azure : Stockage (TP11 et TP12)

!!! abstract "Ce que tu vas faire"
    Compte de stockage, SAS, cycle de vie et immuabilité, puis accès privé par Service Endpoint et Private Endpoint.

**Niveau :** 🟡 Intermédiaire · **Durée :** 3 h · **Outil :** Azure for Students

Fais TP11 et TP12 dans le même groupe `rg-stockage` : le TP12 réutilise le compte du TP11.

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP11 — Compte de stockage, SAS et immuabilité

**Objectif** : l'exercice de la p. 93, plus l'équivalent Azure du bucket immuable des p. 127-129. **Slides du cours** p. 85-100, 127-129. **Coût** < 0,10 $.

1. **Step 1** — crée un compte StorageV2 : redondance LRS (justifie ce choix), accès anonyme aux blobs désactivé, réplication inter-tenant désactivée, TLS 1.2 minimum, accès réseau limité à ton IP publique (CIDR /32), versioning et suppression réversible activés.
2. **Step 2** — crée un conteneur `docs`. Essaie d'abord en mode `--auth-mode login` : refusé alors que tu es Owner. Donne-toi le rôle **Storage Blob Data Contributor** et recommence (attends 2 à 5 min).
3. Dépose un fichier, écrase-le deux fois, liste ses versions.
4. **Step 3** — génère un SAS en lecture seule, valable 2 h, HTTPS uniquement, de type *user delegation* (signé par ton identité, pas par la clé du compte). Télécharge le fichier avec `curl` et l'URL complète.
5. Ouvre la même URL depuis ton téléphone en 4G : refusé. Le SAS ne contourne pas le pare-feu réseau.
6. Change le tier d'un blob en Cool puis en Archive, et essaie de le lire. Ajoute une règle de cycle de vie « Cool après 30 jours, suppression après 365 ».
7. **Immuabilité** : sur un conteneur `archive`, pose une stratégie de rétention temporelle de 1 jour **non verrouillée**. Tente de supprimer un blob.

**Validation** : `curl` renvoie le contenu avec le SAS ; 403 depuis la 4G ; suppression refusée dans `archive`.

**Attention** : ne **verrouille** jamais une stratégie d'immuabilité dans un lab. Verrouillée, elle ne peut plus être raccourcie ni supprimée, et elle empêche de supprimer le compte jusqu'à expiration.

**Questions** : fais la correspondance avec les slides 128-129 — bucket immuable, mode Gouvernance, mode Compliance — et leurs équivalents Azure (stratégie non verrouillée, verrouillée, legal hold). Pourquoi un SAS *user delegation* est-il préférable à un SAS signé par clé de compte ? Quel type de blob (bloc, ajout, page) pour des logs, un disque de VM, une photo (p. 86) ?

**Nettoyage** : garde `rg-stockage` pour le TP12.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-stockage; MYIP=<ton IP>; SA=stlab$RANDOM
    az group create -n $RG -l $LOC --tags tp=11 owner=admin

    az storage account create -g $RG -n $SA -l $LOC --sku Standard_LRS --kind StorageV2 \
      --allow-blob-public-access false --allow-cross-tenant-replication false \
      --min-tls-version TLS1_2 --https-only true --default-action Deny
    az storage account network-rule add -g $RG --account-name $SA --ip-address $MYIP
    az storage account blob-service-properties update -g $RG --account-name $SA \
      --enable-versioning true --enable-delete-retention true --delete-retention-days 7
    echo "Nom du compte, à garder pour le TP12 : $SA"

    # Owner ne suffit pas pour le plan de données
    az storage container create --account-name $SA -n docs --auth-mode login     # 403
    ME=$(az ad signed-in-user show --query id -o tsv)
    SAID=$(az storage account show -g $RG -n $SA --query id -o tsv)
    az role assignment create --assignee $ME --role "Storage Blob Data Contributor" --scope $SAID
    sleep 180
    az storage container create --account-name $SA -n docs --auth-mode login

    for v in 1 2 3; do
      echo "version $v" > hello.txt
      az storage blob upload --account-name $SA -c docs -n hello.txt -f hello.txt --overwrite --auth-mode login
    done
    az storage blob list --account-name $SA -c docs --include v --auth-mode login \
      --query "[].{nom:name, version:versionId, courante:isCurrentVersion}" -o table

    # SAS user delegation : signé par ton identité, 2 h, lecture, HTTPS
    END=$(date -u -d "+2 hours" '+%Y-%m-%dT%H:%MZ')
    SAS=$(az storage blob generate-sas --account-name $SA -c docs -n hello.txt --permissions r \
      --expiry $END --https-only --auth-mode login --as-user -o tsv)
    URL="https://$SA.blob.core.windows.net/docs/hello.txt?$SAS"
    curl "$URL"; echo; echo "$URL"      # ouvre aussi cette URL en 4G : 403

    # Tiers
    az storage blob set-tier --account-name $SA -c docs -n hello.txt --tier Cool --auth-mode login
    az storage blob set-tier --account-name $SA -c docs -n hello.txt --tier Archive --auth-mode login
    curl "$URL"                         # 409 : blob archivé, il faut le réhydrater (plusieurs heures)
    ```

    Règle de cycle de vie `policy.json`, appliquée avec `az storage account management-policy create -g $RG --account-name $SA --policy @policy.json` :

    ```json
    {
      "rules": [{
        "enabled": true,
        "name": "cool-puis-suppression",
        "type": "Lifecycle",
        "definition": {
          "filters": { "blobTypes": ["blockBlob"] },
          "actions": { "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "delete":     { "daysAfterModificationGreaterThan": 365 }
          } }
        }
      }]
    }
    ```

    Immuabilité : fais-la dans un **second compte sans versioning**, pour éviter les interactions entre versioning et stratégie au niveau conteneur.

    ```bash
    ARC=starchive$RANDOM
    az storage account create -g $RG -n $ARC -l $LOC --sku Standard_LRS --kind StorageV2 \
      --allow-blob-public-access false --default-action Deny
    az storage account network-rule add -g $RG --account-name $ARC --ip-address $MYIP
    az role assignment create --assignee $ME --role "Storage Blob Data Contributor" \
      --scope $(az storage account show -g $RG -n $ARC --query id -o tsv)
    sleep 180
    az storage container create --account-name $ARC -n archive --auth-mode login
    az storage container immutability-policy create -g $RG --account-name $ARC -c archive --period 1   # NE PAS faire "lock"
    az storage blob upload --account-name $ARC -c archive -n preuve.txt -f hello.txt --auth-mode login
    az storage blob delete --account-name $ARC -c archive -n preuve.txt --auth-mode login   # refusé

    # Au nettoyage, retirer la stratégie non verrouillée
    ETAG=$(az storage container immutability-policy show -g $RG --account-name $ARC -c archive --query etag -o tsv)
    az storage container immutability-policy delete -g $RG --account-name $ARC -c archive --if-match $ETAG
    ```

    Réponses :

    | Slides (S3) | Azure Blob | Qui peut revenir en arrière ? |
    | --- | --- | --- |
    | Bucket immuable (`--object-lock-enabled-for-bucket`) | Immuabilité au niveau conteneur ou version | — |
    | Mode Gouvernance | Stratégie de rétention temporelle **non verrouillée** | Un administrateur qui a le droit de modifier ou supprimer la stratégie |
    | Mode Compliance | Stratégie **verrouillée** | Personne, pas même le support Microsoft, jusqu'à expiration |
    | Legal hold | Conservation légale (legal hold) | Retrait explicite du hold |

    Un SAS *user delegation* est signé par une clé liée à ton identité Entra ID : il hérite de tes droits RBAC, dure au plus 7 jours, apparaît dans les journaux sous ton nom, et se révoque en révoquant les clés de délégation. Un SAS signé par clé de compte donne potentiellement tout, et ne se révoque qu'en changeant la clé — ce qui casse toutes les applications qui l'utilisent. Logs → blob d'ajout ; disque de VM → blob de pages ; photo → blob de blocs. LRS suffit pour un lab : trois copies dans un seul datacenter ; en production, ZRS ou GRS selon le sinistre que tu veux couvrir.

## TP12 — Service Endpoint vs Private Endpoint

**Objectif** : réaliser le schéma des p. 48 et 101 — « atteindre le compte depuis l'IP du PE » — et comprendre ce que chaque mécanisme change au DNS et au routage. **Slides du cours** p. 48, 70, 101. **Coût** < 0,50 $.

1. Crée `vnet1` (10.50.0.0/16) et `vnet2` (10.51.0.0/16) appairés, une petite VM dans chacun (IP publique restreinte à ton IP pour SSH).
2. **Service Endpoint** : active `Microsoft.Storage` sur le sous-réseau de `vnet1` et autorise ce sous-réseau dans le pare-feu du compte. Depuis chaque VM : `nslookup <compte>.blob.core.windows.net` et `curl` avec un SAS.
3. **Private Endpoint** : crée un PE pour le sous-service `blob` dans un sous-réseau dédié de `vnet1`, la zone DNS privée `privatelink.blob.core.windows.net` liée aux **deux** VNet, puis désactive l'accès public du compte.
4. Refais `nslookup` et `curl` depuis les deux VM et depuis ton PC.

**Validation** : avec le PE, le nom du compte résout en 10.50.x.x depuis les deux VM et le `curl` fonctionne depuis `vnet2` ; depuis ton PC, accès refusé.

**Questions** : remplis un tableau Service Endpoint / Private Endpoint : IP utilisée, résolution DNS, fonctionne à travers un appairage ou un VPN, coût, protection contre l'exfiltration. Pourquoi faut-il lier la zone DNS privée à `vnet2` ?

**Piège** : génère ton SAS *avant* de couper l'accès public, car la clé de délégation s'obtient par l'endpoint public.

**Nettoyage** : `az group delete -n rg-stockage`.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-stockage; MYIP=<ton IP>; SA=<nom du compte du TP11>
    az network vnet create -g $RG -n vnet1 --address-prefixes 10.50.0.0/16 --subnet-name snet-vm --subnet-prefixes 10.50.1.0/24
    az network vnet subnet create -g $RG --vnet-name vnet1 -n snet-pe --address-prefixes 10.50.2.0/24
    az network vnet create -g $RG -n vnet2 --address-prefixes 10.51.0.0/16 --subnet-name snet-vm --subnet-prefixes 10.51.1.0/24
    az network vnet peering create -g $RG -n v1-to-v2 --vnet-name vnet1 --remote-vnet vnet2 --allow-vnet-access
    az network vnet peering create -g $RG -n v2-to-v1 --vnet-name vnet2 --remote-vnet vnet1 --allow-vnet-access

    for n in 1 2; do
      az vm create -g $RG -n vm$n --image Ubuntu2404 --size Standard_B1s --vnet-name vnet$n --subnet snet-vm \
        --public-ip-sku Standard --nsg-rule NONE --admin-username azureuser --generate-ssh-keys
      az network nsg rule create -g $RG --nsg-name vm${n}NSG -n ssh-mon-ip --priority 100 \
        --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow
    done

    # SAS généré AVANT de couper l'accès public (1 jour)
    END=$(date -u -d "+1 day" '+%Y-%m-%dT%H:%MZ')
    SAS=$(az storage blob generate-sas --account-name $SA -c docs -n hello.txt --permissions r \
      --expiry $END --https-only --auth-mode login --as-user -o tsv)
    echo "https://$SA.blob.core.windows.net/docs/hello.txt?$SAS"     # copie cette URL
    ```

    Remets `hello.txt` en tier Hot si tu l'as archivé au TP11 (ou dépose un nouveau fichier). Test, à lancer sur `vm1` puis `vm2` :

    ```bash
    nslookup <compte>.blob.core.windows.net
    curl -s -o /dev/null -w "%{http_code}\n" "<URL avec SAS>"
    ```

    **Partie A — Service Endpoint**

    ```bash
    az network vnet subnet update -g $RG --vnet-name vnet1 -n snet-vm --service-endpoints Microsoft.Storage
    az storage account network-rule add -g $RG --account-name $SA --vnet-name vnet1 --subnet snet-vm
    ```

    Attendu : résolution vers une IP **publique** partout ; 200 depuis `vm1`, 403 depuis `vm2`.

    **Partie B — Private Endpoint**

    ```bash
    SAID=$(az storage account show -g $RG -n $SA --query id -o tsv)
    az network private-endpoint create -g $RG -n pe-blob --vnet-name vnet1 --subnet snet-pe \
      --private-connection-resource-id $SAID --group-id blob --connection-name pe-blob-conn
    az network private-dns zone create -g $RG -n privatelink.blob.core.windows.net
    az network private-dns link vnet create -g $RG -z privatelink.blob.core.windows.net -n link-vnet1 -v vnet1 -e false
    az network private-dns link vnet create -g $RG -z privatelink.blob.core.windows.net -n link-vnet2 -v vnet2 -e false
    az network private-endpoint dns-zone-group create -g $RG --endpoint-name pe-blob -n default \
      --private-dns-zone privatelink.blob.core.windows.net --zone-name blob
    az storage account update -g $RG -n $SA --public-network-access Disabled
    ```

    Attendu : `<compte>.blob.core.windows.net` → CNAME `<compte>.privatelink.blob.core.windows.net` → 10.50.2.4 depuis les deux VM ; 200 depuis `vm1` et `vm2` ; depuis ton PC, 403.

    |  | Service Endpoint | Private Endpoint |
    | --- | --- | --- |
    | IP jointe | IP publique du service, par le réseau Azure | IP privée dans ton VNet |
    | DNS | inchangé | zone privée `privatelink` |
    | Depuis un VNet appairé, un VPN, l'on-premise | non | oui |
    | Coût | gratuit | à l'heure + au Go traité |
    | Exfiltration | le sous-réseau atteint *tous* les comptes du service | lié à un seul compte |

    Sans lien vers `vnet2`, la zone privée est invisible pour ce VNet : `vm2` résout le nom vers l'IP publique, que l'accès public désactivé refuse. C'est le piège DNS numéro un des Private Endpoints, surtout en hybride (il faut alors un résolveur DNS privé pour l'on-premise).

    Nettoyage : `az group delete -n rg-stockage --yes` (après avoir retiré la stratégie d'immuabilité).
