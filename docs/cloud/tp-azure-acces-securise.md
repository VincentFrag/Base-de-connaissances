---
icon: material/flask-outline
title: "TP Azure : Accès sécurisé (TP8 à TP10)"
tags: [cloud, azure, tp, bastion, teleport, wireguard, vpn gateway, point-à-site, vpn]
---

# 🧪 TP Azure : Accès sécurisé (TP8 à TP10)

!!! abstract "Ce que tu vas faire"
    Trois façons d'atteindre une VM privée : Azure Bastion puis Teleport, WireGuard entre deux VNet, VPN Gateway point-à-site.

**Niveau :** 🟡 Intermédiaire · **Durée :** 5 h 30 · **Outil :** Azure for Students

Trois façons d'atteindre une VM privée : un bastion (TP8), un VPN que tu gères toi-même (TP9), un VPN managé par Azure (TP10). Fais-les dans cet ordre : du moins cher au plus cher.

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP8 — Azure Bastion puis Teleport (Exo Bastion)

**Objectif** : se connecter à des VM sans IP publique, d'abord avec le service managé, puis avec un bastion open source. **Slides du cours** p. 82-83, 134. **Coût** < 1 $.

**Exo 1 — Azure Bastion**

1. Crée `vnet-hub` (avec une VM privée) et `vnet-prod` (avec une VM privée), appairés.
2. Depuis la VM du hub, « Se connecter → Bastion » et déploie la référence **Developer** : gratuite, mais limitée au VNet où elle est déployée. Ouvre une session SSH dans le navigateur.
3. Essaie d'atteindre la VM de `vnet-prod` : échec, Developer ne suit pas l'appairage.
4. Remplace-la par la référence **Basic** (sous-réseau `AzureBastionSubnet` en /26, IP publique Standard, environ 10 min de déploiement, facturée à l'heure). Connecte-toi à la VM de `vnet-prod`. **Supprime le Bastion Basic dès que c'est validé.**

**Exo 2 — Teleport** (alternative : Guacamole ou The Bastion)

1. Comme p. 83 : une VM `bastion` (B2s) avec IP publique et nom DNS (`<nom>.francecentral.cloudapp.azure.com`), NSG ouvert en 443 et 3022-3025 ; une VM `prod` sans IP publique dans le VNet appairé.
2. Installe Teleport Community Edition en suivant l'historique de commandes p. 83 (certificat Let's Encrypt via ACME), crée un utilisateur admin, active la MFA.
3. Inscris la VM `prod` comme nœud Teleport avec un jeton d'invitation. Elle a besoin d'Internet pour installer le paquet : rends son sous-réseau non privé le temps de l'installation.
4. Ouvre une session SSH sur `prod` depuis l'interface web de Teleport, puis avec `tsh ssh`.

**Validation** : connexion aux deux VM privées sans aucune IP publique sur elles ; la session Teleport est enregistrée et rejouable.

**Questions** : compare Bastion Basic et Teleport sur le coût mensuel, la traçabilité et la dépendance au fournisseur. Pourquoi un bastion réduit-il la surface d'attaque (p. 45) ? Qu'est-ce qu'un certificat SSH à courte durée de vie change par rapport à `authorized_keys` (p. 125) ?

**Nettoyage** : `az group delete -n rg-tp08`.

??? success "✅ Corrigé et commandes"

    **Exo 1**

    ```bash
    LOC=francecentral; RG=rg-tp08; MYIP=<ton IP>
    az group create -n $RG -l $LOC --tags tp=08 owner=admin

    az network vnet create -g $RG -n vnet-hub  --address-prefixes 10.70.0.0/16 --subnet-name snet-vm --subnet-prefixes 10.70.1.0/24
    az network vnet create -g $RG -n vnet-prod --address-prefixes 10.71.0.0/16 --subnet-name snet-vm --subnet-prefixes 10.71.1.0/24
    az network vnet peering create -g $RG -n hub-to-prod --vnet-name vnet-hub  --remote-vnet vnet-prod --allow-vnet-access
    az network vnet peering create -g $RG -n prod-to-hub --vnet-name vnet-prod --remote-vnet vnet-hub  --allow-vnet-access

    for v in hub prod; do
      az vm create -g $RG -n vm-$v --image Ubuntu2404 --size Standard_B1s --vnet-name vnet-$v \
        --subnet snet-vm --public-ip-address "" --nsg "" --admin-username azureuser --generate-ssh-keys
    done
    ```

    Bastion Developer : au portail, `vm-hub` → Se connecter → Bastion → « Deploy Bastion Developer », authentification par clé privée SSH. Puis Basic, après avoir supprimé le Developer :

    ```bash
    az network vnet subnet create -g $RG --vnet-name vnet-hub -n AzureBastionSubnet --address-prefixes 10.70.255.0/26
    az network public-ip create -g $RG -n pip-bastion --sku Standard
    az network bastion create -g $RG -n bas-basic --vnet-name vnet-hub \
      --public-ip-address pip-bastion --sku Basic -l $LOC        # environ 10 min
    # portail : vm-prod → Se connecter → Bastion
    az network bastion delete -g $RG -n bas-basic               # dès que c'est validé
    ```

    **Exo 2 — Teleport**

    ```bash
    az vm create -g $RG -n vm-bastion --image Ubuntu2404 --size Standard_B2s --vnet-name vnet-hub \
      --subnet snet-vm --public-ip-sku Standard --public-ip-address-dns-name bastion-lab \
      --nsg-rule NONE --admin-username azureuser --generate-ssh-keys
    for r in "ssh-mon-ip 100 22 $MYIP/32" "https 110 443 Internet" "teleport 120 3022-3025 $MYIP/32"; do
      set -- $r
      az network nsg rule create -g $RG --nsg-name vm-bastionNSG -n $1 --priority $2 \
        --destination-port-ranges $3 --source-address-prefixes "$4" --protocol Tcp --access Allow
    done
    ```

    Sur `vm-bastion` (historique de la p. 83, mis à jour) :

    ```bash
    FQDN=bastion-lab.francecentral.cloudapp.azure.com
    sudo apt update && sudo apt -y dist-upgrade
    curl https://cdn.teleport.dev/install.sh | sudo bash -s <VERSION>   # 18.6.1 dans le cours : prends la version courante
    sudo teleport configure -o file --acme --acme-email=<ton e-mail> --cluster-name=$FQDN
    sudo systemctl enable --now teleport
    sudo tctl users add admin --roles=editor,access --logins=root,ubuntu,azureuser
    # ouvre le lien d'invitation : mot de passe + MFA (TOTP)
    sudo tctl tokens add --type=node --ttl=1h        # note le jeton
    ```

    Pour `vm-prod`, ouvre temporairement Internet, installe, puis fais pointer le nom du proxy vers son IP privée :

    ```bash
    # depuis ton PC
    az network vnet subnet update -g $RG --vnet-name vnet-prod -n snet-vm --default-outbound-access true
    az vm deallocate -g $RG -n vm-prod && az vm start -g $RG -n vm-prod
    ssh -J azureuser@bastion-lab.francecentral.cloudapp.azure.com azureuser@10.71.1.4

    # sur vm-prod
    curl https://cdn.teleport.dev/install.sh | sudo bash -s <VERSION>
    echo "<IP privée de vm-bastion> bastion-lab.francecentral.cloudapp.azure.com" | sudo tee -a /etc/hosts
    sudo teleport node configure --output=file:///etc/teleport.yaml --token=<JETON> \
      --proxy=bastion-lab.francecentral.cloudapp.azure.com:443
    sudo systemctl enable --now teleport
    ```

    Remets ensuite le sous-réseau en privé (`--default-outbound-access false`, désallocation, démarrage) : le nœud reste joignable via l'appairage. Depuis ton PC : `tsh login --proxy=bastion-lab.francecentral.cloudapp.azure.com --user=admin`, `tsh ls`, `tsh ssh azureuser@vm-prod`. Les enregistrements sont dans l'interface web, rubrique Audit. L'IP privée de `vm-bastion` s'obtient avec `az vm show -d -g $RG -n vm-bastion --query privateIps -o tsv` (probablement 10.70.1.5, `vm-hub` ayant pris .4).

    Réponses : Bastion Basic est facturé à l'heure en continu (environ 0,19 $/h, soit plus de 130 $/mois, à vérifier) ; Teleport coûte une VM B2s et ton temps d'exploitation. Teleport enregistre les sessions et relie chaque accès à une identité + MFA ; chez Azure, l'enregistrement de session n'existe qu'en Premium. Teleport fonctionne partout (on-premise, multi-cloud), Bastion seulement dans Azure. Un bastion concentre les accès sur un seul point durci et supervisé, au lieu d'exposer le port 22 de chaque VM. Un certificat SSH court remplace les clés statiques dans `authorized_keys` : rien à révoquer à la main au départ d'un employé, l'accès expire tout seul.

## TP9 — WireGuard entre deux VNet (Exo VPN 2)

**Objectif** : construire ton propre VPN point-à-site vers un VNet de production. **Slides du cours** p. 49, 81. **Coût** < 0,30 $.

1. `vnet-secu` (10.30.0.0/16) avec `vm-wg` (IP publique, UDP 51820 ouvert à ton IP) ; `vnet-prod` (10.31.0.0/16) avec `vm-prod` sans IP publique ; appairage dans les deux sens.
2. Sur `vm-wg` : installe WireGuard, génère les clés, crée `wg0` sur 10.99.0.1/24, active `ip_forward` et un `MASQUERADE` vers `eth0`.
3. Sur ton PC Windows : client WireGuard, tunnel en 10.99.0.2/32, `AllowedIPs` limité aux réseaux Azure (split tunnel).
4. Ping et SSH vers l'IP privée de `vm-prod`.
5. Passe `AllowedIPs` à `0.0.0.0/0` (full tunnel) et regarde ton IP publique sur ifconfig.me.

**Validation** : `ssh azureuser@10.31.1.4` depuis ton PC fonctionne ; en full tunnel, ton IP publique devient celle de `vm-wg`.

**Questions** : split vs full tunnel (p. 49) : avantages, risques, cas d'usage ? Sans `MASQUERADE`, que faudrait-il ajouter côté Azure pour que `vm-prod` sache répondre à 10.99.0.2 ? Pourquoi WireGuard n'a-t-il pas besoin de la règle « transfert IP » sur la carte Azure dans cette configuration ?

**Nettoyage** : `az group delete -n rg-tp09`, supprime le tunnel dans le client.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-tp09; MYIP=<ton IP>
    az group create -n $RG -l $LOC --tags tp=09 owner=admin
    az network vnet create -g $RG -n vnet-secu --address-prefixes 10.30.0.0/16 --subnet-name snet-wg   --subnet-prefixes 10.30.1.0/24
    az network vnet create -g $RG -n vnet-prod --address-prefixes 10.31.0.0/16 --subnet-name snet-prod --subnet-prefixes 10.31.1.0/24
    az network vnet peering create -g $RG -n secu-to-prod --vnet-name vnet-secu --remote-vnet vnet-prod --allow-vnet-access
    az network vnet peering create -g $RG -n prod-to-secu --vnet-name vnet-prod --remote-vnet vnet-secu --allow-vnet-access

    az vm create -g $RG -n vm-wg --image Ubuntu2404 --size Standard_B1s --vnet-name vnet-secu --subnet snet-wg \
      --public-ip-sku Standard --nsg-rule NONE --admin-username azureuser --generate-ssh-keys
    az network nsg rule create -g $RG --nsg-name vm-wgNSG -n ssh-mon-ip --priority 100 \
      --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow
    az network nsg rule create -g $RG --nsg-name vm-wgNSG -n wireguard --priority 110 \
      --source-address-prefixes $MYIP/32 --destination-port-ranges 51820 --protocol Udp --access Allow
    az vm create -g $RG -n vm-prod --image Ubuntu2404 --size Standard_B1s --vnet-name vnet-prod --subnet snet-prod \
      --public-ip-address "" --nsg "" --admin-username azureuser --generate-ssh-keys
    ```

    Sur `vm-wg` :

    ```bash
    sudo apt-get update && sudo apt-get install -y wireguard
    wg genkey | sudo tee /etc/wireguard/server.key | wg pubkey | sudo tee /etc/wireguard/server.pub
    sudo chmod 600 /etc/wireguard/server.key
    echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf && sudo sysctl --system
    sudo nano /etc/wireguard/wg0.conf
    ```

    ```ini
    [Interface]
    Address = 10.99.0.1/24
    ListenPort = 51820
    PrivateKey = <contenu de server.key>
    PostUp   = iptables -t nat -A POSTROUTING -s 10.99.0.0/24 -o eth0 -j MASQUERADE
    PostDown = iptables -t nat -D POSTROUTING -s 10.99.0.0/24 -o eth0 -j MASQUERADE

    [Peer]
    # ton PC : clé publique affichée par le client Windows (« Ajouter un tunnel vide »)
    PublicKey = <clé publique du PC>
    AllowedIPs = 10.99.0.2/32
    ```

    ```bash
    sudo systemctl enable --now wg-quick@wg0 && sudo wg show
    ```

    Tunnel côté Windows (la clé privée est déjà générée par l'application) :

    ```ini
    [Interface]
    PrivateKey = <générée par le client>
    Address = 10.99.0.2/32

    [Peer]
    PublicKey = <contenu de server.pub>
    Endpoint = <IP publique de vm-wg>:51820
    AllowedIPs = 10.30.0.0/16, 10.31.0.0/16, 10.99.0.0/24     # split tunnel ; 0.0.0.0/0 = full tunnel
    PersistentKeepalive = 25
    ```

    Réponses : le split tunnel n'envoie dans le VPN que le trafic vers Azure (sobre, mais le PC reste exposé directement à Internet) ; le full tunnel fait tout passer par Azure (contrôle et filtrage centralisés, mais latence, coût d'egress et dépendance au VPN). Sans `MASQUERADE`, `vm-prod` reçoit des paquets de 10.99.0.2 et n'a pas de route retour : il faut une table de routage sur `snet-prod` avec `10.99.0.0/24 → VirtualAppliance 10.30.1.4` et le transfert IP activé sur la carte de `vm-wg`. Avec `MASQUERADE`, tout ce que `vm-wg` émet vers le VNet porte sa propre IP et les réponses lui reviennent : Azure ne voit jamais de paquet « étranger », le transfert IP sur la carte est inutile.

## TP10 — VPN Gateway point-à-site (Exo VPN 1)

**Objectif** : pinguer depuis ton PC l'IP privée d'une VM Azure à travers la passerelle VPN managée. **Slides du cours** p. 48-49, 81. **Coût** 0,50 à 2 $ si supprimée le jour même.

1. VNet 10.40.0.0/16 avec un sous-réseau VM et un `GatewaySubnet` en /27 ; une VM B1s sans IP publique.
2. Lance la création d'une passerelle VPN **route-based**, référence **Basic** (la moins chère ; elle exige désormais une IP publique Standard). Compte 30 à 45 min : avance sur autre chose pendant ce temps.
3. Sur ton PC Windows, génère un certificat racine auto-signé et un certificat client (PowerShell).
4. Configure le point-à-site : pool d'adresses clients 172.16.201.0/24, protocole SSTP, certificat racine importé.
5. Télécharge le package client VPN, installe-le, connecte-toi.
6. `ping 10.40.1.4`, puis `route print` pour voir ce qui passe dans le tunnel.

**Validation** : le ping répond depuis ton PC ; la table de routage Windows montre 10.40.0.0/16 via l'interface VPN.

**Questions** : différences entre point-à-site et site-à-site (p. 49) ? Pourquoi la référence Basic est-elle déconseillée en production (protocoles, débit, zones) ? Compare le coût mensuel de cette passerelle avec ta VM WireGuard du TP9.

**Bonus** : si ton homelab a une IP publique et un pfSense, monte un site-à-site (Local Network Gateway + connexion IPsec).

**Nettoyage** : `az group delete -n rg-tp10` le jour même. La passerelle coûte même quand personne n'est connecté.

??? success "✅ Corrigé et commandes"

    ```bash
    LOC=francecentral; RG=rg-tp10
    az group create -n $RG -l $LOC --tags tp=10 owner=admin
    az network vnet create -g $RG -n vnet-vpn --address-prefixes 10.40.0.0/16 --subnet-name snet-vm --subnet-prefixes 10.40.1.0/24
    az network vnet subnet create -g $RG --vnet-name vnet-vpn -n GatewaySubnet --address-prefixes 10.40.255.0/27
    az vm create -g $RG -n vm-cible --image Ubuntu2404 --size Standard_B1s --vnet-name vnet-vpn --subnet snet-vm \
      --public-ip-address "" --nsg "" --admin-username azureuser --generate-ssh-keys

    az network public-ip create -g $RG -n pip-vpngw --sku Standard
    az network vnet-gateway create -g $RG -n vpngw --vnet vnet-vpn --public-ip-addresses pip-vpngw \
      --gateway-type Vpn --vpn-type RouteBased --sku Basic --no-wait
    az network vnet-gateway wait -g $RG -n vpngw --created       # 30 à 45 min
    ```

    Certificats, dans PowerShell sur ton PC :

    ```powershell
    $root = New-SelfSignedCertificate -Type Custom -KeySpec Signature -Subject "CN=P2SRootCert" `
      -KeyExportPolicy Exportable -HashAlgorithm sha256 -KeyLength 2048 `
      -CertStoreLocation "Cert:\CurrentUser\My" -KeyUsageProperty Sign -KeyUsage CertSign
    New-SelfSignedCertificate -Type Custom -DnsName P2SChildCert -KeySpec Signature -Subject "CN=P2SChildCert" `
      -KeyExportPolicy Exportable -HashAlgorithm sha256 -KeyLength 2048 `
      -CertStoreLocation "Cert:\CurrentUser\My" -Signer $root `
      -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")
    [Convert]::ToBase64String($root.RawData) | Set-Content -NoNewline "$HOME\root.b64"
    ```

    ```bash
    CERT=$(cat /mnt/c/Users/<ton_user>/root.b64)
    az network vnet-gateway update -g $RG -n vpngw --address-prefixes 172.16.201.0/24 --client-protocol SSTP
    az network vnet-gateway root-cert create -g $RG --gateway-name vpngw -n P2SRootCert --public-cert-data "$CERT"
    az network vnet-gateway vpn-client generate -g $RG -n vpngw --processor-architecture Amd64 -o tsv   # URL du zip
    ```

    Dans le zip, lance `WindowsAmd64\VpnClientSetupAmd64.exe`, puis connecte-toi depuis Paramètres → VPN. Ensuite `ping 10.40.1.4` et `route print | findstr 10.40`. Si la référence Basic est refusée dans ta région, prends `VpnGw1AZ` avec `--client-protocol IkeV2 OpenVPN` : plus chère, donc supprime-la encore plus vite.

    Réponses : point-à-site = un poste client vers un réseau ; site-à-site = deux réseaux reliés en permanence par un tunnel IPsec entre passerelles. Basic : SSTP uniquement (clients Windows), pas de BGP, pas de redondance de zone, débit limité : réservée aux tests. Côté coût, la passerelle est facturée chaque heure, connectée ou non, alors que la VM WireGuard du TP9 peut être désallouée et tient dans l'offre B1s gratuite. Compare les deux dans la calculatrice Azure.

    Nettoyage : `az group delete -n rg-tp10 --yes`, puis supprime les certificats de `Cert:\CurrentUser\My`.
