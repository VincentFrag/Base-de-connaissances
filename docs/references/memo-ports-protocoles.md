---
icon: material/clipboard-text-outline
title: Mémo ports et protocoles
tags: [référence, mémo, ports, protocoles, réseau]
---

# 📋 Mémo ports et protocoles

!!! abstract "Usage"
    Retrouver en 10 secondes le port d'un service, s'il passe en TCP ou en UDP, et à quelle couche OSI il appartient. Utile pour écrire une règle de pare-feu ou comprendre pourquoi un service ne répond pas.
    Pour les couches : 🧠 [Le modèle OSI et TCP/IP](../reseau/concept-modele-osi-tcpip.md). Pour les règles : 🧠 [Pare-feu et NAT](../reseau/concept-pare-feu-nat.md).

!!! info "À savoir"
    - Les protocoles de cette page sont des protocoles **applicatifs** : ils sont tous en **couche 7** (Application) du modèle OSI. Ce qui change, c'est le transport utilisé en **couche 4** : TCP, UDP ou les deux.
    - **Ports connus** : 0 à 1023 (services standards, réservés à root sous Linux). **Ports enregistrés** : 1024 à 49151. **Ports éphémères** : 49152 à 65535, pris au hasard par le client pour sa connexion sortante.
    - Un port n'est qu'une convention : on peut faire tourner SSH sur le port 2222. Le pare-feu doit alors suivre.

## Accès distant et administration

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 22 | SSH / SFTP / SCP | TCP | Ligne de commande chiffrée, transfert de fichiers :material-arrow-right: [SSH](../reseau/concept-ssh.md) |
| 23 | Telnet | TCP | Ancien accès distant **non chiffré** : à éviter |
| 3389 | RDP | TCP + UDP | Bureau à distance Windows |
| 5900 | VNC | TCP | Prise en main graphique (aussi utilisé par la console Proxmox) |
| 5985 / 5986 | WinRM | TCP | PowerShell à distance (HTTP / HTTPS) |
| 8006 | Proxmox VE | TCP | Interface web de Proxmox (`https://pve01:8006`) |

## Web

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 80 | HTTP | TCP | Web non chiffré (souvent redirigé vers 443) |
| 443 | HTTPS | TCP (UDP pour HTTP/3) | Web chiffré par TLS |
| 8080 / 8443 | HTTP / HTTPS alternatifs | TCP | Applications web, proxys, interfaces d'admin |

## Résolution de noms, adressage et heure

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 53 | DNS | UDP (TCP pour les grosses réponses et transferts de zone) | Traduire un nom en IP :material-arrow-right: [DHCP et DNS](../reseau/concept-dhcp-dns.md) |
| 67 / 68 | DHCP | UDP | 67 côté serveur, 68 côté client |
| 69 | TFTP | UDP | Transfert simple (démarrage PXE, sauvegarde de config Cisco) |
| 123 | NTP | UDP | Synchroniser l'heure |
| 5353 | mDNS | UDP | Découverte de noms en `.local` sans serveur DNS |

## Active Directory et Windows

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 88 | Kerberos | TCP + UDP | Authentification du domaine |
| 135 | RPC | TCP | Appels de procédure distants (outils d'admin Windows) |
| 137-139 | NetBIOS | UDP + TCP | Anciens noms et partages Windows |
| 389 | LDAP | TCP + UDP | Interroger l'annuaire |
| 636 | LDAPS | TCP | LDAP chiffré |
| 445 | SMB | TCP | Partage de fichiers et imprimantes, accès à SYSVOL (GPO) |
| 464 | Kerberos (mot de passe) | TCP + UDP | Changement de mot de passe |
| 3268 / 3269 | Catalogue global | TCP | Recherche dans toute la forêt AD |

!!! tip "Joindre un poste au domaine"
    Un poste qui rejoint `lab.local` doit pouvoir joindre `srv-dc01` (10.0.0.10) au minimum sur **53, 88, 135, 389, 445 et 464**. Si un pare-feu bloque l'un d'eux, la jonction échoue. :material-arrow-right: [Monter un Active Directory](../systeme/tuto-monter-ad-proxmox.md)

## Partage de fichiers et stockage

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 20 / 21 | FTP | TCP | Transfert de fichiers non chiffré (21 = commandes, 20 = données) |
| 445 | SMB / CIFS | TCP | Partages Windows (et Samba sous Linux) |
| 2049 | NFS | TCP + UDP | Partages Linux/Unix |
| 111 | rpcbind | TCP + UDP | Annuaire des ports RPC, requis par NFSv3 |
| 3260 | iSCSI | TCP | Disques réseau (SAN) |
| 873 | rsync | TCP | Synchronisation de fichiers (mode démon) |
| 8007 | Proxmox Backup Server | TCP | Interface et stockage PBS :material-arrow-right: [Sauvegarder avec Proxmox](../transverse/tuto-proxmox-backup.md) |

## Messagerie

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 25 | SMTP | TCP | Envoi entre serveurs de mail |
| 587 | SMTP Submission | TCP | Envoi depuis un client mail (avec STARTTLS) |
| 465 | SMTPS | TCP | Envoi chiffré en TLS direct |
| 110 / 995 | POP3 / POP3S | TCP | Récupérer ses mails (ils quittent le serveur) |
| 143 / 993 | IMAP / IMAPS | TCP | Lire ses mails (ils restent sur le serveur) |

## Supervision et journaux

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 161 | SNMP | UDP | Interroger un équipement (état, trafic) |
| 162 | SNMP Trap | UDP | Alertes envoyées par l'équipement |
| 514 | Syslog | UDP (TCP possible) | Centraliser les journaux |
| 10050 / 10051 | Zabbix | TCP | Agent (10050) et serveur (10051) Zabbix |
| 3000 | Grafana | TCP | Tableaux de bord |
| 9090 | Prometheus | TCP | Collecte de métriques |

## VPN et tunnels

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 51820 | WireGuard | UDP | VPN moderne et simple |
| 1194 | OpenVPN | UDP (TCP possible) | VPN classique |
| 500 / 4500 | IPsec (IKE / NAT-T) | UDP | VPN site à site |
| 1701 | L2TP | UDP | Souvent couplé à IPsec |

## Bases de données et applications

| Port | Protocole | Transport | Usage |
|---|---|---|---|
| 1433 | Microsoft SQL Server | TCP | Base de données Microsoft |
| 3306 | MySQL / MariaDB | TCP | Base de données (WordPress…) |
| 5432 | PostgreSQL | TCP | Base de données |
| 6379 | Redis | TCP | Cache en mémoire |
| 27017 | MongoDB | TCP | Base de données NoSQL |
| 2375 / 2376 | API Docker | TCP | Pilotage de Docker à distance (2375 non chiffré : jamais exposé) |

## Et sans port ?

Certains protocoles n'utilisent pas TCP ni UDP, donc **pas de port** : on les filtre par leur nom ou leur numéro de protocole IP.

| Protocole | Couche OSI | Usage |
|---|---|---|
| ICMP | 3 (Réseau) | `ping`, `traceroute`, messages d'erreur réseau |
| ARP | 2-3 | Trouver la MAC qui correspond à une IP |
| ESP (protocole 50) | 3 | Données chiffrées d'IPsec |
| GRE (protocole 47) | 3 | Tunnels génériques |
| OSPF (protocole 89) | 3 | Routage dynamique entre routeurs |
| STP | 2 (Liaison) | Évite les boucles entre switchs |
| 802.1Q | 2 (Liaison) | Étiquette VLAN sur les ports trunk :material-arrow-right: [VLAN](../reseau/concept-vlan.md) |

## Tester un port

=== "Windows"

    ```powershell
    # Le port 443 de srv-web01 répond-il ?
    Test-NetConnection srv-web01 -Port 443

    # Quels ports écoutent sur ma machine, et quel programme ?
    netstat -ano | findstr LISTENING
    Get-NetTCPConnection -State Listen

    # Ouvrir un port dans le pare-feu Windows
    New-NetFirewallRule -DisplayName "HTTPS" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
    ```

=== "Linux"

    ```bash
    # Le port 443 de srv-web01 répond-il ?
    nc -zv srv-web01 443

    # Quels ports écoutent sur ma machine, et quel programme ?
    sudo ss -tulpn

    # Scanner les ports ouverts d'une machine (à faire seulement sur ton lab)
    nmap -sV 10.0.0.31

    # Ouvrir un port (Debian avec ufw / Rocky avec firewalld)
    sudo ufw allow 443/tcp
    sudo firewall-cmd --add-port=443/tcp --permanent && sudo firewall-cmd --reload
    ```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `TcpTestSucceeded : True` / `succeeded!` | Le service écoute et le chemin est ouvert | Le problème est ailleurs (appli, identifiants) |
| `Connection refused` | La machine répond mais rien n'écoute sur ce port | Vérifier que le service tourne (`systemctl status`) |
| Délai dépassé (*timeout*) | Un pare-feu bloque en silence, ou la machine est injoignable | Tester le `ping`, puis les règles du pare-feu |

!!! warning "Piège : UDP"
    `Test-NetConnection` et `nc -zv` testent en **TCP**. Pour un service UDP (DNS, DHCP, WireGuard), teste le service lui-même : `nslookup lab.local 10.0.0.10` pour le DNS, par exemple.
