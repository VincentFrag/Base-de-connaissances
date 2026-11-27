---
icon: material/flask-outline
title: "TP Azure : Réseau avancé (TP4 et TP5)"
tags: [cloud, azure, tp, hub and spoke, appairage, nva, routage, load balancer, haute disponibilité]
---

# 🧪 TP Azure : Réseau avancé (TP4 et TP5)

!!! abstract "Ce que tu vas faire"
    Hub & spoke avec une VM passerelle à la place d'Azure Firewall, puis un Load Balancer L4 sur deux zones.

**Niveau :** 🟡 Intermédiaire · **Durée :** 3 h 30 · **Outil :** Azure for Students

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP4 — Hub & spoke avec VM passerelle

**Objectif** : reproduire l'« Exo : Network » (p. 76-79) et le hub & spoke de la p. 62, en remplaçant Azure Firewall par une VM Linux passerelle (NVA) pilotée par une route personnalisée (p. 69). **Coût** < 0,50 $.

1. Crée `vnet-hub` (10.0.0.0/16, sous-réseau `snet-nva` 10.0.1.0/24) et `vnet-spoke` (10.1.0.0/16, sous-réseau `snet-workers` 10.1.1.0/24).
2. Appaire les deux VNet dans les deux sens en autorisant le trafic transféré (« forwarded traffic »).
3. Déploie `vm-nva` (IP publique, IP privée fixe 10.0.1.4) et active le transfert IP sur sa carte réseau côté Azure.
4. Sur `vm-nva`, active `ip_forward` de façon permanente et ajoute la règle `MASQUERADE` de la slide pour 10.1.0.0/16. Rends les règles iptables persistantes.
5. Déploie `vm-worker` sans IP publique dans `snet-workers`.
6. Crée une table de routage avec `0.0.0.0/0 → VirtualAppliance 10.0.1.4` et associe-la à `snet-workers`.
7. Depuis `vm-worker` (rebond par `vm-nva`) : `curl ifconfig.me`, `sudo apt update`, `tracepath 1.1.1.1`.

**Validation** : `vm-worker` sort sur Internet avec l'IP publique de `vm-nva` ; en retirant la table de routage, il perd Internet.

**Différence avec le cours** : la partie VLAN 802.1Q (netplan `vlans:`) ne fonctionne pas dans Azure, dont le réseau virtuel ne transporte pas les trames taguées. Fais-la en local avec deux VM sur ton PC ou ton homelab, avec le netplan de la p. 78 ; dans Azure, c'est la route personnalisée qui joue le rôle du VLAN + passerelle.

**Questions** : la slide p. 79 écrit `sudo echo "net.ipv4.ip_forward=1" >> sudo /etc/sysctl.conf` — pourquoi cette commande ne marche-t-elle pas ? Pourquoi faut-il activer le transfert IP à la fois dans Linux et sur la carte Azure ? Ajoute un second spoke : peut-il joindre le premier ? Que faut-il pour que ce soit le cas ?

**Nettoyage** : `az group delete -n rg-tp04`.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-tp04; MYIP=<ton IP>
    az group create -n $RG -l $LOC --tags tp=04 owner=admin

    az network vnet create -g $RG -n vnet-hub   --address-prefixes 10.0.0.0/16 \
      --subnet-name snet-nva     --subnet-prefixes 10.0.1.0/24
    az network vnet create -g $RG -n vnet-spoke --address-prefixes 10.1.0.0/16 \
      --subnet-name snet-workers --subnet-prefixes 10.1.1.0/24

    az network vnet peering create -g $RG -n hub-to-spoke --vnet-name vnet-hub \
      --remote-vnet vnet-spoke --allow-vnet-access --allow-forwarded-traffic
    az network vnet peering create -g $RG -n spoke-to-hub --vnet-name vnet-spoke \
      --remote-vnet vnet-hub --allow-vnet-access --allow-forwarded-traffic

    # Passerelle : IP privée fixe, NSG sans règle SSH ouverte à tous
    az vm create -g $RG -n vm-nva --image Ubuntu2404 --size Standard_B1s \
      --vnet-name vnet-hub --subnet snet-nva --private-ip-address 10.0.1.4 \
      --public-ip-sku Standard --nsg-rule NONE --admin-username azureuser --generate-ssh-keys
    az network nsg rule create -g $RG --nsg-name vm-nvaNSG -n ssh-mon-ip --priority 100 \
      --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow

    # Transfert IP côté Azure (sinon la plateforme jette les paquets qui ne sont pas pour la VM)
    NIC=$(az vm show -g $RG -n vm-nva --query "networkProfile.networkInterfaces[0].id" -o tsv)
    az network nic update --ids $NIC --ip-forwarding true

    az vm create -g $RG -n vm-worker --image Ubuntu2404 --size Standard_B1s \
      --vnet-name vnet-spoke --subnet snet-workers --public-ip-address "" --nsg "" \
      --admin-username azureuser --generate-ssh-keys

    az network route-table create -g $RG -n rt-spoke
    az network route-table route create -g $RG --route-table-name rt-spoke -n default-via-nva \
      --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.4
    az network vnet subnet update -g $RG --vnet-name vnet-spoke -n snet-workers --route-table rt-spoke
    ```

    Sur `vm-nva` (`ssh azureuser@<IP publique>`) :

    ```bash
    echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf
    sudo sysctl --system
    sudo iptables -P FORWARD ACCEPT                      # acceptable en lab, à restreindre en vrai
    sudo iptables -t nat -A POSTROUTING -s 10.1.0.0/16 ! -d 10.0.0.0/8 -o eth0 -j MASQUERADE
    sudo DEBIAN_FRONTEND=noninteractive apt-get install -y iptables-persistent
    sudo netfilter-persistent save
    ```

    Test depuis ton PC : `ssh -J azureuser@<IP nva> azureuser@10.1.1.4 'curl -s ifconfig.me; tracepath -n 1.1.1.1 | head -3'`. Dans Azure l'interface s'appelle `eth0`, pas `ens3` comme chez DENV-R. Si le trafic ne passe pas, regarde `sudo iptables -t nat -L -v` (compteurs) et « Effective routes » de la carte de `vm-worker`.

    Réponses : `sudo echo … >> sudo /etc/sysctl.conf` échoue deux fois — la redirection `>>` est faite par ton shell non privilégié (le `sudo` ne s'applique qu'à `echo`), et `sudo` est pris pour un nom de fichier. On utilise `| sudo tee -a`, ou mieux un fichier dans `/etc/sysctl.d/`. Le transfert IP est nécessaire deux fois : Linux doit accepter de router, et Azure doit accepter que la carte émette et reçoive des paquets dont elle n'est ni la source ni la destination. Un second spoke ne joint pas le premier (appairage non transitif) ; il faut dans chaque spoke une route vers l'autre spoke via 10.0.1.4. C'est pour ça que la règle `MASQUERADE` exclut `10.0.0.0/8` : on garde les vraies IP sources entre réseaux internes.

## TP5 — Load Balancer L4

**Objectif** : répartir du trafic HTTP sur deux VM dans deux zones, avec sonde de santé et sortie Internet maîtrisée. **Slides du cours** p. 65-68, 72. **Coût** < 0,50 $.

1. Crée un VNet et un sous-réseau privé avec un NSG qui autorise le port 80 depuis Internet.
2. Crée un Load Balancer Standard avec IP publique, un pool backend, une sonde HTTP sur `/`, une règle 80 → 80 et une règle sortante (outbound rule) pour que les VM puissent installer leurs paquets.
3. Crée deux VM sans IP publique, en zones 1 et 2, déjà rattachées au pool, avec les deux cloud-init de la p. 65 (Apache sur l'une, Nginx sur l'autre) qui affichent le nom de la VM.
4. Appelle l'IP du Load Balancer une dizaine de fois, puis arrête une VM et recommence.

**Validation** : les réponses alternent entre les deux VM ; après l'arrêt d'une VM, la sonde la sort du pool en moins d'une minute et le service continue.

**Questions** : pourquoi rattacher les VM au pool *avant* leur premier démarrage ? Quelle est l'IP de sortie des VM et pourquoi ? Quand choisirais-tu Application Gateway, Traffic Manager ou Front Door à la place (p. 66-68) ? Calcule le SLA de cette architecture (E3).

**Nettoyage** : `az group delete -n rg-tp05`.

??? success "✅ Corrigé et commandes"

    `cloud-init-1.yaml` (Apache) est celui du TP2 ; `cloud-init-2.yaml` (Nginx) :

    ```yaml
    #cloud-config
    package_update: true
    packages:
      - nginx
    runcmd:
      - echo "<h1>$(hostname) - Nginx</h1>" > /var/www/html/index.html
      - systemctl enable nginx
      - systemctl restart nginx
    ```

    ```bash
    LOC=francecentral; RG=rg-tp05
    az group create -n $RG -l $LOC --tags tp=05 owner=admin

    az network vnet create -g $RG -n vnet-lb --address-prefixes 10.20.0.0/16 \
      --subnet-name snet-web --subnet-prefixes 10.20.1.0/24
    az network nsg create -g $RG -n nsg-web
    az network nsg rule create -g $RG --nsg-name nsg-web -n http --priority 100 \
      --source-address-prefixes Internet --destination-port-ranges 80 --protocol Tcp --access Allow
    az network vnet subnet update -g $RG --vnet-name vnet-lb -n snet-web --network-security-group nsg-web

    az network public-ip create -g $RG -n pip-lb --sku Standard
    az network lb create -g $RG -n lb-web --sku Standard --public-ip-address pip-lb \
      --frontend-ip-name fe-web --backend-pool-name bp-web
    az network lb probe create -g $RG --lb-name lb-web -n probe-http --protocol Http --port 80 --path /
    az network lb rule create -g $RG --lb-name lb-web -n rule-http --protocol Tcp \
      --frontend-port 80 --backend-port 80 --frontend-ip-name fe-web --backend-pool-name bp-web \
      --probe-name probe-http --disable-outbound-snat true
    az network lb outbound-rule create -g $RG --lb-name lb-web -n outbound-web \
      --frontend-ip-configs fe-web --protocol All --address-pool bp-web

    # Cartes créées DÉJÀ dans le pool, pour que cloud-init ait Internet au premier boot
    for i in 1 2; do
      az network nic create -g $RG -n nic-web0$i --vnet-name vnet-lb --subnet snet-web \
        --lb-name lb-web --lb-address-pools bp-web
      az vm create -g $RG -n vm-web0$i --nics nic-web0$i --zone $i --image Ubuntu2404 \
        --size Standard_B1s --custom-data cloud-init-$i.yaml \
        --admin-username azureuser --generate-ssh-keys
    done

    LBIP=$(az network public-ip show -g $RG -n pip-lb --query ipAddress -o tsv)
    for i in $(seq 10); do curl -s http://$LBIP; done
    az vm stop -g $RG -n vm-web01        # puis relance la boucle au bout d'une minute
    ```

    Réponses : les VM sont dans un sous-réseau privé ; leur seul accès Internet est la règle sortante du Load Balancer, qui ne s'applique qu'aux membres du pool. Hors du pool au premier démarrage, cloud-init n'installe rien. L'IP de sortie est donc `pip-lb`. Application Gateway : L7 régional (routage par chemin ou hôte, terminaison TLS, WAF). Traffic Manager : répartition globale par DNS. Front Door : L7 global avec CDN et WAF, devant plusieurs régions. SLA : 99,99 % (LB) × 99,99 % (deux VM sur deux zones, valeur contractuelle) ≈ 99,98 %.
