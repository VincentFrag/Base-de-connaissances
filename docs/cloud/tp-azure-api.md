---
icon: material/flask-outline
title: "TP Azure : API (TP13 et TP14)"
tags: [cloud, azure, tp, api rest, postman, oauth2, aws cli, s3, object lock]
---

# 🧪 TP Azure : API (TP13 et TP14)

!!! abstract "Ce que tu vas faire"
    Piloter Azure par son API REST avec Postman, puis AWS S3 et Object Lock en ligne de commande.

**Niveau :** 🟡 Intermédiaire · **Durée :** 2 h 30 · **Outil :** Azure for Students

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP13 — Postman et l'API REST Azure (Exercice POSTMAN)

**Objectif** : piloter Azure sans portail ni CLI, uniquement par l'API REST d'Azure Resource Manager, avec une collection Postman propre. **Slides du cours** p. 110, 136. **Coût** 0 $ (quelques centimes si tu crées un compte de stockage).

Le cours utilise l'API de la plateforme DENV-R (p. 136), inaccessible hors formation ; l'API Azure Resource Manager joue exactement le même rôle.

1. Crée un environnement Postman `azure-lab` avec les variables `tenantId`, `subscriptionId`, `rg`, `location` et `token` (type *secret*).
2. Récupère un jeton d'accès avec `az account get-access-token` et colle-le dans `token`. Note sa durée de vie.
3. Crée une collection « Azure ARM », authentification *Bearer* `{{token}}` définie au niveau de la collection, avec un dossier par ressource :
   - GET liste des groupes de ressources ;
   - PUT création de `rg-tp13` avec des tags ;
   - PUT création d'un compte de stockage, puis suivi de l'opération asynchrone (réponse 202 et en-tête `Azure-AsyncOperation` ou `Location`) ;
   - POST `listKeys` sur ce compte, puis `regenerateKey` ;
   - DELETE du groupe.
4. Ajoute un test Postman sur chaque requête (code attendu) et lance toute la collection avec le Runner.
5. **Gestion des clés d'API** : remplace le jeton collé à la main par une authentification OAuth 2.0 *client credentials* (principal de service ou client Keycloak `postman` du TP7). Si ton école bloque la création de principaux de service, fais cette étape avec Keycloak.
6. Exporte la collection (sans secrets) et range-la dans un dépôt Git.

**Validation** : le Runner passe tous les tests ; aucun secret n'apparaît dans l'export.

**Questions** : différence entre 401 et 403 (provoque les deux : jeton expiré, identité Reader) ? Pourquoi `listKeys` est-il un POST et pas un GET ? Que se passe-t-il pour les applications qui utilisaient l'ancienne clé après `regenerateKey`, et comment fait-on une rotation sans coupure (deux clés) ?

**Nettoyage** : la dernière requête de la collection supprime le groupe ; vérifie-le.

??? success "✅ Corrigé et commandes"

    Valeurs de l'environnement Postman :

    ```bash
    az account show --query "{tenantId:tenantId, subscriptionId:id}" -o json
    az account get-access-token --resource https://management.azure.com/ --query accessToken -o tsv
    ```

    | Requête | Méthode et URL | Corps |
    | --- | --- | --- |
    | Lister les groupes | `GET https://management.azure.com/subscriptions/{{subscriptionId}}/resourcegroups?api-version=2021-04-01` | — |
    | Créer le groupe | `PUT …/subscriptions/{{subscriptionId}}/resourcegroups/{{rg}}?api-version=2021-04-01` | `{"location":"{{location}}","tags":{"tp":"13"}}` |
    | Créer un compte | `PUT …/resourceGroups/{{rg}}/providers/Microsoft.Storage/storageAccounts/{{sa}}?api-version=2023-05-01` | `{"location":"{{location}}","sku":{"name":"Standard_LRS"},"kind":"StorageV2","properties":{"allowBlobPublicAccess":false}}` |
    | Suivre l'opération | `GET {{asyncUrl}}` | — |
    | Lister les clés | `POST …/storageAccounts/{{sa}}/listKeys?api-version=2023-05-01` | — |
    | Régénérer key1 | `POST …/storageAccounts/{{sa}}/regenerateKey?api-version=2023-05-01` | `{"keyName":"key1"}` |
    | Supprimer le groupe | `DELETE …/resourcegroups/{{rg}}?api-version=2021-04-01` | — |

    Onglet *Scripts → Post-response* de la création du compte (le nom `{{sa}}` doit être unique, en minuscules) :

    ```javascript
    pm.test("accepté", () => pm.expect(pm.response.code).to.be.oneOf([200, 202]));
    const loc = pm.response.headers.get("Azure-AsyncOperation") || pm.response.headers.get("Location");
    if (loc) pm.collectionVariables.set("asyncUrl", loc);
    ```

    **OAuth 2.0 client credentials vers Azure** (si ton école autorise la création de principaux de service) :

    ```bash
    az ad sp create-for-rbac --name sp-postman-lab --role Contributor \
      --scopes /subscriptions/<SUB>/resourceGroups/rg-tp13      # renvoie appId, password, tenant
    ```

    Dans Postman, Authorization de la collection → OAuth 2.0 → Grant type *Client Credentials* ; Access Token URL `https://login.microsoftonline.com/{{tenantId}}/oauth2/v2.0/token` ; Client ID = `appId`, Client Secret = `password` (variable secrète) ; Scope `https://management.azure.com/.default`.

    **Variante Keycloak (TP7)** : Access Token URL `http://localhost:8080/realms/cesi/protocol/openid-connect/token`, client `postman` et son secret. Donne au compte de service du client le rôle `view-users` du client `realm-management`, puis appelle `GET http://localhost:8080/admin/realms/cesi/users` avec ce jeton.

    Réponses : **401** = pas authentifié (jeton absent, expiré ou mal formé) ; **403** = authentifié mais pas autorisé (l'identité Reader du TP6). `listKeys` est un POST parce que c'est une *action* qui renvoie des secrets : elle a sa propre permission RBAC (`Microsoft.Storage/storageAccounts/listkeys/action`), qu'un Reader n'a pas, alors qu'un GET doit rester une lecture sans effet. Après `regenerateKey`, tout ce qui utilisait key1 tombe en 403. Rotation sans coupure : basculer les applications sur key2, régénérer key1, rebasculer sur key1, régénérer key2 — ou mieux, supprimer les clés au profit d'Entra ID et d'identités managées.

## TP14 — AWS CLI et S3 Object Lock (Exercice AWS CLI)

**Objectif** : refaire les commandes `s3api` des p. 127-129 sur un vrai S3, et voir la différence entre un bucket classique et un bucket immuable. **Coût** 0 $ avec l'offre gratuite d'AWS (quelques Ko stockés).

Le point d'accès `s3.denv-r.com` n'est pas joignable hors formation. Deux options : un compte AWS avec l'offre gratuite (carte bancaire demandée à l'inscription ; vérifie les conditions actuelles), ou un serveur compatible S3 en local qui gère Object Lock (Ceph RGW si ton homelab le permet). Évite MinIO : son édition communautaire n'est plus distribuée en binaires depuis fin 2025.

1. Crée un utilisateur IAM dédié avec une clé d'accès, jamais le compte root. `aws configure` avec la région `eu-west-3` (Paris).
2. `list-buckets`, puis crée un bucket classique et un bucket avec `--object-lock-enabled-for-bucket`.
3. Sur le bucket immuable, pose une rétention par défaut en mode **GOVERNANCE**, 1 jour.
4. Dépose un objet, note son `VersionId`, essaie de supprimer cette version, puis recommence avec `--bypass-governance-retention`.
5. Lis la configuration avec `get-object-lock-configuration` (la commande de la p. 129).

**Validation** : la suppression simple est refusée (`AccessDenied`), la suppression avec contournement réussit.

**Questions** : pourquoi un schéma « Veeam → S3 immuable » (p. 127) protège-t-il contre un rançongiciel qui a volé les identifiants d'administration ? Pourquoi ne jamais tester le mode **COMPLIANCE** sur un compte personnel ? À quoi sert `--endpoint-url` ?

**Bonus AWS (Hands-on p. 121)** : avec le même compte, crée un VPC, une Internet Gateway, une table de routage publique et un groupe de sécurité, et compare chaque objet à son équivalent Azure.

**Nettoyage** : vide et supprime les buckets (la rétention de 1 jour expire seule), supprime la clé d'accès IAM.

??? success "✅ Corrigé et commandes"

    Crée l'utilisateur IAM `cli-lab` (politique `AmazonS3FullAccess` pour le lab), puis une clé d'accès « Command Line Interface ». Mieux encore si tu as le temps : IAM Identity Center et `aws configure sso`, sans clé permanente.

    ```bash
    aws configure                       # clé, secret, région eu-west-3, sortie json
    aws sts get-caller-identity
    aws s3api list-buckets

    B=cesi-lab-$RANDOM
    aws s3api create-bucket --bucket $B-classique --create-bucket-configuration LocationConstraint=eu-west-3
    aws s3api create-bucket --bucket $B-lock --create-bucket-configuration LocationConstraint=eu-west-3 \
      --object-lock-enabled-for-bucket
    aws s3api put-object-lock-configuration --bucket $B-lock --object-lock-configuration \
      '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"GOVERNANCE","Days":1}}}'
    aws s3api get-object-lock-configuration --bucket $B-lock

    echo test > f.txt
    VID=$(aws s3api put-object --bucket $B-lock --key f.txt --body f.txt --query VersionId --output text)

    aws s3api delete-object --bucket $B-lock --key f.txt                     # "réussit" : pose un delete marker
    aws s3api list-object-versions --bucket $B-lock                          # la version est toujours là
    aws s3api delete-object --bucket $B-lock --key f.txt --version-id $VID   # AccessDenied
    aws s3api delete-object --bucket $B-lock --key f.txt --version-id $VID --bypass-governance-retention   # OK
    ```

    Nettoyage :

    ```bash
    aws s3 rb s3://$B-classique --force
    for q in Versions DeleteMarkers; do
      aws s3api list-object-versions --bucket $B-lock --query "$q[].[Key,VersionId]" --output text |
      while read -r k v; do
        [ "$k" = "None" ] && continue
        aws s3api delete-object --bucket $B-lock --key "$k" --version-id "$v" --bypass-governance-retention
      done
    done
    aws s3api delete-bucket --bucket $B-lock
    ```

    Réponses : un rançongiciel qui a volé les identifiants du serveur de sauvegarde peut supprimer « l'objet » (delete marker), mais pas ses versions verrouillées. En mode Gouvernance, une identité dotée de `s3:BypassGovernanceRetention` peut encore tout effacer : pour une vraie protection, mode Compliance ou compte de sauvegarde séparé. Jamais Compliance sur un compte personnel : une faute de frappe (3650 jours au lieu de 1) et personne, pas même root, ne peut supprimer les objets avant l'échéance — tu paies le stockage jusque-là. `--endpoint-url` envoie les mêmes commandes à n'importe quel stockage compatible S3 (DENV-R, Ceph RGW, OVHcloud, Scaleway…).

    Bonus AWS, correspondance avec Azure :

    | AWS | Azure | Différence à retenir |
    | --- | --- | --- |
    | VPC | VNet | — |
    | Subnet | Subnet | un subnet AWS vit dans une seule AZ ; un subnet Azure couvre toute la région |
    | Internet Gateway + route 0.0.0.0/0 | IP publique / NAT Gateway / Load Balancer | AWS exige une IGW explicite et une route |
    | Route table | Route table (UDR) | — |
    | Security Group | NSG (sur carte) / ASG | SG : autorisations seulement, attaché à l'interface |
    | Network ACL | — | sans état, au niveau subnet ; le NSG Azure, lui, est à états |
