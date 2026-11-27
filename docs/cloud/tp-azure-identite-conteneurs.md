---
icon: material/flask-outline
title: "TP Azure : Identité et conteneurs (TP6 et TP7)"
tags: [cloud, azure, tp, rbac, identité managée, entra id, keycloak, docker, container instances]
---

# 🧪 TP Azure : Identité et conteneurs (TP6 et TP7)

!!! abstract "Ce que tu vas faire"
    Rôles RBAC prouvés par l'expérience avec une identité managée, puis Keycloak sur VM Docker et en conteneur managé.

**Niveau :** 🟡 Intermédiaire · **Durée :** 2 h 30 · **Outil :** Azure for Students

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP6 — RBAC et identité managée (Exo IAM 1)

**Objectif** : prouver par l'expérience ce qu'un rôle autorise et refuse, à quelle portée. **Slides du cours** p. 46, 51-56. **Coût** < 0,10 $.

Le cours fait cet exercice en binôme (se donner mutuellement le rôle Lecteur). Seul, tu remplaces ton binôme par une **identité managée** : un compte de service Entra ID que tu peux créer sans droits d'administration sur le tenant.

1. Suis le module Learn cité p. 56 (« Secure Azure resources with RBAC ») pour le vocabulaire : principal, rôle, portée, attribution.
2. Dans `rg-tp06`, crée une identité managée affectée par l'utilisateur `id-lecteur` et donne-lui le rôle **Reader** sur ce seul groupe.
3. Crée une VM B1s à laquelle tu attaches `id-lecteur`. Installe Azure CLI dessus et connecte-toi avec `az login --identity`.
4. Depuis la VM, liste les ressources du groupe, puis essaie de créer un groupe de ressources et d'ajouter un tag à la VM.
5. Remplace Reader par **Virtual Machine Contributor** et refais l'essai.
6. Bonus : écris un rôle personnalisé « Opérateur VM » qui permet seulement `start`, `restart` et `deallocate`.
7. Si ton école autorise les invités, fais aussi la version binôme : invite un camarade et donne-lui Reader sur un groupe.

**Validation** : avec Reader, la lecture marche et la création renvoie `AuthorizationFailed` ; avec VM Contributor, la modification de la VM passe mais la création de groupe reste refusée.

**Questions** : différence entre identité managée système et utilisateur ? Pourquoi être Owner de l'abonnement ne te donne-t-il pas accès aux données d'un blob (tu le vérifieras au TP11) ? Donne un exemple d'ABAC tiré de ton travail (p. 56 : poste d'entreprise vs personnel, horaires).

**Nettoyage** : `az group delete -n rg-tp06`, puis `az role definition delete` pour le rôle personnalisé.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-tp06
    az group create -n $RG -l $LOC --tags tp=06 owner=admin

    az identity create -g $RG -n id-lecteur
    PRINCIPAL=$(az identity show -g $RG -n id-lecteur --query principalId -o tsv)
    CLIENTID=$(az identity show -g $RG -n id-lecteur --query clientId -o tsv)
    IDID=$(az identity show -g $RG -n id-lecteur --query id -o tsv)
    RGID=$(az group show -n $RG --query id -o tsv)

    az role assignment create --assignee-object-id $PRINCIPAL \
      --assignee-principal-type ServicePrincipal --role Reader --scope $RGID

    az vm create -g $RG -n vm-rbac --image Ubuntu2404 --size Standard_B1s \
      --assign-identity $IDID --public-ip-sku Standard \
      --admin-username azureuser --generate-ssh-keys
    echo "clientId à utiliser sur la VM : $CLIENTID"
    ```

    Sur la VM :

    ```bash
    curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
    az login --identity --client-id <CLIENTID>     # versions anciennes de la CLI : --username <CLIENTID>

    az resource list -g rg-tp06 -o table                                   # OK
    az group create -n rg-interdit -l francecentral --tags tp=x            # AuthorizationFailed
    az vm update -g rg-tp06 -n vm-rbac --set tags.rbac=ok                  # AuthorizationFailed
    ```

    Pour tester une modification, on change un tag de la VM plutôt que de la redémarrer : la redémarrer depuis elle-même couperait ta session. Bascule ensuite de rôle (depuis ton PC), attends 2 à 5 min, refais `az login --identity` sur la VM :

    ```bash
    az role assignment delete --assignee $PRINCIPAL --role Reader --scope $RGID
    az role assignment create --assignee-object-id $PRINCIPAL \
      --assignee-principal-type ServicePrincipal --role "Virtual Machine Contributor" --scope $RGID
    ```

    Rôle personnalisé `operateur-vm.json` :

    ```json
    {
      "Name": "Operateur VM (lab)",
      "Description": "Démarrer, redémarrer et désallouer des VM, rien d'autre",
      "Actions": [
        "Microsoft.Compute/virtualMachines/read",
        "Microsoft.Compute/virtualMachines/start/action",
        "Microsoft.Compute/virtualMachines/restart/action",
        "Microsoft.Compute/virtualMachines/deallocate/action"
      ],
      "NotActions": [],
      "AssignableScopes": ["/subscriptions/<SUBSCRIPTION_ID>"]
    }
    ```

    ```bash
    az role definition create --role-definition @operateur-vm.json
    az role definition delete --name "Operateur VM (lab)"     # au nettoyage
    ```

    Réponses : une identité **système** naît et meurt avec la ressource ; une identité **utilisateur** est une ressource à part, réutilisable par plusieurs VM ou services et qui survit à leur suppression. Owner donne les *Actions* du plan de gestion (créer le compte, lire ses clés) mais pas les *DataActions* du plan de données (lire un blob avec ton identité) : il faut un rôle « Storage Blob Data … ». ABAC : une condition sur l'attribution de rôle, par exemple « lecture des blobs seulement si leur tag `projet` vaut `tp11` » ; côté Entra ID, l'accès conditionnel « poste conforme Intune ou joint hybride » que tu connais applique la même logique d'attributs.

## TP7 — Keycloak sur VM Docker puis en Container Instance (Exo IAM 2 + Exo container)

**Objectif** : déployer le même service d'identité de deux façons et comparer IaaS et conteneur managé. **Slides du cours** p. 56, 75, 134. **Coût** < 0,50 $.

1. **Version VM** : VM `Standard_B2s` (Keycloak a besoin de RAM), Docker installé, Keycloak lancé en mode `start-dev` avec les variables `KC_BOOTSTRAP_ADMIN_USERNAME` et `KC_BOOTSTRAP_ADMIN_PASSWORD` de la slide p. 75.
2. N'ouvre **pas** le port 8080 sur Internet : accède à la console par un tunnel SSH (`ssh -L 8080:localhost:8080 …`) puis `http://localhost:8080`.
3. Crée un realm `cesi`, deux utilisateurs, un groupe et un client OIDC confidentiel `postman` (il servira au TP13).
4. **Version conteneur** : supprime la VM et déploie la même image dans Azure Container Instances avec un nom DNS, comme sur la capture p. 75 (politique de redémarrage « On failure », mot de passe en variable sécurisée).
5. Ouvre `http://<nom>.francecentral.azurecontainer.io:8080`. Si Keycloak répond « HTTPS required », lis le corrigé.
6. Redémarre le conteneur : ton realm `cesi` existe-t-il encore ?

**Validation** : console admin accessible dans les deux cas ; le realm disparaît au redémarrage du conteneur.

**Questions** : pourquoi le realm disparaît-il ? Relie à « Pets vs Cattle » et aux composants stateful (p. 102-103) : que faudrait-il externaliser ? Pourquoi le mode `start-dev` est-il interdit en production ? Combien coûte chaque version à l'heure ?

**Nettoyage** : `az group delete -n rg-tp07`.

??? success "✅ Corrigé et commandes"

    **Version VM**

    ```bash
    LOC=francecentral; RG=rg-tp07
    az group create -n $RG -l $LOC --tags tp=07 owner=admin
    az vm create -g $RG -n vm-keycloak --image Ubuntu2404 --size Standard_B2s \
      --public-ip-sku Standard --admin-username azureuser --generate-ssh-keys
    IP=$(az vm show -d -g $RG -n vm-keycloak --query publicIps -o tsv)
    ssh azureuser@$IP
    ```

    Sur la VM (le port est publié sur 127.0.0.1 seulement : inaccessible depuis Internet) :

    ```bash
    sudo apt-get update && sudo apt-get install -y docker.io
    sudo docker run -d --name keycloak --restart unless-stopped -p 127.0.0.1:8080:8080 \
      -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD='<mot de passe fort>' \
      quay.io/keycloak/keycloak:latest start-dev
    sudo docker logs -f keycloak       # attendre "Running the server in development mode"
    ```

    Depuis ton PC : `ssh -N -L 8080:localhost:8080 azureuser@$IP`, puis `http://localhost:8080`. Client `postman` : Clients → Create client → Client authentication **On**, Service accounts roles **On** ; le secret est dans l'onglet Credentials.

    **Version Container Instances**

    ```bash
    az container create -g $RG -n aci-keycloak --image quay.io/keycloak/keycloak:latest \
      --os-type Linux --cpu 1 --memory 2 --ports 8080 --ip-address Public \
      --dns-name-label kc-lab-$RANDOM --restart-policy OnFailure \
      --environment-variables KC_BOOTSTRAP_ADMIN_USERNAME=admin \
      --secure-environment-variables KC_BOOTSTRAP_ADMIN_PASSWORD='<mot de passe fort>' \
      --command-line "/opt/keycloak/bin/kc.sh start-dev"
    az container show -g $RG -n aci-keycloak --query ipAddress.fqdn -o tsv
    az container logs -g $RG -n aci-keycloak
    ```

    « HTTPS required » : le realm `master` exige HTTPS pour toute IP cliente non privée. Contournement **de lab uniquement** (mot de passe admin en clair sur Internet sinon) :

    ```bash
    az container exec -g $RG -n aci-keycloak --exec-command "/bin/bash"
    # dans le conteneur
    /opt/keycloak/bin/kcadm.sh config credentials --server http://localhost:8080 \
      --realm master --user admin --password '<mot de passe>'
    /opt/keycloak/bin/kcadm.sh update realms/master -s sslRequired=NONE
    ```

    La commande de la slide (`start --hostname=localhost --http-enabled=true`) lance le mode production avec un nom d'hôte `localhost` : elle ne règle pas ce problème pour un accès distant. La bonne solution est un proxy TLS devant Keycloak (bonus : un conteneur Caddy dans le même groupe de conteneurs).

    Réponses : en `start-dev`, Keycloak stocke tout dans une base H2 locale au conteneur ; `az container restart` recrée le conteneur et efface le realm. Keycloak est donc un composant stateful déguisé en « cattle » : il faut externaliser sa base (PostgreSQL managé) pour pouvoir le détruire et le recréer librement. `start-dev` désactive le cache distribué et HTTPS et utilise H2 : interdit en production. Coût : quelques centimes de l'heure dans les deux cas ; Container Instances est facturé à la seconde et rien quand le groupe est supprimé, sans OS à maintenir.
