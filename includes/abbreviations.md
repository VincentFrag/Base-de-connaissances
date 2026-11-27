<!-- Sigles expliqués au survol sur tout le site (ajouté automatiquement à chaque page). Définitions complètes : docs/references/glossaire.md -->
*[ABAC]: Attribute-Based Access Control : droits selon des attributs (poste, horaire, étiquette)
*[ACL]: Access Control List : liste de règles qui autorise ou bloque du trafic
*[AD]: Active Directory : annuaire Microsoft des comptes, groupes et ordinateurs
*[AD DS]: Active Directory Domain Services : le rôle qui fait d'un serveur un contrôleur de domaine
*[APIPA]: Adresse 169.254.x.x auto-attribuée quand le DHCP ne répond pas
*[API]: Interface qui permet à un programme d'en piloter un autre
*[ARP]: Address Resolution Protocol : trouve l'adresse MAC qui correspond à une IP
*[BIOS]: Micrologiciel qui démarre l'ordinateur avant le système
*[CIDR]: Notation du masque en nombre de bits (/24 = 255.255.255.0)
*[CLI]: Command Line Interface : outil en ligne de commande
*[CNAME]: Enregistrement DNS d'alias vers un autre nom
*[CPU]: Processeur
*[CRLF]: Fin de ligne Windows (retour chariot + saut de ligne)
*[DC]: Domain Controller : serveur qui héberge l'Active Directory
*[DHCP]: Dynamic Host Configuration Protocol : distribue automatiquement IP, masque, passerelle et DNS
*[DMZ]: Zone réseau isolée pour les serveurs exposés à Internet
*[DNS]: Domain Name System : traduit un nom en adresse IP
*[DORA]: Discover, Offer, Request, Acknowledge : les 4 étapes du DHCP
*[FQDN]: Fully Qualified Domain Name : nom complet avec le domaine (srv-dc01.lab.local)
*[FTP]: File Transfer Protocol : transfert de fichiers non chiffré
*[GPO]: Group Policy Object : réglages appliqués automatiquement aux postes d'un domaine
*[HTTP]: HyperText Transfer Protocol : protocole du web
*[HTTPS]: HTTP chiffré par TLS (port 443)
*[IaaS]: Infrastructure as a Service : on loue des VM, du réseau, des disques
*[IaC]: Infrastructure as Code : l'infrastructure décrite dans des fichiers (Terraform)
*[ICMP]: Protocole des messages de contrôle réseau (ping, traceroute)
*[IDS]: Intrusion Detection System : détecte les attaques
*[IPS]: Intrusion Prevention System : bloque les attaques
*[IMAP]: Protocole de lecture des mails (ils restent sur le serveur)
*[IP]: Internet Protocol : adressage et acheminement des paquets
*[ISO]: Image d'un disque d'installation
*[JSON]: Format texte d'échange de données, utilisé par les API web
*[LAN]: Local Area Network : réseau local
*[LDAP]: Protocole d'accès à un annuaire (port 389)
*[LDAPS]: LDAP chiffré (port 636)
*[LXC]: Linux Containers : conteneurs système proposés par Proxmox
*[LVM]: Logical Volume Manager : gestion souple des volumes disque sous Linux
*[MAC]: Adresse physique d'une carte réseau
*[MFA]: Authentification à plusieurs facteurs
*[MTU]: Taille maximale d'un paquet sur un lien (1500 en Ethernet)
*[MX]: Enregistrement DNS du serveur de mail
*[NAS]: Stockage partagé en fichiers sur le réseau
*[NAT]: Network Address Translation : traduction d'adresses privées vers publiques
*[NFS]: Network File System : partage de fichiers Linux/Unix (port 2049)
*[NS]: Enregistrement DNS des serveurs qui font autorité sur une zone
*[NSG]: Network Security Group : pare-feu Azure d'un sous-réseau ou d'une carte réseau
*[NTFS]: Système de fichiers de Windows, avec ses propres droits d'accès
*[NTP]: Network Time Protocol : synchronisation de l'heure
*[NVA]: Network Virtual Appliance : VM qui joue le rôle de routeur ou de pare-feu
*[OIDC]: OpenID Connect : protocole d'authentification web basé sur OAuth 2.0
*[OS]: Système d'exploitation
*[OSI]: Modèle en 7 couches du fonctionnement d'un réseau
*[OU]: Organizational Unit : dossier de l'AD pour ranger objets et appliquer des GPO
*[PaaS]: Platform as a Service : plateforme managée (base, conteneur) sans gérer l'OS
*[PBS]: Proxmox Backup Server
*[PTR]: Enregistrement DNS inverse (IP vers nom)
*[PVE]: Proxmox Virtual Environment
*[RAID]: Combinaison de disques pour la performance ou la tolérance aux pannes
*[RAM]: Mémoire vive
*[RBAC]: Role-Based Access Control : droits accordés par rôle sur une portée
*[RDP]: Remote Desktop Protocol : bureau à distance Windows (port 3389)
*[RPO]: Recovery Point Objective : quantité de données qu'on accepte de perdre
*[RTO]: Recovery Time Objective : temps acceptable pour redémarrer
*[SaaS]: Software as a Service : logiciel loué, prêt à l'emploi
*[SAN]: Réseau de stockage qui fournit des disques bruts
*[SAS]: Shared Access Signature : lien signé à durée limitée vers un stockage Azure
*[SCP]: Secure Copy : copie de fichiers à travers SSH
*[SFTP]: Transfert de fichiers chiffré par SSH
*[SID]: Security Identifier : identifiant unique d'un compte ou d'une machine Windows
*[SLA]: Service Level Agreement : engagement de disponibilité d'un service
*[SMB]: Server Message Block : partage de fichiers Windows (port 445)
*[SMTP]: Simple Mail Transfer Protocol : envoi de mails
*[SNAT]: Source NAT : traduction de l'adresse source pour sortir sur Internet
*[SNMP]: Protocole de supervision des équipements (port 161)
*[SOA]: Enregistrement DNS d'informations de la zone
*[SRV]: Enregistrement DNS qui localise un service
*[SSH]: Secure Shell : connexion chiffrée en ligne de commande (port 22)
*[SSL]: Ancien nom de TLS, obsolète
*[SSO]: Single Sign-On : authentification unique
*[STP]: Spanning Tree Protocol : évite les boucles entre switchs
*[TCP]: Transmission Control Protocol : transport fiable, avec vérification de livraison
*[TFTP]: Trivial FTP : transfert simple en UDP (PXE, configs Cisco)
*[TLS]: Transport Layer Security : chiffrement des connexions
*[TTL]: Time To Live : durée de vie d'un paquet ou d'une réponse DNS en cache
*[UDP]: User Datagram Protocol : transport rapide, sans vérification de livraison
*[UEFI]: Version moderne du BIOS
*[URL]: Adresse d'une ressource web (https://nom:port/chemin)
*[VMSS]: Virtual Machine Scale Set : groupe de VM identiques qui s'adapte à la charge
*[VNet]: Virtual Network : réseau privé virtuel dans Azure
*[VSS]: Volume Shadow Copy Service : clichés instantanés d'un volume Windows
*[VLAN]: Virtual LAN : réseau local virtuel isolé sur un même switch
*[VM]: Machine virtuelle
*[VNC]: Prise en main graphique à distance
*[VPN]: Virtual Private Network : tunnel chiffré vers un réseau distant
*[WAN]: Wide Area Network : réseau étendu reliant des sites
*[WinRM]: PowerShell à distance (ports 5985/5986)
*[YAML]: Format texte de configuration fondé sur l'indentation (Ansible, Docker Compose)
