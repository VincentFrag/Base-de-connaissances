---
icon: material/flask-outline
title: "TP Azure : Fondations (TP1 à TP3)"
tags: [cloud, azure, tp, finops, budget, tags, azure policy, vm, cloud-init, vnet, nsg, network watcher]
---

# 🧪 TP Azure : Fondations (TP1 à TP3)

!!! abstract "Ce que tu vas faire"
    Budget et garde-fous FinOps, première VM amorcée par cloud-init, puis un VNet avec deux sous-réseaux protégés par des NSG.

**Niveau :** 🟢 Bases · **Durée :** 3 h · **Outil :** Azure for Students

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP1 — Budget, alertes et tags (FinOps)

**Objectif** : appliquer les piliers Informer / Optimiser / Opérer avant de dépenser le moindre centime. **Slides du cours** p. 39-42. **Coût** 0 $.

1. Dans Cost Management, crée un budget mensuel de 15 $ sur ton abonnement, avec des alertes à 50 %, 80 % et 100 % (réel) et 100 % (prévision), envoyées à ton e-mail. Si les budgets ne sont pas disponibles sur ton offre, suis le solde dans l'Azure Education Hub.
2. Définis ta convention de tags : `tp`, `owner`, `env`. Applique une Azure Policy « Require a tag on resource groups » pour le tag `tp`.
3. Essaie de créer un groupe de ressources sans tag : tu dois être bloqué.
4. Ouvre Cost analysis, regroupe par tag `tp`. Tu y reviendras après chaque TP.

**Validation** : un groupe sans tag est refusé ; le budget apparaît avec ses seuils.

**Questions** : quelle part de tes coûts est attribuable grâce aux tags (objectif du cours : plus de 90 %) ? Qu'est-ce qu'un arrêt automatique de VM apporte au pilier « Opérer » ?

**Piège** : Azure crée tout seul le groupe `NetworkWatcherRG` au premier VNet ; la policy le bloquerait. Crée-le toi-même avec un tag avant le TP3.

**Nettoyage** : rien à supprimer, garde le budget et la policy toute l'année.

??? success "✅ Corrigé et commandes"

    Le budget se crée plus simplement au portail : Cost Management → Budgets → Ajouter → portée = ton abonnement, montant 15 $, mensuel, alertes à 50/80/100 % en « Réel » et 100 % en « Prévu ». La policy, en CLI :

    ```bash
    LOC=francecentral
    # Groupe que Azure créerait seul au premier VNet : on le crée avec un tag pour ne pas être bloqué
    az group create -n NetworkWatcherRG -l $LOC --tags tp=infra owner=admin

    SUB=$(az account show --query id -o tsv)
    DEF=$(az policy definition list \
      --query "[?displayName=='Require a tag on resource groups'].name" -o tsv)
    az policy assignment create -n require-tp-tag --policy "$DEF" \
      --scope /subscriptions/$SUB --params '{"tagName":{"value":"tp"}}'

    # Après quelques minutes de propagation, ceci doit échouer (RequestDisallowedByPolicy)
    az group create -n rg-sans-tag -l $LOC
    ```

    Réponses : avec un tag `tp` obligatoire sur chaque groupe et des ressources toujours créées dans un groupe de TP, tu approches 100 % de coûts attribuables (Cost analysis → regrouper par tag). L'arrêt automatique (`az vm auto-shutdown`) est de l'automatisation « Opérer » : la règle s'applique sans qu'on ait à y penser.

## TP2 — Première VM et cloud-init

**Objectif** : maîtriser les choix de la création de VM (région, zone, image, taille, identifiants) et l'amorçage automatique. **Slides du cours** p. 57-59, 64-65. **Coût** < 0,20 $.

1. Au portail, crée `vm-web01` : Ubuntu 24.04, `Standard_B1s`, zone 1 de France Central, authentification par clé SSH, IP publique Standard.
2. Dans l'onglet Avancé, colle le cloud-init Apache de la slide p. 65 (corrigé : version qui affiche le nom de la VM).
3. Restreins le port 22 à ton IP et ouvre le port 80.
4. Configure l'arrêt automatique à 20 h.
5. Supprime tout, puis refais la même VM en une commande CLI.

**Validation** : `http://<IP publique>` affiche la page avec le nom de la VM ; `ssh azureuser@<IP>` fonctionne avec ta clé, pas par mot de passe.

**Questions** : différence entre région, zone de disponibilité, domaine d'erreur et domaine de mise à jour (p. 64) ? Que paies-tu encore quand la VM est arrêtée depuis Linux ? quand elle est désallouée ?

**Nettoyage** : `az group delete -n rg-tp02`.

??? success "✅ Corrigé et commandes"

    `cloud-init-apache.yaml` (la version de la p. 65, plus une page qui affiche le nom de la VM) :

    ```yaml
    #cloud-config
    package_update: true
    package_upgrade: true
    packages:
      - apache2
    runcmd:
      - echo "<h1>$(hostname) - Apache</h1>" > /var/www/html/index.html
      - systemctl enable apache2
      - systemctl restart apache2
    ```

    ```bash
    LOC=francecentral; RG=rg-tp02; MYIP=<ton IP publique, vue sur ifconfig.me depuis ton PC>
    az group create -n $RG -l $LOC --tags tp=02 owner=admin

    az vm create -g $RG -n vm-web01 --image Ubuntu2404 --size Standard_B1s --zone 1 \
      --admin-username azureuser --generate-ssh-keys --public-ip-sku Standard \
      --custom-data cloud-init-apache.yaml --nsg-rule NONE

    az network nsg rule create -g $RG --nsg-name vm-web01NSG -n ssh-mon-ip --priority 100 \
      --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow
    az network nsg rule create -g $RG --nsg-name vm-web01NSG -n http --priority 110 \
      --destination-port-ranges 80 --protocol Tcp --access Allow

    az vm auto-shutdown -g $RG -n vm-web01 --time 1800   # heure UTC

    IP=$(az vm show -d -g $RG -n vm-web01 --query publicIps -o tsv)
    curl http://$IP          # cloud-init peut prendre 3-5 min sur une B1s
    ssh azureuser@$IP 'cloud-init status --long'

    az group delete -n $RG --yes --no-wait
    ```

    Réponses : la **région** est une zone géographique (France Central = Paris) ; une **zone de disponibilité** est un ou plusieurs datacenters isolés (énergie, réseau, refroidissement) dans la région ; **domaines d'erreur** (rack, alimentation commune) et **domaines de mise à jour** (groupes redémarrés ensemble pendant la maintenance) répartissent les VM d'un groupe à haute disponibilité dans un datacenter. VM arrêtée depuis Linux : toujours allouée, **le calcul reste facturé**. Désallouée : tu ne paies plus que les disques et l'IP publique.

## TP3 — VNet, sous-réseaux, NSG et Network Watcher

**Objectif** : l'« exo » des p. 61-62 : un VNet, deux sous-réseaux, des NSG appliqués aux sous-réseaux, une VM publique et une VM privée. **Coût** < 0,30 $.

1. Crée `vnet-lab` en 10.10.0.0/16 avec `snet-front` (10.10.1.0/24) et `snet-back` (10.10.2.0/24).
2. Crée `nsg-front` (SSH depuis ton IP, HTTP depuis Internet) et `nsg-back` (SSH depuis `snet-front` uniquement). Associe-les aux sous-réseaux, pas aux cartes réseau.
3. Déploie `vm-front` avec IP publique dans `snet-front` et `vm-back` sans IP publique dans `snet-back`.
4. Connecte-toi à `vm-back` par rebond : `ssh -J azureuser@<IP front> azureuser@10.10.2.4`.
5. Depuis `vm-back`, lance `curl -m 5 ifconfig.me` et `sudo apt update`. Observe.
6. Dans Network Watcher, utilise « IP flow verify » et « Connection troubleshoot » (p. 74) pour expliquer pourquoi un SSH depuis Internet vers `vm-back` est refusé, et quelle règle l'a décidé.

**Validation** : le rebond SSH fonctionne ; `vm-back` n'a pas Internet (sous-réseau privé par défaut) ; Network Watcher nomme la règle qui bloque.

**Questions** : pourquoi `vm-back` n'a-t-il pas Internet alors que la slide p. 72 parle de SNAT implicite ? Quelles sont les quatre façons de lui en donner ? Pourquoi associer le NSG au sous-réseau plutôt qu'à la carte ?

**Nettoyage** : garde tes commandes, elles serviront au TP17. Supprime `rg-tp03`.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-tp03; MYIP=<ton IP>
    az group create -n $RG -l $LOC --tags tp=03 owner=admin

    az network vnet create -g $RG -n vnet-lab --address-prefixes 10.10.0.0/16 \
      --subnet-name snet-front --subnet-prefixes 10.10.1.0/24
    az network vnet subnet create -g $RG --vnet-name vnet-lab -n snet-back --address-prefixes 10.10.2.0/24

    az network nsg create -g $RG -n nsg-front
    az network nsg rule create -g $RG --nsg-name nsg-front -n ssh-mon-ip --priority 100 \
      --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow
    az network nsg rule create -g $RG --nsg-name nsg-front -n http --priority 110 \
      --source-address-prefixes Internet --destination-port-ranges 80 --protocol Tcp --access Allow

    az network nsg create -g $RG -n nsg-back
    az network nsg rule create -g $RG --nsg-name nsg-back -n ssh-depuis-front --priority 100 \
      --source-address-prefixes 10.10.1.0/24 --destination-port-ranges 22 --protocol Tcp --access Allow

    az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-front --network-security-group nsg-front
    az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-back  --network-security-group nsg-back

    # --nsg "" : pas de NSG sur la carte, seul celui du sous-réseau s'applique
    az vm create -g $RG -n vm-front --image Ubuntu2404 --size Standard_B1s --vnet-name vnet-lab \
      --subnet snet-front --nsg "" --public-ip-sku Standard --admin-username azureuser --generate-ssh-keys
    az vm create -g $RG -n vm-back --image Ubuntu2404 --size Standard_B1s --vnet-name vnet-lab \
      --subnet snet-back --nsg "" --public-ip-address "" --admin-username azureuser --generate-ssh-keys

    FRONT=$(az vm show -d -g $RG -n vm-front --query publicIps -o tsv)
    ssh -J azureuser@$FRONT azureuser@10.10.2.4 'curl -m 5 ifconfig.me; echo code=$?'

    # Network Watcher : qui décide ?
    az network watcher test-ip-flow -g $RG --vm vm-back --direction Inbound --protocol TCP \
      --local 10.10.2.4:22 --remote 198.51.100.7:50000    # Deny, DenyAllInBound
    az network watcher test-ip-flow -g $RG --vm vm-back --direction Inbound --protocol TCP \
      --local 10.10.2.4:22 --remote 10.10.1.4:50000       # Allow, ssh-depuis-front
    ```

    Réponses : depuis le 31 mars 2026, les sous-réseaux des nouveaux VNet sont créés avec `defaultOutboundAccess = false` : plus de SNAT implicite, le `curl` expire. Quatre façons de donner Internet à `vm-back` : une IP publique sur la carte, une NAT Gateway sur le sous-réseau, une règle sortante de Load Balancer (TP5), une route 0.0.0.0/0 vers une passerelle (TP4). Pour un lab seulement, `az network vnet subnet update … --default-outbound-access true` puis désallocation/démarrage de la VM rétablit l'ancien comportement. NSG au sous-réseau : un seul endroit à gérer, appliqué d'office à toute nouvelle VM, et pas de double filtrage difficile à déboguer (E5).
