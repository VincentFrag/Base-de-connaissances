---
icon: material/wrench-outline
title: Déployer une infrastructure Azure avec Terraform
tags: [cloud, tuto, azure, terraform, iac, vnet, nsg, cloud-init, ansible]
---

# 🛠️ Déployer une infrastructure Azure avec Terraform

!!! abstract "Objectif"
    Décrire en **Terraform** un petit environnement web Azure (groupe de ressources, VNet, sous-réseau, NSG, IP publique, VM Linux configurée par **cloud-init**), le créer avec `plan` puis `apply`, le modifier dans le code, détecter une **dérive** faite à la main dans le portail, et tout détruire avec `destroy`. En bonus : passer la main à **Ansible** pour la configuration.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 h 30 · **Coût :** quelques centimes si détruit dans la foulée

## Prérequis

- Comprendre déclaratif, plan et fichier d'état → 🧠 [L'Infrastructure as Code](concept-iac.md)
- Comprendre VNet et NSG → 🧠 [Le réseau dans Azure](concept-reseau-azure.md)
- Azure CLI installé et connecté (`az login`) → 🛠️ [Démarrer sur Azure](tuto-demarrer-azure.md)
- Une clé SSH : `ssh-keygen -t ed25519 -f ~/.ssh/azure_lab`

## Le plan

| Élément | Valeur |
|---|---|
| Dossier | `~/iac/site-tf/` (versionné dans Git) |
| Fournisseur | `hashicorp/azurerm` `~> 4.0` |
| Groupe | `rg-site-tf` |
| Réseau | `vnet-site-tf` `10.30.0.0/16` · `snet-web` `10.30.1.0/24` · `nsg-web` |
| VM | `vm-web01` · Ubuntu 24.04 · `Standard_B1s` · nginx par cloud-init |
| Accès | SSH depuis ton IP seulement, HTTP depuis Internet |

---

## Étape 1 — Installer Terraform

=== "Linux / WSL (Debian, Ubuntu)"

    ```bash
    wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
    echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
      | sudo tee /etc/apt/sources.list.d/hashicorp.list
    sudo apt update && sudo apt install -y terraform
    terraform version
    ```

=== "Windows (PowerShell)"

    ```powershell
    winget install --exact --id Hashicorp.Terraform
    # rouvrir le terminal
    terraform version
    ```

Terraform réutilise ta session Azure CLI : rien d'autre à configurer pour un usage personnel.

## Étape 2 — Le squelette du projet

```bash
mkdir -p ~/iac/site-tf && cd ~/iac/site-tf && git init
cat > .gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
EOF
```

| Fichier | Contenu |
|---|---|
| `versions.tf` | Version de Terraform et du fournisseur |
| `variables.tf` | Les paramètres (région, ton IP, taille de VM) |
| `main.tf` | Les ressources |
| `outputs.tf` | Ce qu'on veut afficher à la fin (IP publique) |
| `cloud-init.yaml` | La configuration de la VM au premier démarrage |
| `terraform.tfvars` | Tes valeurs (exclu de Git) |

## Étape 3 — Écrire le code

`versions.tf` :

```hcl
terraform {
  required_version = ">= 1.6"
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
```

`variables.tf` :

```hcl
variable "subscription_id" {
  type        = string
  description = "Identifiant de l'abonnement (az account show --query id -o tsv)"
}

variable "location" {
  type    = string
  default = "francecentral"
}

variable "admin_ip" {
  type        = string
  description = "Ton IP publique, autorisée en SSH"
}

variable "vm_size" {
  type    = string
  default = "Standard_B1s"
}

variable "tags" {
  type = map(string)
  default = {
    projet = "site-tf"
    owner  = "alice"
    env    = "lab"
  }
}
```

`main.tf` :

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "rg-site-tf"
  location = var.location
  tags     = var.tags
}

resource "azurerm_virtual_network" "vnet" {
  name                = "vnet-site-tf"
  address_space       = ["10.30.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  tags                = var.tags
}

resource "azurerm_subnet" "web" {
  name                 = "snet-web"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.30.1.0/24"]
}

resource "azurerm_network_security_group" "web" {
  name                = "nsg-web"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  tags                = var.tags

  security_rule {
    name                       = "ssh-admin"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "${var.admin_ip}/32"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "http"
    priority                   = 200
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = azurerm_subnet.web.id
  network_security_group_id = azurerm_network_security_group.web.id
}

resource "azurerm_public_ip" "web" {
  name                = "pip-web01"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
  tags                = var.tags
}

resource "azurerm_network_interface" "web" {
  name                = "nic-web01"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  tags                = var.tags

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = azurerm_subnet.web.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.30.1.10"
    public_ip_address_id          = azurerm_public_ip.web.id
  }
}

resource "azurerm_linux_virtual_machine" "web" {
  name                  = "vm-web01"
  location              = azurerm_resource_group.rg.location
  resource_group_name   = azurerm_resource_group.rg.name
  size                  = var.vm_size
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.web.id]
  custom_data           = base64encode(file("${path.module}/cloud-init.yaml"))
  tags                  = var.tags

  admin_ssh_key {
    username   = "azureuser"
    public_key = file(pathexpand("~/.ssh/azure_lab.pub"))
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
```

`outputs.tf` :

```hcl
output "ip_publique" {
  value = azurerm_public_ip.web.ip_address
}

output "ssh" {
  value = "ssh -i ~/.ssh/azure_lab azureuser@${azurerm_public_ip.web.ip_address}"
}
```

`cloud-init.yaml` :

```yaml
#cloud-config
package_update: true
packages:
  - nginx
write_files:
  - path: /var/www/html/index.html
    content: |
      <h1>vm-web01 - déployée par Terraform</h1>
runcmd:
  - systemctl enable --now nginx
```

`terraform.tfvars` (exclu de Git) :

```hcl
subscription_id = "<résultat de : az account show --query id -o tsv>"
admin_ip        = "<résultat de : curl -s https://ifconfig.me>"
```

## Étape 4 — Init, plan, apply

```bash
terraform init          # télécharge azurerm
terraform fmt           # met le code en forme
terraform validate      # Success! The configuration is valid.
terraform plan          # Plan: 8 to add, 0 to change, 0 to destroy.
terraform apply         # répondre yes
```

À la fin, les sorties s'affichent. Attendre 2 à 3 minutes que cloud-init installe nginx, puis :

```bash
curl http://$(terraform output -raw ip_publique)     # <h1>vm-web01 - déployée par Terraform</h1>
$(terraform output -raw ssh)                         # connexion SSH
terraform plan                                       # No changes. Your infrastructure matches the configuration.
```

Ce dernier `plan` sans changement est la preuve de l'**idempotence**.

## Étape 5 — Modifier par le code

Ouvrir aussi le HTTPS : dans `main.tf`, règle `http`, remplacer `destination_port_range = "80"` par :

```hcl
    destination_port_ranges    = ["80", "443"]
```

et supprimer la ligne `destination_port_range`. Puis :

```bash
terraform plan
#   ~ resource "azurerm_network_security_group" "web" {
#       ~ security_rule { … }
# Plan: 0 to add, 1 to change, 0 to destroy.
terraform apply
git add . && git commit -m "NSG : ouvre aussi 443"
```

Le `~` signifie **modification sur place** : le NSG n'est pas recréé, le site ne coupe pas.

## Étape 6 — Provoquer et corriger une dérive

Dans le portail : `nsg-web` → **Règles de sécurité de trafic entrant** → supprimer `ssh-admin`. Puis :

```bash
terraform plan
#   ~ resource "azurerm_network_security_group" "web" {
#       + security_rule { name = "ssh-admin" … }
```

Terraform a vu que la réalité ne correspond plus au code. `terraform apply` remet la règle : le code fait foi.

!!! warning "Avant chaque apply, lire les symboles"
    | Symbole | Sens | Risque |
    |---|---|---|
    | `+` | Création | Aucun |
    | `~` | Modification sur place | Faible |
    | `-` | Destruction | Perte de la ressource |
    | `-/+` | Destruction puis recréation | **Perte des données** de la ressource (disque, IP publique qui change) |

## Étape 7 (bonus) — Passer la main à Ansible

Terraform crée la machine, Ansible la configure dans la durée. Générer l'inventaire depuis les sorties :

```bash
cat > inventaire.ini <<EOF
[web]
vm-web01 ansible_host=$(terraform output -raw ip_publique) ansible_user=azureuser ansible_ssh_private_key_file=~/.ssh/azure_lab
EOF
ansible -i inventaire.ini web -m ping
```

Le playbook nginx de 🛠️ [Automatiser ses VM avec Ansible](../transverse/tuto-ansible-homelab.md) s'applique ensuite tel quel avec `ansible-playbook -i inventaire.ini … -l web`.

## Étape 8 — Tout détruire

```bash
terraform destroy       # Plan: 0 to add, 0 to change, 8 to destroy. → yes
az group list -o table  # rg-site-tf a disparu
```

Le code reste dans Git : `terraform apply` recrée l'environnement à l'identique en quelques minutes.

## Ce que tu vois / ce que ça veut dire / action

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `subscription ID could not be determined` | Le fournisseur v4 exige l'abonnement | Renseigner `subscription_id` dans `terraform.tfvars` |
| `Error: building account … az login` | Pas de session Azure CLI | `az login`, vérifier `az account show` |
| `A resource with the ID … already exists` | La ressource existe déjà hors de l'état Terraform | La supprimer, ou l'importer : `terraform import` |
| `SkuNotAvailable` | Taille indisponible dans la région | Changer `vm_size` (ex. `Standard_B2ats_v2`) |
| `Error acquiring the state lock` | Un autre `apply` tourne, ou un précédent a planté | Attendre ; en dernier recours `terraform force-unlock <id>` |
| `-/+` sur la VM après une petite modif | Cet attribut ne peut pas changer sans recréer | Annuler si la VM a des données ; sinon accepter |

## Vérification finale

- [ ] `terraform plan` affiche `No changes` après l'`apply`
- [ ] La page web répond sur l'IP publique, SSH ne marche que depuis ton IP
- [ ] La modification de la règle NSG apparaît en `~`, la dérive est détectée puis corrigée
- [ ] `terraform destroy` ne laisse aucune ressource ; `*.tfstate` et `*.tfvars` ne sont pas dans Git

## Dépannage courant

- **Le site ne répond pas juste après `apply`** : cloud-init travaille encore. Sur la VM : `cloud-init status --wait`, puis `sudo cat /var/log/cloud-init-output.log`.
- **`file("~/.ssh/azure_lab.pub")` introuvable** : Terraform ne développe pas `~` seul, d'où `pathexpand()` ; vérifier que la clé existe.
- **Travailler à plusieurs** : déplacer l'état dans un compte de stockage avec un bloc `backend "azurerm"`, voir 🛠️ [Stocker et partager des fichiers dans Azure](tuto-stockage-azure-sas.md) pour créer le compte.

## Pour aller plus loin

- 🧠 [L'Infrastructure as Code](concept-iac.md) : état, dérive, outils
- 🧠 [Ansible](../transverse/concept-ansible.md) : la configuration après la création
- 🛠️ [Monter un VNet avec une VM publique et une VM privée](tuto-vnet-nsg-azure.md) : la même chose en commandes, pour comparer
