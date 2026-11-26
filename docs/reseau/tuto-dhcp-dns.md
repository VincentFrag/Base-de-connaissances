---
title: Installer et configurer DHCP et DNS
tags: [réseau, tuto, dhcp, dns, windows server, powershell, linux]
---

# 🛠️ Installer et configurer DHCP et DNS

!!! abstract "Objectif"
    Mettre en place un serveur qui **distribue les adresses IP** (DHCP) et **résout les noms internes** (DNS) d'un réseau, sur Windows Server en PowerShell ou sur Linux avec dnsmasq.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 30 à 45 min

## Prérequis

- Comprendre DORA, étendues, réservations, enregistrements A/PTR/CNAME → 🧠 [DHCP et DNS](concept-dhcp-dns.md)
- Un serveur avec une **IP fixe** (jamais d'IP DHCP sur un serveur DHCP ou DNS)
- Pour Windows : PowerShell **en administrateur**

## Le plan qu'on va monter

| Élément | Valeur |
|---|---|
| Serveur DHCP + DNS | `srv01` · `192.168.20.10` (VLAN 20) |
| Domaine interne | `lab.local` |
| Étendue DHCP | VLAN 10 : `192.168.10.100` → `192.168.10.200` |
| Passerelle distribuée | `192.168.10.254` |
| Réservation | Imprimante `00-11-22-33-44-55` → `192.168.10.150` |
| Redirecteurs DNS (Internet) | `1.1.1.1` et `9.9.9.9` |

!!! info "Serveur et clients dans des VLAN différents"
    Ici le serveur est dans le VLAN 20 et les clients dans le VLAN 10 : il faut un **relais DHCP** sur la passerelle du VLAN 10 (`ip helper-address 192.168.20.10`) → voir 🛠️ [Créer et configurer des VLAN](tuto-creer-vlan.md), étape 5.

---

## Partie 1 — Le DHCP

=== "Windows Server (PowerShell)"

    **1. Installer le rôle**

    ```powershell
    Install-WindowsFeature DHCP -IncludeManagementTools
    ```

    **2. Finaliser l'installation**

    ```powershell
    # Créer les groupes de sécurité DHCP et redémarrer le service
    netsh dhcp add securitygroups
    Restart-Service DHCPServer

    # Si le serveur est membre d'un domaine AD : l'autoriser
    Add-DhcpServerInDC -DnsName srv01.lab.local -IPAddress 192.168.20.10

    # Masquer l'alerte « configuration post-installation » du Gestionnaire de serveur
    Set-ItemProperty -Path HKLM:\SOFTWARE\Microsoft\ServerManager\Roles\12 -Name ConfigurationState -Value 2
    ```

    **3. Créer l'étendue**

    ```powershell
    Add-DhcpServerv4Scope -Name "UTILISATEURS" `
      -StartRange 192.168.10.100 -EndRange 192.168.10.200 `
      -SubnetMask 255.255.255.0 -LeaseDuration 08:00:00 -State Active
    ```

    **4. Définir les options (passerelle, DNS, domaine)**

    ```powershell
    Set-DhcpServerv4OptionValue -ScopeId 192.168.10.0 `
      -Router 192.168.10.254 -DnsServer 192.168.20.10 -DnsDomain lab.local
    ```

    **5. Exclusions et réservations**

    ```powershell
    # Ne jamais distribuer .100 à .109 (équipements à IP fixe)
    Add-DhcpServerv4ExclusionRange -ScopeId 192.168.10.0 `
      -StartRange 192.168.10.100 -EndRange 192.168.10.109

    # Toujours donner .150 à l'imprimante
    Add-DhcpServerv4Reservation -ScopeId 192.168.10.0 `
      -IPAddress 192.168.10.150 -ClientId "00-11-22-33-44-55" -Description "Imprimante"
    ```

    **Vérifier**

    ```powershell
    Get-DhcpServerv4Scope
    Get-DhcpServerv4OptionValue -ScopeId 192.168.10.0
    Get-DhcpServerv4Lease -ScopeId 192.168.10.0      # baux distribués
    ```

=== "Linux (dnsmasq)"

    dnsmasq fait **DHCP et DNS dans un seul service léger** : idéal pour un lab ou un petit site.

    **1. Installer**

    ```bash
    sudo apt update && sudo apt install dnsmasq
    ```

    **2. Créer la configuration** dans `/etc/dnsmasq.d/lab.conf`

    ```ini
    # --- DHCP ---
    dhcp-range=192.168.10.100,192.168.10.200,255.255.255.0,8h
    dhcp-option=option:router,192.168.10.254
    dhcp-option=option:dns-server,192.168.20.10
    dhcp-option=option:domain-name,lab.local
    dhcp-host=00:11:22:33:44:55,imprimante,192.168.10.150
    dhcp-authoritative
    ```

    !!! note "Exclusions"
        dnsmasq ne gère pas d'exclusions : il suffit de **commencer la plage** après les adresses fixes (ex. `192.168.10.110`).

    **3. Tester et appliquer**

    ```bash
    sudo dnsmasq --test             # vérifie la syntaxe
    sudo systemctl restart dnsmasq
    sudo systemctl enable dnsmasq
    ```

    **Vérifier**

    ```bash
    systemctl status dnsmasq
    cat /var/lib/misc/dnsmasq.leases     # baux distribués
    journalctl -u dnsmasq -f             # suivre les DHCPDISCOVER / DHCPACK en direct
    ```

---

## Partie 2 — Le DNS

=== "Windows Server (PowerShell)"

    **1. Installer le rôle** (déjà présent si le serveur est contrôleur de domaine)

    ```powershell
    Install-WindowsFeature DNS -IncludeManagementTools
    ```

    **2. Créer la zone directe et la zone inverse**

    ```powershell
    # Serveur hors domaine : zone stockée dans un fichier
    Add-DnsServerPrimaryZone -Name "lab.local" -ZoneFile "lab.local.dns"
    Add-DnsServerPrimaryZone -NetworkId "192.168.20.0/24" -ZoneFile "20.168.192.in-addr.arpa.dns"
    Add-DnsServerPrimaryZone -NetworkId "192.168.10.0/24" -ZoneFile "10.168.192.in-addr.arpa.dns"
    ```

    !!! tip "Sur un contrôleur de domaine"
        La zone `lab.local` existe déjà. Pour les zones inverses, remplace `-ZoneFile …` par `-ReplicationScope Domain` : elles seront stockées et répliquées dans l'AD.

    **3. Ajouter des enregistrements**

    ```powershell
    # A + PTR automatique
    Add-DnsServerResourceRecordA -ZoneName lab.local -Name srv01 `
      -IPv4Address 192.168.20.10 -CreatePtr

    # Alias
    Add-DnsServerResourceRecordCName -ZoneName lab.local -Name intranet `
      -HostNameAlias srv01.lab.local
    ```

    **4. Configurer les redirecteurs** (pour résoudre Internet)

    ```powershell
    Set-DnsServerForwarder -IPAddress 1.1.1.1, 9.9.9.9
    ```

    **5. Mises à jour dynamiques par le DHCP** (les postes s'inscrivent seuls dans le DNS)

    ```powershell
    Set-DhcpServerv4DnsSetting -ScopeId 192.168.10.0 -DynamicUpdates Always `
      -DeleteDnsRROnLeaseExpiry $true
    ```

    **Vérifier**

    ```powershell
    Get-DnsServerZone
    Get-DnsServerResourceRecord -ZoneName lab.local
    Resolve-DnsName srv01.lab.local -Server 192.168.20.10
    Resolve-DnsName 192.168.20.10 -Server 192.168.20.10   # résolution inverse
    ```

=== "Linux (dnsmasq)"

    dnsmasq sert en DNS **le contenu de `/etc/hosts`** et le nom des clients DHCP.

    **1. Compléter** `/etc/dnsmasq.d/lab.conf`

    ```ini
    # --- DNS ---
    domain=lab.local
    local=/lab.local/          # ne jamais transférer lab.local vers Internet
    expand-hosts               # « srv01 » dans /etc/hosts devient aussi srv01.lab.local
    no-resolv                  # ignorer /etc/resolv.conf…
    server=1.1.1.1             # …et utiliser ces redirecteurs
    server=9.9.9.9
    cname=intranet.lab.local,srv01.lab.local
    ```

    **2. Déclarer les machines** dans `/etc/hosts`

    ```text
    192.168.20.10   srv01
    192.168.20.11   srv02
    ```

    Les enregistrements **PTR** (inverse) sont créés automatiquement.

    **3. Appliquer**

    ```bash
    sudo dnsmasq --test && sudo systemctl restart dnsmasq
    ```

    !!! warning "Ubuntu : « port 53 already in use »"
        `systemd-resolved` occupe déjà le port 53. Mettre `DNSStubListener=no` dans `/etc/systemd/resolved.conf`, puis `sudo systemctl restart systemd-resolved`.

    **Vérifier**

    ```bash
    dig @192.168.20.10 srv01.lab.local +short
    dig @192.168.20.10 -x 192.168.20.10 +short     # résolution inverse
    dig @192.168.20.10 intranet.lab.local
    dig @192.168.20.10 google.com +short           # redirecteurs OK
    ```

---

## ✅ Vérification finale, depuis un poste client du VLAN 10

=== "Windows"
    ```powershell
    ipconfig /release
    ipconfig /renew
    ipconfig /all              # IP dans la plage, passerelle .254, DNS 192.168.20.10, suffixe lab.local
    nslookup srv01.lab.local
    nslookup intranet
    ping google.com
    ```

=== "Linux"
    ```bash
    sudo dhclient -r && sudo dhclient
    ip a && ip r
    resolvectl status
    dig srv01.lab.local +short
    ping -c 2 google.com
    ```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| IP en `169.254.x.x` | Aucune réponse DHCP | Service démarré ? Relais DHCP sur le VLAN ? Pare-feu (UDP 67/68) ? |
| IP OK mais mauvaise passerelle ou DNS | Options DHCP incorrectes | Vérifier les options de l'étendue |
| `srv01.lab.local` ne se résout pas | Zone ou enregistrement absent | Vérifier la zone et l'enregistrement A |
| Noms internes OK, Internet KO | Redirecteurs absents ou bloqués | Configurer les redirecteurs, ouvrir UDP/TCP 53 sortant |
| `nslookup intranet` KO mais `intranet.lab.local` OK | Suffixe DNS non distribué | Option *domaine* du DHCP |

## 🔧 Dépannage courant

!!! question "Windows : « Le serveur DHCP n'est pas autorisé » ?"
    Dans un domaine AD, un serveur DHCP doit être **autorisé** : `Add-DhcpServerInDC`. Vérifier avec `Get-DhcpServerInDC`.

!!! question "`Set-DhcpServerv4OptionValue` refuse l'adresse DNS ?"
    La commande vérifie que le serveur DNS répond. S'il n'est pas encore prêt, ajouter `-Force`.

!!! question "Les postes gardent une ancienne IP ou un ancien nom ?"
    Bail encore valide côté client (`ipconfig /renew`), ou cache DNS (`ipconfig /flushdns`, et `Clear-DnsServerCache` côté serveur).

!!! question "Le DHCP marche dans le VLAN du serveur mais pas dans les autres ?"
    Il manque le **relais DHCP** (`ip helper-address`) sur la passerelle des autres VLAN, ou l'étendue correspondante sur le serveur.

## Voir aussi

- 🧠 [DHCP et DNS](concept-dhcp-dns.md)
- 🛠️ [Créer et configurer des VLAN](tuto-creer-vlan.md)
- 📋 [Mémo commandes réseau](memo-commandes-reseau.md)
