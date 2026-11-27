---
icon: material/flask-outline
title: "TP Azure : Automatisation (TP17)"
tags: [cloud, azure, tp, iac, azure cli, terraform, ansible, vmss, autoscaling]
---

# 🧪 TP Azure : Automatisation (TP17)

!!! abstract "Ce que tu vas faire"
    La même infrastructure par script CLI, Terraform et Ansible, puis un VMSS qui s'adapte à la charge.

**Niveau :** 🟡 Intermédiaire · **Durée :** 3 h (en quatre séances) · **Outil :** Azure for Students

!!! tip "Comment utiliser cette page"
    Fais d'abord chaque TP **seul**, au portail puis en CLI. Le corrigé est replié sous chaque TP : ouvre **✅ Corrigé et commandes** pour vérifier ou si tu bloques.

    Toutes les commandes supposent un shell bash avec Azure CLI connecté (`az login`), idéalement WSL sur ton PC, et les variables `LOC=francecentral` et `RG=<groupe du TP>`. Remplace les valeurs entre chevrons.

    Règles du parcours (budget, nettoyage, tailles de VM) : [Démarrer le parcours Azure](memo-parcours-azure.md).

---

## TP17 — IaC : script CLI, Terraform, Ansible, autoscaling

**Objectif** : déployer la même infrastructure de trois façons et vivre la différence impératif / déclaratif (p. 117), puis l'autoscaling des Hands-on (p. 122). **Slides du cours** p. 117-120, 122. **Coût** < 1 $. Fais-le en quatre séances.

**Partie A — Script Azure CLI (impératif)**

1. À partir de la commande de la p. 118 (`az group list … | grep -q || az group create …`), écris `deploy-tp03.sh` qui recrée l'infrastructure du TP3 (VNet, 2 sous-réseaux, 2 NSG, 1 VM).
2. Rends chaque étape idempotente : le script relancé ne doit ni planter ni dupliquer.
3. Ajoute `set -euo pipefail`, des variables en tête et un `destroy-tp03.sh`.

**Partie B — Terraform (déclaratif)**

1. Décris la même infrastructure dans `main.tf` avec le fournisseur `azurerm`.
2. `terraform init`, `plan`, `apply`. Puis change le port d'une règle NSG et relance `plan` : observe la modification en place.
3. Supprime une règle à la main dans le portail et relance `plan` : Terraform détecte la dérive.
4. `terraform destroy`.

**Partie C — Ansible (configuration)**

1. Après `terraform apply`, écris un inventaire avec l'IP de la VM et un playbook qui installe Nginx et dépose une page d'accueil.
2. Lance-le deux fois : la seconde exécution doit afficher `changed=0`.

**Partie D — VMSS et autoscaling**

1. Crée un Virtual Machine Scale Set Ubuntu B1s (1 instance, sans Load Balancer ni accès Internet).
2. Ajoute une règle d'échelle : +1 instance si CPU > 60 % sur 5 min, –1 si CPU < 25 %, entre 1 et 3 instances.
3. Charge le CPU de l'instance à distance avec `az vmss run-command` et observe l'ajout d'instances, puis le retour à 1.

**Validation** : le script se relance sans erreur ; `terraform plan` affiche « No changes » après `apply` ; Ansible affiche `changed=0` au second passage ; le VMSS monte à 2 ou 3 instances puis redescend.

**Questions** : pourquoi le cours conseille-t-il Terraform pour l'infrastructure et Ansible pour la configuration ? Où Terraform stocke-t-il l'état et pourquoi ce fichier est-il sensible ? Pourquoi l'autoscaling n'a-t-il de sens qu'avec des VM « cattle » ?

**Attention quota** : 3 instances B1s = 3 vCPU. Désalloue tes autres VM avant la partie D.

**Nettoyage** : `terraform destroy`, `az group delete -n rg-tp17`.

??? success "✅ Corrigé — Partie A — Script idempotent"

    La ligne de la p. 118 a un piège : `grep $resourceGroup -q` cherche une sous-chaîne, donc `rg-tp1` « existe » dès que `rg-tp17` existe. On utilise plutôt `az group exists` et des `show` ciblés. Les règles NSG et les associations sont des PUT côté Azure : les rejouer met à jour sans dupliquer.

    ```bash
    #!/usr/bin/env bash
    # deploy-tp03.sh — usage : MYIP=x.x.x.x ./deploy-tp03.sh
    set -euo pipefail
    LOC=francecentral; RG=rg-tp17; VNET=vnet-lab
    MYIP=${MYIP:?définis MYIP}
    exists() { "$@" >/dev/null 2>&1; }

    [ "$(az group exists -n $RG)" = true ] || az group create -n $RG -l $LOC --tags tp=17 owner=admin -o none
    exists az network vnet show -g $RG -n $VNET || \
      az network vnet create -g $RG -n $VNET --address-prefixes 10.10.0.0/16 -o none

    for s in "snet-front 10.10.1.0/24 nsg-front" "snet-back 10.10.2.0/24 nsg-back"; do
      set -- $s
      exists az network vnet subnet show -g $RG --vnet-name $VNET -n $1 || \
        az network vnet subnet create -g $RG --vnet-name $VNET -n $1 --address-prefixes $2 -o none
      exists az network nsg show -g $RG -n $3 || az network nsg create -g $RG -n $3 -o none
    done

    az network nsg rule create -g $RG --nsg-name nsg-front -n ssh-mon-ip --priority 100 \
      --source-address-prefixes $MYIP/32 --destination-port-ranges 22 --protocol Tcp --access Allow -o none
    az network nsg rule create -g $RG --nsg-name nsg-front -n http --priority 110 \
      --source-address-prefixes Internet --destination-port-ranges 80 --protocol Tcp --access Allow -o none
    az network nsg rule create -g $RG --nsg-name nsg-back -n ssh-depuis-front --priority 100 \
      --source-address-prefixes 10.10.1.0/24 --destination-port-ranges 22 --protocol Tcp --access Allow -o none
    az network vnet subnet update -g $RG --vnet-name $VNET -n snet-front --network-security-group nsg-front -o none
    az network vnet subnet update -g $RG --vnet-name $VNET -n snet-back  --network-security-group nsg-back  -o none

    exists az vm show -g $RG -n vm-front || \
      az vm create -g $RG -n vm-front --image Ubuntu2404 --size Standard_B1s --vnet-name $VNET \
        --subnet snet-front --nsg "" --public-ip-sku Standard \
        --admin-username azureuser --generate-ssh-keys -o none
    echo "OK — $(az vm show -d -g $RG -n vm-front --query publicIps -o tsv)"
    ```

    `destroy-tp03.sh` tient en une ligne : `az group delete -n rg-tp17 --yes`.

??? success "✅ Corrigé — Partie B — Terraform"

    `main.tf` :

    ```hcl
    terraform {
      required_providers {
        azurerm = {
          source  = "hashicorp/azurerm"
          version = "~> 4.0"
        }
      }
    }

    provider "azurerm" {
      features {}
      subscription_id = var.subscription_id
    }

    variable "subscription_id" { type = string }
    variable "my_ip" { type = string }
    variable "location" { default = "francecentral" }

    locals {
      tags = { tp = "17", owner = "admin" }
    }

    resource "azurerm_resource_group" "rg" {
      name     = "rg-tp17-tf"
      location = var.location
      tags     = local.tags
    }

    resource "azurerm_virtual_network" "vnet" {
      name                = "vnet-lab"
      location            = azurerm_resource_group.rg.location
      resource_group_name = azurerm_resource_group.rg.name
      address_space       = ["10.10.0.0/16"]
      tags                = local.tags
    }

    resource "azurerm_subnet" "front" {
      name                 = "snet-front"
      resource_group_name  = azurerm_resource_group.rg.name
      virtual_network_name = azurerm_virtual_network.vnet.name
      address_prefixes     = ["10.10.1.0/24"]
    }

    resource "azurerm_subnet" "back" {
      name                 = "snet-back"
      resource_group_name  = azurerm_resource_group.rg.name
      virtual_network_name = azurerm_virtual_network.vnet.name
      address_prefixes     = ["10.10.2.0/24"]
    }

    resource "azurerm_network_security_group" "front" {
      name                = "nsg-front"
      location            = azurerm_resource_group.rg.location
      resource_group_name = azurerm_resource_group.rg.name
      tags                = local.tags

      security_rule {
        name                       = "ssh-mon-ip"
        priority                   = 100
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "${var.my_ip}/32"
        destination_address_prefix = "*"
      }

      security_rule {
        name                       = "http"
        priority                   = 110
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "80"
        source_address_prefix      = "Internet"
        destination_address_prefix = "*"
      }
    }

    resource "azurerm_network_security_group" "back" {
      name                = "nsg-back"
      location            = azurerm_resource_group.rg.location
      resource_group_name = azurerm_resource_group.rg.name
      tags                = local.tags

      security_rule {
        name                       = "ssh-depuis-front"
        priority                   = 100
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "*"
        destination_port_range     = "22"
        source_address_prefix      = "10.10.1.0/24"
        destination_address_prefix = "*"
      }
    }

    resource "azurerm_subnet_network_security_group_association" "front" {
      subnet_id                 = azurerm_subnet.front.id
      network_security_group_id = azurerm_network_security_group.front.id
    }

    resource "azurerm_subnet_network_security_group_association" "back" {
      subnet_id                 = azurerm_subnet.back.id
      network_security_group_id = azurerm_network_security_group.back.id
    }

    resource "azurerm_public_ip" "front" {
      name                = "pip-vm-front"
      location            = azurerm_resource_group.rg.location
      resource_group_name = azurerm_resource_group.rg.name
      sku                 = "Standard"
      allocation_method   = "Static"
      tags                = local.tags
    }

    resource "azurerm_network_interface" "front" {
      name                = "nic-vm-front"
      location            = azurerm_resource_group.rg.location
      resource_group_name = azurerm_resource_group.rg.name

      ip_configuration {
        name                          = "ipconfig1"
        subnet_id                     = azurerm_subnet.front.id
        private_ip_address_allocation = "Dynamic"
        public_ip_address_id          = azurerm_public_ip.front.id
      }
    }

    resource "azurerm_linux_virtual_machine" "front" {
      name                  = "vm-front"
      location              = azurerm_resource_group.rg.location
      resource_group_name   = azurerm_resource_group.rg.name
      size                  = "Standard_B1s"
      admin_username        = "azureuser"
      network_interface_ids = [azurerm_network_interface.front.id]
      tags                  = local.tags

      admin_ssh_key {
        username   = "azureuser"
        public_key = file("~/.ssh/id_rsa.pub")
      }

      os_disk {
        caching              = "ReadWrite"
        storage_account_type = "Standard_LRS"
      }

      source_image_reference {
        publisher = "Canonical"
        offer     = "ubuntu-24_04-lts"
        sku       = "server"
        version   = "latest"
      }
    }

    output "front_ip" {
      value = azurerm_public_ip.front.ip_address
    }
    ```

    ```bash
    cat > terraform.tfvars <<EOF
    subscription_id = "$(az account show --query id -o tsv)"
    my_ip           = "<ton IP>"
    EOF
    terraform init
    terraform plan -out tf.plan && terraform apply tf.plan
    # change destination_port_range "80" en "8080" dans la règle http, puis :
    terraform plan        # ~ update in-place sur azurerm_network_security_group.front
    # supprime la règle ssh-depuis-front au portail, puis :
    terraform plan        # Terraform veut la recréer : c'est la détection de dérive
    terraform destroy
    ```

    Observation bonus : selon la version d'API qu'utilise le fournisseur, tes sous-réseaux Terraform peuvent ne pas être privés, contrairement à ceux de la CLI. Vérifie avec `az network vnet subnet show … --query defaultOutboundAccess`, puis fixe-le explicitement dans le code.

??? success "✅ Corrigé — Partie C — Ansible"

    ```bash
    sudo apt-get install -y ansible
    cat > inventory.ini <<EOF
    [web]
    vm-front ansible_host=$(terraform output -raw front_ip) ansible_user=azureuser
    EOF
    ```

    `nginx.yml` :

    ```yaml
    - name: Serveur web
      hosts: web
      become: true
      tasks:
        - name: Installer nginx
          ansible.builtin.apt:
            name: nginx
            state: present
            update_cache: true
            cache_valid_time: 3600

        - name: Page d'accueil
          ansible.builtin.copy:
            dest: /var/www/html/index.html
            content: "<h1>{{ inventory_hostname }} - deploye par Ansible</h1>\n"
            mode: "0644"

        - name: Nginx actif au demarrage
          ansible.builtin.service:
            name: nginx
            state: started
            enabled: true
    ```

    `ansible-playbook -i inventory.ini nginx.yml`, deux fois : la seconde doit finir sur `changed=0`.

??? success "✅ Corrigé — Partie D — VMSS et autoscaling"

    Pour rester robuste, le scale set n'a ni Load Balancer ni accès Internet : on observe seulement le nombre d'instances, et on charge le CPU avec `yes` (déjà présent, contrairement à `stress-ng` qu'il faudrait installer).

    ```bash
    LOC=francecentral; RG=rg-tp17
    az group create -n $RG -l $LOC --tags tp=17 owner=admin
    az network vnet create -g $RG -n vnet-vmss --address-prefixes 10.90.0.0/16 \
      --subnet-name snet-vmss --subnet-prefixes 10.90.1.0/24
    az vmss create -g $RG -n vmss-web --image Ubuntu2404 --vm-sku Standard_B1s --instance-count 1 \
      --vnet-name vnet-vmss --subnet snet-vmss --orchestration-mode Uniform \
      --upgrade-policy-mode Automatic --lb "" --admin-username azureuser --generate-ssh-keys

    az monitor autoscale create -g $RG --resource vmss-web \
      --resource-type Microsoft.Compute/virtualMachineScaleSets \
      --name as-web --min-count 1 --max-count 3 --count 1
    az monitor autoscale rule create -g $RG --autoscale-name as-web \
      --condition "Percentage CPU > 60 avg 5m" --scale out 1
    az monitor autoscale rule create -g $RG --autoscale-name as-web \
      --condition "Percentage CPU < 25 avg 5m" --scale in 1

    az vmss run-command invoke -g $RG -n vmss-web --instance-id 0 --command-id RunShellScript \
      --scripts "nohup timeout 1200 sh -c 'yes > /dev/null' >/dev/null 2>&1 &"
    watch -n 30 "az vmss list-instances -g $RG -n vmss-web -o table"
    ```

    À observer : le scale set passe à 2 instances après 5 à 10 min, puis s'arrête là. La moyenne de deux instances (100 % + 0 %) vaut 50 %, sous le seuil de 60 % : c'est la moyenne du groupe qui compte, pas la VM la plus chargée. Après 20 min, la charge s'arrête et le scale set redescend à 1.

    Réponses : Terraform connaît l'état de toute l'infrastructure, calcule un plan et détecte la dérive ; Ansible configure l'intérieur des machines avec des modules idempotents. L'état est par défaut dans `terraform.tfstate` en local ; il contient les identifiants des ressources et parfois des secrets en clair : il va dans un backend distant verrouillé (conteneur Azure Storage), jamais dans Git. L'autoscaling crée et détruit des instances identiques : tout ce qui est écrit localement sur une instance disparaît au scale-in, d'où des VM « cattle » sans état.
