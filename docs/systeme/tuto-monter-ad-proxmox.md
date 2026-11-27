---
icon: material/wrench-outline
title: Monter un Active Directory sur une VM Proxmox
tags: [système, tuto, active directory, windows server, powershell, gpo, proxmox]
---

# 🛠️ Monter un Active Directory sur une VM Proxmox

!!! abstract "Objectif"
    Créer une VM Windows Server sur Proxmox, la promouvoir **contrôleur de domaine** `lab.local`, ranger les comptes dans des **OU**, créer utilisateurs et groupes, appliquer une première **GPO**, puis joindre un poste client au domaine.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 h 30 à 2 h (dont l'installation de Windows)

## Prérequis

- Comprendre domaine, OU, groupes et GPO → 🧠 [Active Directory](concept-active-directory.md)
- Comprendre le DNS et les enregistrements SRV → 🧠 [DHCP et DNS](../reseau/concept-dhcp-dns.md)
- Un hôte Proxmox avec deux ISO dans le stockage `local` : **Windows Server 2022** (version d'évaluation, 180 jours) et les pilotes **VirtIO** (`virtio-win.iso`)
- Un poste client Windows 10/11 **Pro ou Entreprise** (l'édition Famille ne peut pas rejoindre un domaine)

## Le plan qu'on va monter

| Élément | Valeur |
|---|---|
| Réseau du lab | `10.0.0.0/24` sur le pont `vmbr2` · passerelle `10.0.0.1` |
| Contrôleur de domaine | VM `srv-dc01` · `10.0.0.10` · 2 vCPU, 4 Go RAM, 60 Go |
| Domaine | `lab.local` (nom NetBIOS `LAB`) |
| Redirecteur DNS (Internet) | `1.1.1.1` |
| OU | `LAB` → `Utilisateurs` (`Compta`, `Informatique`), `Ordinateurs`, `Groupes` |
| Utilisateur de test | `alice` dans `Compta`, membre de `GG_Compta` |
| Poste client | `PC-01` · IP en DHCP ou fixe, **DNS = `10.0.0.10`** |

---

## Étape 1 — Créer la VM dans Proxmox

**Create VM**, puis onglet par onglet :

| Onglet | Réglage |
|---|---|
| General | *Name* : `srv-dc01` |
| OS | ISO Windows Server · *Type* : **Microsoft Windows**, version **11/2022/2025** · cocher **Add additional drive for VirtIO drivers** et choisir `virtio-win.iso` |
| System | *Machine* : `q35` · *BIOS* : **OVMF (UEFI)** + EFI disk · *TPM* : ajouter (v2.0) · **Qemu Agent** coché |
| Disks | *Bus* : **SCSI** (contrôleur VirtIO SCSI single) · 60 Go · *Discard* coché |
| CPU | 2 cœurs · *Type* : `host` |
| Memory | 4096 Mo |
| Network | *Bridge* : `vmbr2` · *Model* : **VirtIO (paravirtualized)** |

!!! tip "Pas de disque visible pendant l'installation ?"
    C'est normal avec un disque VirtIO : clique **Charger un pilote** → lecteur VirtIO → `vioscsi\2k22\amd64`. Le disque apparaît.

## Étape 2 — Installer Windows Server

1. Démarrer la VM, choisir **Windows Server 2022 Standard (expérience de bureau)**.
2. Installation personnalisée, charger le pilote (encadré ci-dessus), installer.
3. Définir le mot de passe du compte **Administrateur**.
4. Une fois sur le bureau : lancer `virtio-win-guest-tools.exe` depuis le lecteur VirtIO (pilote réseau + agent QEMU).

## Étape 3 — Nom et IP fixe

Un contrôleur de domaine a **toujours une IP fixe**, et se désigne **lui-même** comme DNS. PowerShell **en administrateur** :

```powershell
Get-NetAdapter                                  # repérer le nom de la carte (ex. "Ethernet")

New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.10 `
  -PrefixLength 24 -DefaultGateway 10.0.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1

Rename-Computer -NewName "srv-dc01" -Restart
```

!!! warning "Avant de continuer"
    Renommer **avant** la promotion : renommer un contrôleur de domaine après coup est bien plus délicat.

## Étape 4 — Installer le rôle et créer la forêt

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" `
  -InstallDns -SafeModeAdministratorPassword (Read-Host -AsSecureString "Mot de passe DSRM")
```

- `-InstallDns` installe le DNS et crée la zone `lab.local` avec les enregistrements SRV.
- Le mot de passe **DSRM** sert à réparer l'AD en mode restauration : le noter.
- Le serveur **redémarre seul**. On se reconnecte ensuite en `LAB\Administrateur`.

## Étape 5 — Vérifier le domaine et le DNS

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, PDCEmulator
Get-Service adws, kdc, netlogon, dns            # tous "Running"
Resolve-DnsName _ldap._tcp.dc._msdcs.lab.local -Type SRV
dcdiag /q                                       # aucune ligne = aucune erreur

# Pour résoudre les noms Internet
Set-DnsServerForwarder -IPAddress 1.1.1.1
Resolve-DnsName google.com
```

!!! info "Zone inverse"
    Utile pour `nslookup` et certains outils : `Add-DnsServerPrimaryZone -NetworkId "10.0.0.0/24" -ReplicationScope Domain`. Détails dans 🛠️ [Installer et configurer DHCP et DNS](../reseau/tuto-dhcp-dns.md), partie 2.

## Étape 6 — Créer l'arborescence d'OU

```powershell
$base = "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "LAB" -Path $base
New-ADOrganizationalUnit -Name "Utilisateurs" -Path "OU=LAB,$base"
New-ADOrganizationalUnit -Name "Ordinateurs"  -Path "OU=LAB,$base"
New-ADOrganizationalUnit -Name "Groupes"      -Path "OU=LAB,$base"
New-ADOrganizationalUnit -Name "Compta"       -Path "OU=Utilisateurs,OU=LAB,$base"
New-ADOrganizationalUnit -Name "Informatique" -Path "OU=Utilisateurs,OU=LAB,$base"

# Les nouveaux postes joints arriveront directement dans OU=Ordinateurs
redircmp "OU=Ordinateurs,OU=LAB,$base"

Get-ADOrganizationalUnit -Filter * | Select-Object DistinguishedName
```

Les OU sont créées **protégées contre la suppression accidentelle** : pour en supprimer une, décocher l'option dans ses propriétés (*Affichage → Fonctionnalités avancées*, onglet *Objet*).

## Étape 7 — Utilisateurs et groupes

```powershell
$ou = "OU=Compta,OU=Utilisateurs,OU=LAB,DC=lab,DC=local"

New-ADUser -Name "Alice Martin" -GivenName Alice -Surname Martin `
  -SamAccountName alice -UserPrincipalName alice@lab.local -Path $ou `
  -AccountPassword (Read-Host -AsSecureString "Mot de passe") `
  -ChangePasswordAtLogon $true -Enabled $true

New-ADGroup -Name "GG_Compta" -GroupScope Global -GroupCategory Security `
  -Path "OU=Groupes,OU=LAB,DC=lab,DC=local"
Add-ADGroupMember -Identity GG_Compta -Members alice

Get-ADGroupMember GG_Compta
```

!!! tip "Créer plusieurs comptes d'un coup"
    Préparer un `users.csv` (colonnes `Prenom;Nom;Login;OU`) puis :
    ```powershell
    Import-Csv users.csv -Delimiter ";" | ForEach-Object {
      New-ADUser -Name "$($_.Prenom) $($_.Nom)" -GivenName $_.Prenom -Surname $_.Nom `
        -SamAccountName $_.Login -UserPrincipalName "$($_.Login)@lab.local" `
        -Path "OU=$($_.OU),OU=Utilisateurs,OU=LAB,DC=lab,DC=local" `
        -AccountPassword (ConvertTo-SecureString "Azerty123!" -AsPlainText -Force) `
        -ChangePasswordAtLogon $true -Enabled $true
    }
    ```
    Mot de passe initial commun **uniquement en lab**, avec changement forcé à la première connexion.

## Étape 8 — Une première GPO

Exemple : interdire le Panneau de configuration aux utilisateurs de la Compta.

```powershell
New-GPO -Name "U_Compta_Restrictions" |
  New-GPLink -Target "OU=Compta,OU=Utilisateurs,OU=LAB,DC=lab,DC=local"

Set-GPRegistryValue -Name "U_Compta_Restrictions" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
  -ValueName NoControlPanel -Type DWord -Value 1

Get-GPInheritance -Target "OU=Compta,OU=Utilisateurs,OU=LAB,DC=lab,DC=local"
```

!!! tip "En graphique"
    Même chose dans `gpmc.msc` : clic droit sur l'OU *Compta* → **Créer un objet GPO dans ce domaine, et le lier ici**, puis *Modifier* → **Configuration utilisateur → Stratégies → Modèles d'administration → Panneau de configuration → Interdire l'accès au Panneau de configuration** : *Activé*.

Préfixer le nom de la GPO par `U_` (paramètres utilisateur) ou `C_` (ordinateur) aide à ne pas la lier à la mauvaise OU.

## Étape 9 — Joindre un poste client au domaine

Sur le poste client (VM Proxmox sur `vmbr2`, ou machine physique du même réseau), PowerShell **en administrateur** :

```powershell
# 1. Le seul DNS du poste doit être le contrôleur de domaine
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.10

# 2. Vérifier qu'il trouve le domaine
Resolve-DnsName lab.local
nltest /dsgetdc:lab.local

# 3. Joindre le domaine et redémarrer
Add-Computer -DomainName lab.local -NewName PC-01 -Credential LAB\Administrateur -Restart
```

Au redémarrage : **Autre utilisateur** → `LAB\alice` (ou `alice@lab.local`).

---

## ✅ Vérification finale

Sur le poste, connecté en `alice` :

```powershell
whoami                      # lab\alice
whoami /groups | findstr Compta
echo $env:LOGONSERVER       # \\SRV-DC01
gpupdate /force
gpresult /r                 # U_Compta_Restrictions dans « Objets Stratégie de groupe appliqués »
```

Sur le contrôleur de domaine :

```powershell
Get-ADComputer PC-01 | Select-Object DistinguishedName   # dans OU=Ordinateurs,OU=LAB
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| « Impossible de contacter un contrôleur de domaine » à la jonction | Le poste ne trouve pas les SRV | DNS du poste = `10.0.0.10` **uniquement** ; `ping 10.0.0.10` |
| `nltest /dsgetdc` échoue mais le ping passe | DNS du DC mal configuré | `Resolve-DnsName _ldap._tcp.dc._msdcs.lab.local -Type SRV` sur le DC ; `dcdiag` |
| « La relation d'approbation a échoué » | Le compte ordinateur et le poste ne sont plus synchronisés (souvent après restauration d'un snapshot) | `Test-ComputerSecureChannel -Repair -Credential LAB\Administrateur` |
| GPO absente de `gpresult /r` | Mauvaise OU, ou GPO Utilisateur liée à une OU d'ordinateurs | Vérifier où est l'objet et où est liée la GPO |
| GPO listée mais « refusée (filtrage de sécurité) » | Le filtrage ne contient pas l'utilisateur | Remettre *Utilisateurs authentifiés* dans le filtrage |
| Pas d'Internet sur le poste | Pas de redirecteur sur le DC | `Set-DnsServerForwarder -IPAddress 1.1.1.1` |

## 🔧 Dépannage courant

!!! question "Le poste a une IP mais `Resolve-DnsName lab.local` échoue ?"
    Le poste utilise encore le DNS de la box ou un DNS reçu en DHCP. Fixer le DNS à `10.0.0.10`, ou distribuer ce DNS par le DHCP → 🛠️ [Installer et configurer DHCP et DNS](../reseau/tuto-dhcp-dns.md).

!!! question "Pare-feu Proxmox : la jonction échoue alors que tout semble bon ?"
    Si la case **Firewall** est cochée sur la carte réseau des VM et que des règles Proxmox sont actives, les ports AD (53, 88, 389, 445, 135 et ports dynamiques) peuvent être bloqués. Décocher pour tester.

!!! question "J'ai cloné le client depuis un template Windows et la jonction pose problème ?"
    Deux clones Windows identiques ont le même **SID** machine. Passer `sysprep /generalize /oobe /shutdown` sur le modèle **avant** d'en faire un template (même logique que 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md) pour Linux).

!!! question "Snapshots du contrôleur de domaine ?"
    Pratiques en lab, mais restaurer un vieux snapshot du DC désynchronise les postes joints entre-temps (voir « relation d'approbation » ci-dessus). Prendre le snapshot **juste après** la promotion et la création des OU.

## Voir aussi

- 🧠 [Active Directory : domaine, OU et GPO](concept-active-directory.md)
- 🛠️ [Installer et configurer DHCP et DNS](../reseau/tuto-dhcp-dns.md) (ajouter le DHCP sur `srv-dc01`)
- 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)

??? quote "Sources"
    - [PowerShell : Install-ADDSForest](https://learn.microsoft.com/powershell/module/addsdeployment/install-addsforest)
    - [Proxmox VE : pilotes VirtIO pour Windows](https://pve.proxmox.com/wiki/Windows_VirtIO_Drivers)
    - [Wiki Samba : Samba comme contrôleur de domaine AD](https://wiki.samba.org/index.php/Setting_up_Samba_as_an_Active_Directory_Domain_Controller)
