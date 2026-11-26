---
icon: material/flask-outline
title: "TP : Monter l'infra d'une PME sur Proxmox"
tags: [système, tp, proxmox, active directory, windows server, samba, partage de fichiers, dhcp, gpo, sauvegarde]
---

# 🧪 TP : Monter l'infra d'une PME sur Proxmox

!!! abstract "Ce que tu vas faire"
    Monter de bout en bout l'informatique d'une petite entreprise sur un seul hôte Proxmox : un **domaine Active Directory** avec DNS et DHCP, un **serveur de fichiers Linux** membre du domaine avec des partages par service, un **poste client** qui retrouve ses lecteurs réseau tout seul, et des **sauvegardes** planifiées dont on prouve qu'elles se restaurent.

**Niveau :** 🟡 Intermédiaire · **Durée :** 3 h à 4 h (dont les installations de Windows) · **Outil :** Proxmox VE

## Avant de commencer

Ce TP mobilise :

- 🛠️ [Monter un Active Directory sur une VM Proxmox](tuto-monter-ad-proxmox.md)
- 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md)
- 🛠️ [Gérer utilisateurs, groupes et permissions sous Linux](tuto-permissions-linux.md)
- 🛠️ [Installer et configurer DHCP et DNS](../reseau/tuto-dhcp-dns.md)
- 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](../transverse/tuto-proxmox-backup.md)

Il faut aussi : un Proxmox VE avec le pont interne `vmbr2` (`10.0.0.0/24`, passerelle `10.0.0.1` sur `pve01`), les ISO **Windows Server 2022**, **Windows 11 Pro** et **VirtIO**, le template Debian `tpl-debian01`, et un stockage de sauvegarde `sauvegardes` sur un disque séparé (étape 3a du tuto sauvegarde). Prévoir environ **12 Go de RAM** libres pour les trois VM.

!!! tip "Comment utiliser ce TP"
    Pour chaque tâche, essaie d'abord **seul**. Bloqué ? Ouvre l'**💡 Indice**. Toujours bloqué, ou pour vérifier ? Ouvre la **✅ Solution**.

---

## Le contexte

Le **cabinet Bernard** (expertise comptable, 15 salariés) travaille aujourd'hui avec des comptes locaux sur chaque PC et une clé USB qui circule pour les dossiers clients. La gérante veut :

- des **comptes centralisés** : chacun se connecte avec le même identifiant sur n'importe quel poste ;
- que les postes reçoivent leur configuration réseau **automatiquement** ;
- un **dossier commun** à tout le cabinet, et un **dossier Compta** réservé aux comptables ;
- que les lecteurs réseau **apparaissent tout seuls** à l'ouverture de session ;
- ne **rien perdre** si un serveur tombe : une sauvegarde chaque soir, et la preuve qu'elle se restaure.

Le budget est serré : un seul serveur physique, virtualisé avec Proxmox.

## Le cahier des charges

**Réseau :** `10.0.0.0/24` sur le pont `vmbr2` · **Passerelle :** `10.0.0.1` (`pve01`) · **Domaine :** `lab.local` (NetBIOS `LAB`)

| VM (ID) | Rôle | Système | IP | Ressources |
|---|---|---|---|---|
| `srv-dc01` (110) | Contrôleur de domaine, DNS, DHCP | Windows Server 2022 | `10.0.0.10` fixe | 2 vCPU, 4 Go, 60 Go |
| `srv-files01` (120) | Serveur de fichiers (Samba membre du domaine) | Debian, clone de `tpl-debian01` | `10.0.0.20` fixe | 2 vCPU, 2 Go, 32 Go |
| `PC-01` (130) | Poste client | Windows 11 Pro | DHCP | 2 vCPU, 4 Go, 64 Go |

**Annuaire :**

| Utilisateur | Service (OU) | Groupes |
|---|---|---|
| `alice` | Compta | `GG_Compta`, `GG_Personnel` |
| `bruno` | Compta | `GG_Compta`, `GG_Personnel` |
| `chloe` | Informatique | `GG_Informatique`, `GG_Personnel` |

**Partages sur `srv-files01` :**

| Partage | Dossier | Accès | Lecteur sur les postes |
|---|---|---|---|
| `Commun` | `/srv/partages/commun` | `GG_Personnel` : lecture/écriture | `S:` pour tout le monde |
| `Compta` | `/srv/partages/compta` | `GG_Compta` : lecture/écriture · les autres : rien | `P:` pour la Compta seulement |

**Règles :**

- Le DHCP distribue `10.0.0.100` → `10.0.0.199`, avec la passerelle `10.0.0.1` et le DNS `10.0.0.10` **uniquement**.
- Les serveurs ont une IP fixe et utilisent `srv-dc01` comme DNS.
- Sauvegarde de `srv-dc01` et `srv-files01` **chaque soir à 21 h**, rétention 7 jours / 4 semaines / 3 mois.

```
                    [ pve01 ] 10.0.0.1  (passerelle vers Internet)
                        │
  ──────────────────── vmbr2 · 10.0.0.0/24 ────────────────────
        │                       │                        │
  [ srv-dc01 ]           [ srv-files01 ]             [ PC-01 ]
   10.0.0.10              10.0.0.20                  DHCP .100–.199
   AD · DNS · DHCP        Samba : Commun, Compta     lecteurs S: et P:
        │                       │
        └───── vzdump chaque soir ─────▶ [ stockage « sauvegardes » ]
```

---

## Tâche 1 — Créer les VM dans Proxmox

Crée `srv-dc01` et `PC-01` depuis leurs ISO, et `srv-files01` par **clone complet** du template `tpl-debian01`. Toutes sur `vmbr2`.

??? tip "💡 Indice"
    Pour les deux Windows : machine `q35`, BIOS **OVMF (UEFI)** avec disque EFI et **TPM 2.0**, disque **SCSI** VirtIO, carte réseau **VirtIO**, et le second lecteur CD avec `virtio-win.iso`. Pour le clone : mode **Full Clone**, et démarrer les clones un par un tant qu'ils ont l'IP du template.

??? success "✅ Solution"
    **`srv-dc01` et `PC-01`** : **Create VM** avec les réglages du tableau de l'étape 1 du 🛠️ [tuto AD](tuto-monter-ad-proxmox.md) (ID `110` puis `130`, nom, ressources du cahier des charges, *Bridge* `vmbr2`). Installer Windows en chargeant le pilote `vioscsi\2k22\amd64` (Windows Server) ou `vioscsi\w11\amd64` (Windows 11), puis lancer `virtio-win-guest-tools.exe`.

    **`srv-files01`** depuis le **Shell** de `pve01` (remplacer `<ID-template>` par l'ID de `tpl-debian01`) :

    ```bash
    qm clone <ID-template> 120 --name srv-files01 --full
    qm set 120 --cores 2 --memory 2048 --net0 virtio,bridge=vmbr2
    qm config 120 | grep -E 'name|net0|memory'
    ```

    !!! warning "`qm set --net0` change l'adresse MAC"
        Sans précision, Proxmox en génère une nouvelle : c'est ce qu'on veut pour un clone. Ne démarre pas encore `srv-files01` : il sera personnalisé à la tâche 5.

## Tâche 2 — Promouvoir `srv-dc01` contrôleur de domaine

IP fixe, nom, rôle AD DS, forêt `lab.local`, et un redirecteur DNS pour qu'Internet fonctionne.

??? tip "💡 Indice"
    L'ordre compte : IP fixe et DNS `127.0.0.1` → renommer et redémarrer → `Install-WindowsFeature AD-Domain-Services` → `Install-ADDSForest` avec `-InstallDns`. Le redirecteur se pose avec `Set-DnsServerForwarder`.

??? success "✅ Solution"
    PowerShell **en administrateur** sur `srv-dc01` :

    ```powershell
    New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.10 `
      -PrefixLength 24 -DefaultGateway 10.0.0.1
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
    Rename-Computer -NewName "srv-dc01" -Restart
    ```

    Après le redémarrage :

    ```powershell
    Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
    Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" `
      -InstallDns -SafeModeAdministratorPassword (Read-Host -AsSecureString "Mot de passe DSRM")
    ```

    Après le second redémarrage, connecté en `LAB\Administrateur` :

    ```powershell
    Set-DnsServerForwarder -IPAddress 1.1.1.1
    Add-DnsServerPrimaryZone -NetworkId "10.0.0.0/24" -ReplicationScope Domain
    dcdiag /q                                   # aucune ligne = aucune erreur
    Resolve-DnsName google.com
    ```

## Tâche 3 — L'annuaire : OU, groupes et utilisateurs

Crée l'arborescence `LAB` → `Utilisateurs` (`Compta`, `Informatique`), `Ordinateurs`, `Groupes`, les trois groupes et les trois comptes du cahier des charges.

??? tip "💡 Indice"
    `New-ADOrganizationalUnit`, `New-ADGroup -GroupScope Global`, `New-ADUser` puis `Add-ADGroupMember`. Pense à `redircmp` pour que les postes joints arrivent dans `OU=Ordinateurs`. Une boucle évite de taper trois fois `New-ADUser`.

??? success "✅ Solution"
    ```powershell
    $base = "DC=lab,DC=local"
    $lab  = "OU=LAB,$base"
    New-ADOrganizationalUnit -Name "LAB" -Path $base
    "Utilisateurs","Ordinateurs","Groupes" | ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path $lab }
    "Compta","Informatique" | ForEach-Object { New-ADOrganizationalUnit -Name $_ -Path "OU=Utilisateurs,$lab" }
    redircmp "OU=Ordinateurs,$lab"

    "GG_Compta","GG_Informatique","GG_Personnel" | ForEach-Object {
      New-ADGroup -Name $_ -GroupScope Global -GroupCategory Security -Path "OU=Groupes,$lab"
    }

    $mdp = Read-Host -AsSecureString "Mot de passe initial"
    $comptes = @(
      @{ Login="alice"; Prenom="Alice"; Nom="Martin";  OU="Compta";       Groupe="GG_Compta" }
      @{ Login="bruno"; Prenom="Bruno"; Nom="Petit";   OU="Compta";       Groupe="GG_Compta" }
      @{ Login="chloe"; Prenom="Chloé"; Nom="Durand";  OU="Informatique"; Groupe="GG_Informatique" }
    )
    foreach ($c in $comptes) {
      New-ADUser -Name "$($c.Prenom) $($c.Nom)" -GivenName $c.Prenom -Surname $c.Nom `
        -SamAccountName $c.Login -UserPrincipalName "$($c.Login)@lab.local" `
        -Path "OU=$($c.OU),OU=Utilisateurs,$lab" -AccountPassword $mdp `
        -ChangePasswordAtLogon $true -Enabled $true
      Add-ADGroupMember -Identity $c.Groupe     -Members $c.Login
      Add-ADGroupMember -Identity GG_Personnel  -Members $c.Login
    }

    Get-ADGroupMember GG_Personnel | Select-Object SamAccountName
    ```

## Tâche 4 — Le DHCP sur `srv-dc01`

Installe le rôle DHCP, autorise-le dans l'AD et crée l'étendue du cahier des charges.

??? tip "💡 Indice"
    `Install-WindowsFeature DHCP`, puis `Add-DhcpServerInDC` (sans autorisation, un DHCP Windows membre d'un domaine refuse de distribuer), `Add-DhcpServerv4Scope` et `Set-DhcpServerv4OptionValue` avec `-Router`, `-DnsServer` et `-DnsDomain`.

??? success "✅ Solution"
    ```powershell
    Install-WindowsFeature DHCP -IncludeManagementTools
    netsh dhcp add securitygroups
    Restart-Service DHCPServer
    Add-DhcpServerInDC -DnsName srv-dc01.lab.local -IPAddress 10.0.0.10
    Set-ItemProperty -Path HKLM:\SOFTWARE\Microsoft\ServerManager\Roles\12 -Name ConfigurationState -Value 2

    Add-DhcpServerv4Scope -Name "LAB" -StartRange 10.0.0.100 -EndRange 10.0.0.199 `
      -SubnetMask 255.255.255.0 -LeaseDuration 08:00:00 -State Active
    Set-DhcpServerv4OptionValue -ScopeId 10.0.0.0 `
      -Router 10.0.0.1 -DnsServer 10.0.0.10 -DnsDomain lab.local

    Get-DhcpServerv4Scope
    Get-DhcpServerv4OptionValue -ScopeId 10.0.0.0
    ```

    !!! warning "Un seul DHCP sur `vmbr2`"
        Si `pve01` distribue déjà des adresses sur ce pont (dnsmasq, par exemple), le couper : avec deux serveurs DHCP, les postes reçoivent au hasard le DNS de l'un ou de l'autre, et la jonction au domaine échoue une fois sur deux.

## Tâche 5 — Préparer `srv-files01`

Démarre le clone et personnalise-le : nom, IP fixe `10.0.0.20`, identité unique. Puis fais-en un client DNS et NTP de `srv-dc01`.

??? tip "💡 Indice"
    La personnalisation est celle de l'étape 3 du 🛠️ [tuto clone](tuto-preparer-clone-vm.md). Pour rejoindre un domaine, Linux a besoin de trois choses : résoudre `lab.local` (`/etc/resolv.conf`), connaître son nom complet (`/etc/hosts`) et être **à l'heure** du contrôleur (Kerberos refuse plus de 5 minutes d'écart).

??? success "✅ Solution"
    Dans la **Console** de la VM 120 (vérifier que l'invite n'est pas `root@pve01`), en root :

    ```bash
    # Identité du clone (voir le tuto clone pour le détail)
    hostnamectl set-hostname srv-files01
    nano /etc/network/interfaces        # address 10.0.0.20/24 · gateway 10.0.0.1
    rm -f /etc/machine-id /var/lib/dbus/machine-id && systemd-machine-id-setup
    rm -f /etc/ssh/ssh_host_* && ssh-keygen -A

    # Nom complet : remplacer la ligne 127.0.1.1 par l'IP réelle
    sed -i '/^127.0.1.1/d' /etc/hosts
    echo "10.0.0.20   srv-files01.lab.local srv-files01" >> /etc/hosts

    # DNS = le contrôleur de domaine
    printf 'nameserver 10.0.0.10\nsearch lab.local\n' > /etc/resolv.conf

    reboot
    ```

    Après le redémarrage :

    ```bash
    apt update && apt install -y chrony
    sed -i 's/^pool /#pool /' /etc/chrony/chrony.conf
    echo "server 10.0.0.10 iburst" >> /etc/chrony/chrony.conf
    systemctl restart chrony

    hostname -f                         # srv-files01.lab.local
    host -t SRV _ldap._tcp.lab.local    # renvoie srv-dc01.lab.local
    chronyc sources                     # ligne ^* 10.0.0.10
    ```

## Tâche 6 — Joindre `srv-files01` au domaine

Installe Samba et winbind, configure-les en **membre du domaine** `LAB`, joins le domaine et vérifie que Linux « voit » les comptes AD.

??? tip "💡 Indice"
    Paquets `samba winbind libnss-winbind libpam-winbind krb5-user`. Dans `smb.conf` : `security = ADS`, `workgroup`, `realm`, et une correspondance d'identifiants (`idmap config LAB : backend = rid`). Jonction avec `net ads join -U Administrateur`, puis ajouter `winbind` aux lignes `passwd` et `group` de `/etc/nsswitch.conf`. Prends un **snapshot** juste avant la jonction : si ça tourne mal, tu reviens en arrière en dix secondes.

??? success "✅ Solution"
    Sur `pve01` : `qm snapshot 120 avant-jonction`.

    Sur `srv-files01` :

    ```bash
    apt install -y samba winbind libnss-winbind libpam-winbind krb5-user
    # À l'écran Kerberos, royaume par défaut : LAB.LOCAL (en majuscules)
    ```

    `/etc/krb5.conf` (remplacer tout le contenu) :

    ```ini
    [libdefaults]
        default_realm = LAB.LOCAL
        dns_lookup_realm = false
        dns_lookup_kdc = true
    ```

    `/etc/samba/smb.conf` : sauvegarder l'original (`mv /etc/samba/smb.conf /etc/samba/smb.conf.orig`) puis écrire :

    ```ini
    [global]
       workgroup = LAB
       realm = LAB.LOCAL
       security = ADS
       idmap config * : backend = tdb
       idmap config * : range = 3000-7999
       idmap config LAB : backend = rid
       idmap config LAB : range = 10000-999999
       winbind use default domain = yes
       winbind refresh tickets = yes
       template shell = /bin/bash
       vfs objects = acl_xattr
       map acl inherit = yes
    ```

    ```bash
    testparm -s                                  # « Loaded services file OK. »
    net ads join -U Administrateur               # mot de passe de LAB\Administrateur
    sed -i -E 's/^(passwd|group):(.*)/\1:\2 winbind/' /etc/nsswitch.conf
    systemctl restart smbd nmbd winbind

    wbinfo -t                 # « checking the trust secret ... succeeded »
    wbinfo -u                 # alice, bruno, chloe, administrateur…
    id alice                  # uid=1xxxx(alice) … groups=…(gg_compta),…(gg_personnel)
    ```

    Si `srv-files01` n'apparaît pas dans le DNS (`Resolve-DnsName srv-files01.lab.local` sur le DC), le créer à la main :
    `Add-DnsServerResourceRecordA -ZoneName lab.local -Name srv-files01 -IPv4Address 10.0.0.20`

    Une fois tout validé : `qm delsnapshot 120 avant-jonction`.

## Tâche 7 — Créer les partages `Commun` et `Compta`

Crée les deux dossiers avec les bons droits Linux, puis publie-les en SMB avec Samba. Les fichiers créés par un membre du groupe doivent rester modifiables par les autres membres.

??? tip "💡 Indice"
    Même logique que l'étape 3 du 🛠️ [tuto permissions](tuto-permissions-linux.md) : propriétaire `root`, groupe du service, droits `2770` (le **setgid** fait hériter le groupe). Côté Samba : `valid users = @GG_Compta`, `read only = no`, et des masques `create mask` / `directory mask` qui gardent l'écriture pour le groupe.

??? success "✅ Solution"
    ```bash
    mkdir -p /srv/partages/commun /srv/partages/compta
    chown root:GG_Personnel /srv/partages/commun
    chown root:GG_Compta    /srv/partages/compta
    chmod 2770 /srv/partages/commun /srv/partages/compta
    ls -ld /srv/partages/*    # drwxrws--- root gg_personnel / gg_compta
    ```

    Ajouter à la fin de `/etc/samba/smb.conf` :

    ```ini
    [Commun]
       path = /srv/partages/commun
       read only = no
       valid users = @GG_Personnel
       create mask = 0660
       directory mask = 2770

    [Compta]
       path = /srv/partages/compta
       read only = no
       valid users = @GG_Compta
       create mask = 0660
       directory mask = 2770
    ```

    ```bash
    testparm -s && systemctl reload smbd
    smbclient -L //srv-files01 -U alice          # Commun et Compta listés
    smbclient //srv-files01/Compta -U chloe -c ls   # NT_STATUS_ACCESS_DENIED : c'est voulu
    ```

## Tâche 8 — Joindre `PC-01` et distribuer les lecteurs par GPO

Le poste prend son IP en DHCP, rejoint le domaine, et à l'ouverture de session chaque utilisateur retrouve `S:` (Commun) et, s'il est de la Compta, `P:` (Compta).

??? tip "💡 Indice"
    Si le DHCP fonctionne, le poste a déjà le bon DNS : `Add-Computer` suffit. Les lecteurs se distribuent avec les **Préférences** de stratégie de groupe (*Mappages de lecteurs*), et le **ciblage au niveau de l'élément** permet de réserver `P:` au groupe `GG_Compta`. Lier la GPO à l'OU qui contient les **utilisateurs**, pas les ordinateurs.

??? success "✅ Solution"
    **Sur `PC-01`**, PowerShell en administrateur :

    ```powershell
    ipconfig /all                      # IP en 10.0.0.1xx, DNS 10.0.0.10
    Resolve-DnsName lab.local
    Add-Computer -DomainName lab.local -NewName PC-01 -Credential LAB\Administrateur -Restart
    ```

    **Sur `srv-dc01`**, créer et lier la GPO :

    ```powershell
    New-GPO -Name "U_Lecteurs_Reseau" |
      New-GPLink -Target "OU=Utilisateurs,OU=LAB,DC=lab,DC=local"
    ```

    Puis `gpmc.msc` → clic droit sur **U_Lecteurs_Reseau** → *Modifier* → **Configuration utilisateur → Préférences → Paramètres Windows → Mappages de lecteurs** → *Nouveau → Lecteur mappé* :

    | Réglage | Lecteur `S:` | Lecteur `P:` |
    |---|---|---|
    | Action | Mettre à jour | Mettre à jour |
    | Emplacement | `\\srv-files01\Commun` | `\\srv-files01\Compta` |
    | Reconnecter | ☑ · Libellé `Commun` | ☑ · Libellé `Compta` |
    | Lettre | `S` | `P` |
    | Onglet *Commun* | (rien) | ☑ **Ciblage au niveau de l'élément** → *Nouvel élément* → **Groupe de sécurité** → `LAB\GG_Compta` |

## Tâche 9 — Planifier les sauvegardes

Mets en place la sauvegarde nocturne de `srv-dc01` et `srv-files01` vers le stockage `sauvegardes`, avec la rétention du cahier des charges, et lance-la une première fois tout de suite.

??? tip "💡 Indice"
    **Datacenter → Backup → Add**, ou `pvesh create /cluster/backup` avec `--schedule`, `--vmid 110,120` et `--prune-backups`. Mode **Snapshot** pour ne pas couper les serveurs. Installe `qemu-guest-agent` dans `srv-files01` pour des sauvegardes cohérentes.

??? success "✅ Solution"
    Dans `srv-files01` : `apt install -y qemu-guest-agent && systemctl start qemu-guest-agent`, puis sur `pve01` : `qm set 120 --agent enabled=1` (redémarrer la VM pour que l'option s'applique). Sur `srv-dc01`, l'agent a été installé avec `virtio-win-guest-tools.exe` : `qm set 110 --agent enabled=1`.

    ```bash
    pvesh create /cluster/backup \
      --schedule "21:00" --storage sauvegardes --vmid 110,120 \
      --mode snapshot --compress zstd --enabled 1 \
      --prune-backups keep-daily=7,keep-weekly=4,keep-monthly=3
    pvesh get /cluster/backup                    # noter l'id de la tâche

    # Première sauvegarde immédiate (mêmes options que la tâche)
    vzdump 110 120 --storage sauvegardes --mode snapshot --compress zstd
    pvesm list sauvegardes --content backup      # deux archives récentes
    ```

    Dans l'interface, la même chose : **Datacenter → Backup → Add** (onglets *General* et *Retention*), puis **Run now**.

## Tâche 10 — Prouver que la sauvegarde se restaure

Scénario : alice dépose `bilan-2026.txt` dans `P:`, la sauvegarde passe, puis bruno supprime le fichier par erreur. Restaure `srv-files01` dans une **VM de test 920**, sans perturber la production, et vérifie que le fichier y est.

??? tip "💡 Indice"
    Refais une sauvegarde de la VM 120 **après** la création du fichier. Restaure-la sous un **nouvel ID** avec une MAC unique, et **débranche sa carte réseau** avant de la démarrer : sinon elle prend `10.0.0.20` en même temps que l'originale.

??? success "✅ Solution"
    1. Sur `PC-01` en `alice` : créer `P:\bilan-2026.txt`.
    2. Sur `pve01` :

        ```bash
        vzdump 120 --storage sauvegardes --mode snapshot --compress zstd
        ```

    3. Sur `PC-01` en `bruno` : supprimer `P:\bilan-2026.txt` (le partage réseau n'a pas de corbeille).
    4. Sur `pve01`, restaurer la dernière archive :

        ```bash
        pvesm list sauvegardes --content backup | grep qemu-120
        qmrestore /mnt/sauvegardes/dump/vzdump-qemu-120-<date>.vma.zst 920 \
          --storage local-lvm --unique 1
        qm set 920 --net0 virtio,bridge=vmbr2,link_down=1
        qm start 920
        ```

    5. **Console** de la VM 920 : `ls -l /srv/partages/compta/` → `bilan-2026.txt` est là. Pour le rendre à la Compta : `cat` le fichier et le recopier, ou (méthode propre) passer par la restauration de fichier d'un 🛠️ [Proxmox Backup Server](../transverse/tuto-proxmox-backup.md) (bonus du tuto).
    6. Nettoyer : `qm stop 920 && qm destroy 920 --purge`.

---

## ✅ Grille de validation

| # | Test | Depuis | Résultat attendu |
|---|---|---|---|
| 1 | `dcdiag /q` | `srv-dc01` | Aucune erreur |
| 2 | `Resolve-DnsName google.com` | `srv-dc01` | ✅ Réponse (redirecteur) |
| 3 | `Get-DhcpServerv4Lease -ScopeId 10.0.0.0` | `srv-dc01` | Un bail pour `PC-01` en `10.0.0.1xx` |
| 4 | `chronyc sources` | `srv-files01` | `^* 10.0.0.10` (synchronisé sur le DC) |
| 5 | `wbinfo -t` puis `id alice` | `srv-files01` | Secret validé · alice dans `gg_compta` et `gg_personnel` |
| 6 | `Get-ADComputer PC-01` | `srv-dc01` | Dans `OU=Ordinateurs,OU=LAB` |
| 7 | Session `LAB\alice`, `net use` | `PC-01` | `S:` et `P:` connectés |
| 8 | Session `LAB\chloe`, `net use` | `PC-01` | `S:` seulement |
| 9 | `dir \\srv-files01\Compta` | `PC-01` en `chloe` | ❌ Accès refusé |
| 10 | alice crée un fichier dans `S:`, chloe le modifie | `PC-01` | ✅ Modification enregistrée (setgid + masques) |
| 11 | `pvesh get /cluster/backup` et `pvesm list sauvegardes --content backup` | `pve01` | Tâche à 21:00 pour 110 et 120 · archives présentes |
| 12 | `ls /srv/partages/compta` dans la VM 920 restaurée | `pve01` (console) | `bilan-2026.txt` présent |

## 🔧 Si un test échoue

!!! question "`PC-01` a une IP en `169.254.x.x` ou un DNS qui n'est pas `10.0.0.10` (test 3) ?"
    DHCP non autorisé dans l'AD (`Get-DhcpServerInDC`), étendue inactive, ou un autre DHCP sur `vmbr2`. Vérifier aussi que la carte de `PC-01` est bien sur `vmbr2` et pas *Disconnect*.

!!! question "`net ads join` répond « Clock skew too great » ou « KDC has no support for encryption type » ?"
    La première erreur vient de l'heure : `chronyc sources` doit montrer `^*` devant `10.0.0.10`. Pour la seconde, vérifier `/etc/krb5.conf` (royaume **en majuscules**) et que `host -t SRV _kerberos._tcp.lab.local` répond.

!!! question "`wbinfo -u` fonctionne mais `id alice` répond « no such user » (test 5) ?"
    `winbind` manque dans `/etc/nsswitch.conf`, ou le service n'a pas été redémarré après la jonction : `grep winbind /etc/nsswitch.conf` puis `systemctl restart winbind`.

!!! question "`chown root:GG_Compta` répond « invalid group » ?"
    Même cause que ci-dessus : Linux ne voit pas encore les groupes AD. Tant que `getent group GG_Compta` ne répond rien, les partages ne peuvent pas être créés.

!!! question "Le lecteur `P:` n'apparaît pas pour alice (test 7) ?"
    `gpresult /r` doit lister **U_Lecteurs_Reseau**. Sinon : GPO liée à la mauvaise OU, ou session ouverte avant l'ajout au groupe (fermer la session et la rouvrir : l'appartenance aux groupes est lue à la connexion). Tester l'accès à la main : `net use P: \\srv-files01\Compta`.

!!! question "« Accès refusé » pour alice alors qu'elle est dans `GG_Compta` ?"
    Deux verrous doivent être ouverts : Samba (`valid users`) **et** Linux (`ls -ld /srv/partages/compta` doit montrer le groupe `gg_compta` avec `rws`). `smbclient //srv-files01/Compta -U alice -c ls` depuis le serveur aide à savoir lequel bloque.

!!! question "Chloé ne peut pas modifier le fichier d'alice dans `Commun` (test 10) ?"
    Le setgid manque (le `s` dans `drwxrws---`) ou `create mask` retire l'écriture au groupe. `chmod 2770` sur le dossier et `create mask = 0660` dans le partage.

!!! question "La VM 920 restaurée a fait tomber le partage de production ?"
    Sa carte réseau n'était pas débranchée : deux machines en `10.0.0.20`. `qm stop 920`, puis `qm set 920 --net0 virtio,bridge=vmbr2,link_down=1` avant de la redémarrer.

## Pour aller plus loin

- Ajouter un **second contrôleur de domaine** pour que l'AD survive à la panne de `srv-dc01`.
- Activer les **clichés instantanés** côté Samba (module `shadow_copy2` sur un volume ZFS ou LVM) pour que les utilisateurs restaurent eux-mêmes leurs fichiers via *Versions précédentes*.
- Envoyer une troisième copie vers un **Proxmox Backup Server** et tester la restauration d'un seul fichier (bonus du 🛠️ [tuto sauvegarde](../transverse/tuto-proxmox-backup.md)), pour respecter la 🧠 [règle 3-2-1](../transverse/concept-sauvegardes.md).
- Revoir les notions : 🧠 [Virtualisation](concept-virtualisation.md) · 🧠 [Active Directory](concept-active-directory.md) · 🧠 [Permissions Linux](concept-permissions-linux.md)
