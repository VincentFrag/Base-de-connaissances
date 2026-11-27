---
icon: material/wrench-outline
title: Monter un VNet avec une VM publique et une VM privée
tags: [cloud, tuto, azure, azure cli, powershell, vnet, nsg, nat gateway, ssh, network watcher]
---

# 🛠️ Monter un VNet avec une VM publique et une VM privée

!!! abstract "Objectif"
    Construire le réseau d'un petit site web dans Azure : un **VNet** avec un sous-réseau **web** (exposé en HTTP) et un sous-réseau **data** (privé), des **NSG** posés sur les sous-réseaux, une VM web joignable et une VM base de données sans IP publique, administrée **par rebond SSH**. Puis donner Internet à la VM privée avec une **NAT Gateway** et vérifier les règles avec **Network Watcher**.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 h · **Coût :** quelques centimes si tout est supprimé dans la foulée

## Prérequis

- Comprendre VNet, NSG et sortie Internet → 🧠 [Le réseau dans Azure](concept-reseau-azure.md)
- Azure CLI installé, budget et tags en place → 🛠️ [Démarrer sur Azure](tuto-demarrer-azure.md)
- Connaître le rebond SSH → 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md)

## Le plan

| Élément | Valeur |
|---|---|
| Groupe de ressources | `rg-site` (tags `projet=site owner=alice env=lab`) |
| VNet | `vnet-site` · `10.20.0.0/16` |
| Sous-réseau web | `snet-web` · `10.20.1.0/24` · NSG `nsg-web` |
| Sous-réseau data | `snet-data` · `10.20.2.0/24` · NSG `nsg-data` |
| VM web | `vm-web01` · `10.20.1.10` · IP publique · Ubuntu 24.04 · `Standard_B1s` |
| VM base | `vm-db01` · `10.20.2.10` · **pas** d'IP publique · Ubuntu 24.04 · `Standard_B1s` |
| Compte admin | `azureuser`, clé SSH |

Les flux voulus :

| Source | Destination | Port | Décision |
|---|---|---|---|
| Ton IP | `vm-web01` | 22 | ✅ |
| Internet | `vm-web01` | 80 | ✅ |
| `snet-web` | `vm-db01` | 22, 3306 | ✅ |
| Internet | `vm-db01` | tous | ❌ |

---

## Étape 1 — Le groupe et les variables

```bash
RG=rg-site; LOC=francecentral
MYIP=$(curl -s https://ifconfig.me)     # ton IP publique, vue depuis ton PC
az group create -n $RG -l $LOC --tags projet=site owner=alice env=lab
```

## Étape 2 — Le VNet et ses sous-réseaux

=== "Azure CLI"

    ```bash
    az network vnet create -g $RG -n vnet-site --address-prefixes 10.20.0.0/16 \
      --subnet-name snet-web --subnet-prefixes 10.20.1.0/24
    az network vnet subnet create -g $RG --vnet-name vnet-site -n snet-data \
      --address-prefixes 10.20.2.0/24
    ```

=== "PowerShell (module Az)"

    ```powershell
    $rg = "rg-site"; $loc = "francecentral"
    $web  = New-AzVirtualNetworkSubnetConfig -Name snet-web  -AddressPrefix 10.20.1.0/24
    $data = New-AzVirtualNetworkSubnetConfig -Name snet-data -AddressPrefix 10.20.2.0/24
    New-AzVirtualNetwork -ResourceGroupName $rg -Location $loc -Name vnet-site `
      -AddressPrefix 10.20.0.0/16 -Subnet $web, $data
    ```

=== "Portail"

    **Réseaux virtuels** → **Créer** → onglet *Adresses IP* : espace `10.20.0.0/16`, supprimer le sous-réseau `default`, ajouter `snet-web` (`10.20.1.0/24`) et `snet-data` (`10.20.2.0/24`).

## Étape 3 — Les NSG, posés sur les sous-réseaux

```bash
# NSG du sous-réseau web : SSH depuis ton IP seulement, HTTP depuis partout
az network nsg create -g $RG -n nsg-web
az network nsg rule create -g $RG --nsg-name nsg-web -n ssh-admin --priority 100 \
  --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow
az network nsg rule create -g $RG --nsg-name nsg-web -n http --priority 200 \
  --source-address-prefixes Internet --destination-port-ranges 80 --protocol Tcp --access Allow

# NSG du sous-réseau data : SSH et MySQL depuis snet-web, rien d'autre venant du VNet
az network nsg create -g $RG -n nsg-data
az network nsg rule create -g $RG --nsg-name nsg-data -n depuis-web --priority 100 \
  --source-address-prefixes 10.20.1.0/24 --destination-port-ranges 22 3306 --protocol Tcp --access Allow
az network nsg rule create -g $RG --nsg-name nsg-data -n refus-vnet --priority 4000 \
  --source-address-prefixes VirtualNetwork --destination-port-ranges '*' --protocol '*' --access Deny

# Association aux sous-réseaux
az network vnet subnet update -g $RG --vnet-name vnet-site -n snet-web  --network-security-group nsg-web
az network vnet subnet update -g $RG --vnet-name vnet-site -n snet-data --network-security-group nsg-data
```

!!! info "Pourquoi la règle 4000 ?"
    Sans elle, la règle par défaut `AllowVnetInBound` (65000) laisserait **tout le VNet** joindre `snet-data`. On ne garde que ce qu'on a explicitement autorisé.

## Étape 4 — Les deux VM

`--nsg ""` évite qu'Azure crée un NSG sur la carte (celui du sous-réseau suffit) ; `--private-ip-address` fixe l'IP **côté Azure**.

```bash
az vm create -g $RG -n vm-web01 --image Ubuntu2404 --size Standard_B1s \
  --vnet-name vnet-site --subnet snet-web --private-ip-address 10.20.1.10 \
  --public-ip-sku Standard --nsg "" \
  --admin-username azureuser --generate-ssh-keys

az vm create -g $RG -n vm-db01 --image Ubuntu2404 --size Standard_B1s \
  --vnet-name vnet-site --subnet snet-data --private-ip-address 10.20.2.10 \
  --public-ip-address "" --nsg "" \
  --admin-username azureuser --generate-ssh-keys

az vm auto-shutdown -g $RG -n vm-web01 --time 1900
az vm auto-shutdown -g $RG -n vm-db01  --time 1900
```

Installer un serveur web sur `vm-web01` sans s'y connecter :

```bash
az vm run-command invoke -g $RG -n vm-web01 --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y nginx && echo '<h1>vm-web01</h1>' > /var/www/html/index.html"
```

## Étape 5 — Se connecter par rebond

```bash
WEBIP=$(az vm show -d -g $RG -n vm-web01 --query publicIps -o tsv)
curl http://$WEBIP                                  # <h1>vm-web01</h1>
ssh azureuser@$WEBIP                                # accès direct à la VM web
ssh -J azureuser@$WEBIP azureuser@10.20.2.10        # rebond vers la VM privée
```

Pour ne plus taper le rebond, dans `~/.ssh/config` :

```text
Host site-web
    HostName <IP publique de vm-web01>
    User azureuser
Host site-db
    HostName 10.20.2.10
    User azureuser
    ProxyJump site-web
```

## Étape 6 — Donner Internet à la VM privée

Sur `vm-db01`, `curl -m 5 https://ifconfig.me` échoue : le sous-réseau est privé. On ajoute une **NAT Gateway** au sous-réseau `snet-data` :

```bash
az network public-ip create -g $RG -n pip-nat --sku Standard
az network nat gateway create -g $RG -n nat-site --public-ip-addresses pip-nat --idle-timeout 4
az network vnet subnet update -g $RG --vnet-name vnet-site -n snet-data --nat-gateway nat-site
```

Relancer depuis `vm-db01` : `curl https://ifconfig.me` affiche maintenant l'IP de `pip-nat`, et `sudo apt update` passe. La VM sort sur Internet, mais **personne ne peut y entrer** depuis Internet : la NAT Gateway ne fonctionne que dans le sens sortant.

## Étape 7 — Vérifier les règles avec Network Watcher

« Est-ce que ce flux passe, et quelle règle décide ? » sans lancer un seul paquet :

```bash
az network watcher test-ip-flow -g $RG --vm vm-db01 --direction Inbound \
  --protocol TCP --local 10.20.2.10:22 --remote 198.51.100.7:50000
# → Access: Deny, RuleName: …DenyAllInBound

az network watcher test-ip-flow -g $RG --vm vm-db01 --direction Inbound \
  --protocol TCP --local 10.20.2.10:3306 --remote 10.20.1.10:50000
# → Access: Allow, RuleName: …depuis-web
```

Côté portail : **Network Watcher** → **Vérification des flux IP**, ou sur la carte réseau d'une VM → **Règles de sécurité effectives** (la liste fusionnée de tous les NSG qui s'appliquent).

## Ce que tu vois / ce que ça veut dire / action

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `ssh` vers `vm-web01` tourne dans le vide | Ton IP a changé, ou la règle `ssh-admin` vise une autre IP | `curl ifconfig.me`, mettre à jour la règle |
| `curl http://<IP>` ne répond pas | nginx pas installé ou port 80 non autorisé | `az vm run-command` pour vérifier `systemctl status nginx` ; règle `http` |
| Rebond OK mais `vm-db01` refuse SSH | Règle `depuis-web` absente ou mauvais préfixe | `test-ip-flow` depuis `10.20.1.10` |
| `apt update` bloqué sur `vm-db01` | Sous-réseau privé sans sortie | NAT Gateway (étape 6) |
| `RuleName` est un nom de règle d'un NSG de carte | Un NSG a été créé sur la carte en plus | `az network nic update --network-security-group ""` ou le supprimer |

## Vérification finale

- [ ] `curl http://<IP de vm-web01>` affiche la page
- [ ] `ssh -J` atteint `vm-db01` ; un SSH direct vers `10.20.2.10` depuis Internet est impossible
- [ ] `vm-db01` sort sur Internet avec l'IP de la NAT Gateway
- [ ] `test-ip-flow` nomme la règle qui autorise ou refuse chaque flux du plan

## Nettoyage

```bash
az group delete -n rg-site --yes --no-wait
```

La NAT Gateway et les IP publiques coûtent **à l'heure**, même sans trafic : ne pas les garder d'une séance à l'autre.

## Dépannage courant

- **`--nsg ""` refusé par une ancienne version de la CLI** : mettre à jour (`az upgrade`) ou créer la VM puis supprimer le NSG de carte créé automatiquement.
- **La VM ne démarre pas, `SkuNotAvailable`** : changer de zone ou de taille (`Standard_B2ats_v2`), voir 🛠️ [Démarrer sur Azure](tuto-demarrer-azure.md).
- **Clé SSH introuvable** : `--generate-ssh-keys` utilise `~/.ssh/id_rsa` ; avec une autre clé, passer `--ssh-key-values ~/.ssh/ma_cle.pub`.

## Pour aller plus loin

- 🧠 [Le réseau dans Azure](concept-reseau-azure.md) : hub & spoke, appairage et autres sorties Internet
- 🧠 [Identités et droits dans Azure](concept-identite-azure.md) : qui a le droit de modifier ce réseau
- 🛠️ [Déployer une infrastructure Azure avec Terraform](tuto-terraform-azure.md) : le même réseau, décrit en code
