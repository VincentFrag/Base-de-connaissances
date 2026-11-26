---
icon: material/book-alphabet
title: Glossaire de l'informatique
tags: [référence, glossaire, vocabulaire]
---

# 📖 Glossaire de l'informatique

!!! abstract "Comment l'utiliser"
    Les termes utiles à connaître en administration systèmes et réseaux, classés de A à Z. Chaque terme a une définition courte, son domaine et, quand il existe, un lien :material-arrow-right: vers l'article du site qui l'explique en détail.
    Utilise le sommaire à droite pour sauter à une lettre, ou la recherche (touche S).

    Les sigles sont aussi expliqués **au survol** partout sur le site : passe la souris sur DHCP, VLAN ou NAT dans n'importe quel article.

**Domaines** : 🌐 Réseau · 🖥️ Système · 🔒 Sécurité · 📦 Virtualisation · 🌍 Web · 💾 Stockage

## A

**ACL** (*Access Control List*) · 🌐🔒
: Liste de règles qui autorise ou bloque du trafic (sur un routeur, un pare-feu) ou l'accès à une ressource (sur un fichier). Les règles sont lues dans l'ordre, la première qui correspond gagne. :material-arrow-right: [Pare-feu et NAT](../reseau/concept-pare-feu-nat.md)

**AD** (*Active Directory*) · 🖥️
: Annuaire de Microsoft qui centralise les comptes, les groupes, les ordinateurs et les règles d'un réseau d'entreprise. :material-arrow-right: [Active Directory](../systeme/concept-active-directory.md)

**Adresse IP** · 🌐
: Numéro qui identifie une machine sur un réseau, comme une adresse postale (ex. `10.0.0.10`). :material-arrow-right: [Adressage IP](../reseau/concept-adressage-ip.md)

**Adresse MAC** · 🌐
: Identifiant physique gravé dans une carte réseau (ex. `00:1A:2B:3C:4D:5E`). Sert à livrer les trames dans un même réseau local (couche 2).

**APIPA** · 🌐
: Adresse `169.254.x.x` qu'un poste Windows s'attribue tout seul quand il ne trouve pas de serveur DHCP. Si tu la vois, le DHCP ne répond pas. :material-arrow-right: [Diagnostiquer une connexion](../reseau/tuto-diagnostic-connexion.md)

**API** · 🌍
: Interface qui permet à un programme d'en piloter un autre (ex. créer une VM Proxmox par un script au lieu de cliquer).

**ARP** (*Address Resolution Protocol*) · 🌐
: Protocole qui trouve l'adresse MAC correspondant à une adresse IP dans le réseau local. Commande `arp -a`.

**Authentification** · 🔒
: Prouver qui on est (mot de passe, clé, carte). À ne pas confondre avec l'**autorisation**, qui dit ce qu'on a le droit de faire.

## B

**Bail DHCP** (*lease*) · 🌐
: Durée pendant laquelle une adresse IP est prêtée à un poste par le serveur DHCP. :material-arrow-right: [DHCP et DNS](../reseau/concept-dhcp-dns.md)

**Bande passante** · 🌐
: Quantité maximale de données qu'un lien peut transporter par seconde (ex. 1 Gbit/s).

**Bastion** (ou hôte de rebond) · 🔒
: Machine exposée par laquelle on passe obligatoirement pour atteindre les serveurs d'un réseau interne. :material-arrow-right: [SSH par rebond](../reseau/tuto-ssh-rebond.md)

**BIOS / UEFI** · 🖥️
: Micrologiciel qui démarre l'ordinateur avant le système d'exploitation. L'UEFI est la version moderne du BIOS.

**Broadcast** · 🌐
: Message envoyé à toutes les machines d'un réseau (adresse en `.255` dans un `/24`). Un VLAN limite la portée des broadcasts.

## C

**Certificat** · 🔒
: Fichier qui prouve l'identité d'un serveur et permet de chiffrer la connexion (le cadenas HTTPS). Délivré par une autorité de certification (CA).

**Chiffrement** · 🔒
: Rendre des données illisibles sans la bonne clé. Symétrique (une seule clé) ou asymétrique (une clé publique et une clé privée).

**CIDR** · 🌐
: Notation du masque par le nombre de bits réseau : `/24` équivaut à `255.255.255.0`. :material-arrow-right: [Sous-réseaux](../reseau/tuto-sous-reseaux.md)

**Client / serveur** · 🌐
: Le client demande un service (navigateur, poste), le serveur le rend (site web, DNS, partage de fichiers).

**Clone (lié / complet)** · 📦
: Copie d'une VM. Le clone lié dépend du template et prend peu de place, le clone complet est indépendant. :material-arrow-right: [Virtualisation](../systeme/concept-virtualisation.md)

**Cluster** · 📦
: Plusieurs serveurs qui travaillent ensemble comme un seul, pour la puissance ou la haute disponibilité.

**Conteneur** · 📦
: Environnement isolé qui partage le noyau de l'hôte : plus léger qu'une VM (Docker, LXC). :material-arrow-right: [VM ou conteneur LXC](../systeme/concept-virtualisation.md)

**Contrôleur de domaine** (*DC*) · 🖥️
: Serveur qui héberge l'Active Directory et authentifie les utilisateurs du domaine (ex. `srv-dc01`). :material-arrow-right: [Active Directory](../systeme/concept-active-directory.md)

**cron** · 🖥️
: Planificateur de tâches de Linux : lance une commande à heure fixe (ex. une sauvegarde chaque nuit).

## D

**Daemon / service** · 🖥️
: Programme qui tourne en arrière-plan en permanence (`sshd`, serveur DNS…). Géré par `systemctl` sous Linux, par `services.msc` sous Windows.

**DHCP** · 🌐
: Service qui distribue automatiquement l'IP, le masque, la passerelle et le DNS aux postes. Échange en 4 étapes : DORA. :material-arrow-right: [DHCP et DNS](../reseau/concept-dhcp-dns.md)

**DMZ** · 🔒
: Zone réseau à part où l'on place les serveurs accessibles depuis Internet, pour les isoler du réseau interne.

**DNS** · 🌐
: Service qui traduit un nom (`srv-web01.lab.local`) en adresse IP, comme un annuaire téléphonique. :material-arrow-right: [DHCP et DNS](../reseau/concept-dhcp-dns.md)

**Domaine** · 🖥️🌐
: En Active Directory, ensemble de machines et de comptes gérés ensemble (`lab.local`). En DNS, une partie d'un nom (`exemple.fr`).

**DORA** · 🌐
: Les 4 étapes du DHCP : Discover, Offer, Request, Acknowledge. :material-arrow-right: [DHCP et DNS](../reseau/concept-dhcp-dns.md)

## E

**Encapsulation** · 🌐
: Chaque couche réseau ajoute son en-tête autour des données avant de les passer à la couche du dessous. :material-arrow-right: [Modèle OSI](../reseau/concept-modele-osi-tcpip.md)

**Ethernet** · 🌐
: Technologie standard des réseaux locaux filaires (câbles RJ45, trames, adresses MAC).

## F

**FQDN** · 🌐
: Nom complet d'une machine, avec le domaine : `srv-dc01.lab.local`.

**FTP / SFTP** · 🌐
: Protocoles de transfert de fichiers. FTP n'est pas chiffré, SFTP passe par SSH.

## G

**GPO** (*Group Policy Object*) · 🖥️
: Ensemble de réglages appliqués automatiquement aux postes ou aux utilisateurs d'un domaine (fond d'écran, lecteurs réseau, sécurité). :material-arrow-right: [Active Directory](../systeme/concept-active-directory.md)

**Groupe** · 🖥️
: Ensemble de comptes auxquels on donne les mêmes droits d'un coup. :material-arrow-right: [Permissions Linux](../systeme/concept-permissions-linux.md)

## H

**Haute disponibilité** (*HA*) · 📦
: Organisation qui garde un service en marche même si un serveur tombe (redondance, bascule automatique).

**Hash** (empreinte) · 🔒
: Résumé unique et non réversible d'une donnée (SHA-256). Sert à stocker les mots de passe et à vérifier qu'un fichier n'a pas été modifié.

**Hôte** · 📦🌐
: En virtualisation, la machine physique qui fait tourner les VM (ex. `pve01`). En réseau, toute machine qui a une IP.

**HTTP / HTTPS** · 🌍
: Protocole du web. HTTPS est la version chiffrée par TLS (port 443). :material-arrow-right: [Codes HTTP](memo-references-reseau.md#codes-http)

**Hyperviseur** · 📦
: Logiciel qui fait tourner des machines virtuelles. Type 1 directement sur le matériel (Proxmox, ESXi, Hyper-V), type 2 sur un OS (VirtualBox). :material-arrow-right: [Virtualisation](../systeme/concept-virtualisation.md)

## I

**ICMP** · 🌐
: Protocole des messages de contrôle réseau, utilisé par `ping` et `traceroute`.

**IDS / IPS** · 🔒
: Système qui détecte (IDS) ou bloque (IPS) les attaques en analysant le trafic.

**Inode** · 🖥️
: Fiche qui décrit un fichier sous Linux (propriétaire, droits, emplacement sur le disque).

**IPv4 / IPv6** · 🌐
: Versions du protocole IP. IPv4 sur 32 bits (`10.0.0.10`), IPv6 sur 128 bits (`fe80::1`).

**ISO** · 📦
: Image d'un disque d'installation (ex. l'ISO Debian qu'on monte dans une VM pour l'installer).

## K

**Kerberos** · 🔒🖥️
: Protocole d'authentification par tickets utilisé par Active Directory : on s'identifie une fois et on reçoit des tickets pour accéder aux services.

**Kernel** (noyau) · 🖥️
: Cœur du système d'exploitation, qui fait le lien entre les programmes et le matériel.

## L

**LAN / WAN** · 🌐
: LAN = réseau local (un bâtiment). WAN = réseau étendu qui relie des sites (Internet en est un).

**Latence** · 🌐
: Temps que met un paquet pour aller et revenir (le `time=` du `ping`), en millisecondes.

**LDAP** · 🖥️
: Protocole pour interroger et modifier un annuaire comme Active Directory (port 389, 636 en LDAPS).

**Load balancer** (répartiteur de charge) · 🌐
: Équipement ou logiciel qui répartit les visiteurs entre plusieurs serveurs (ex. HAProxy sur `srv-lb01`).

**Loopback** · 🌐
: Adresse qui désigne la machine elle-même : `127.0.0.1` (ou `localhost`).

**LVM** · 💾
: Gestionnaire de volumes Linux : regroupe des disques et découpe l'espace en volumes faciles à agrandir.

**LXC** · 📦
: Conteneurs système de Linux, proposés par Proxmox à côté des VM. :material-arrow-right: [Virtualisation](../systeme/concept-virtualisation.md)

## M

**Masque de sous-réseau** · 🌐
: Indique quelle partie de l'IP désigne le réseau et quelle partie désigne la machine (`255.255.255.0` = `/24`). :material-arrow-right: [Adressage IP](../reseau/concept-adressage-ip.md)

**MFA** · 🔒
: Authentification à plusieurs facteurs : mot de passe plus un second élément (code sur le téléphone, clé physique).

**Montage** (*mount*) · 💾
: Rattacher un disque ou un partage réseau à un dossier pour pouvoir l'utiliser (`mount`, `/etc/fstab`).

**MTU** · 🌐
: Taille maximale d'un paquet sur un lien (1500 octets en Ethernet). Une MTU trop grande cause des coupures bizarres, souvent avec les VPN.

## N

**NAS / SAN** · 💾
: NAS = stockage partagé en fichiers sur le réseau (SMB, NFS). SAN = réseau de stockage qui fournit des disques bruts (iSCSI).

**NAT** · 🌐
: Traduction d'adresses : les postes privés sortent sur Internet avec l'IP publique du routeur. :material-arrow-right: [Pare-feu et NAT](../reseau/concept-pare-feu-nat.md)

**NetBIOS** · 🖥️
: Ancien système de noms courts de Windows (ex. `LAB` pour le domaine `lab.local`).

**NFS** · 💾
: Protocole de partage de fichiers en réseau propre au monde Linux/Unix (port 2049).

**NTFS** · 🖥️💾
: Système de fichiers de Windows. Ses droits (lecture, modification, contrôle total), hérités de dossier en dossier, s'ajoutent à ceux du partage SMB : l'accès réel est le plus restrictif des deux. :material-arrow-right: [TP infra PME](../systeme/tp-infra-pme-proxmox.md)

**NTP** · 🌐
: Protocole qui synchronise l'heure des machines. Indispensable pour Kerberos et les journaux.

## O

**OS** (système d'exploitation) · 🖥️
: Logiciel de base qui gère la machine : Windows, Linux (Debian, Rocky…), macOS.

**OSI** (modèle) · 🌐
: Modèle en 7 couches qui découpe le fonctionnement d'un réseau, du câble (1) à l'application (7). :material-arrow-right: [Modèle OSI et TCP/IP](../reseau/concept-modele-osi-tcpip.md)

**OU** (*Organizational Unit*) · 🖥️
: Dossier de l'Active Directory pour ranger comptes et ordinateurs et leur appliquer des GPO. :material-arrow-right: [Active Directory](../systeme/concept-active-directory.md)

## P

**Paquet / trame** · 🌐
: Unités de données : la **trame** circule en couche 2 (adresses MAC), le **paquet** en couche 3 (adresses IP).

**Pare-feu** (*firewall*) · 🔒
: Filtre qui laisse passer ou bloque le trafic selon des règles (IP, port, protocole). :material-arrow-right: [Pare-feu et NAT](../reseau/concept-pare-feu-nat.md)

**Partition** · 💾
: Découpage d'un disque en zones indépendantes, chacune avec son système de fichiers.

**Passerelle** (*gateway*) · 🌐
: Routeur par lequel un poste sort de son réseau (souvent l'adresse en `.1` ou `.254`). :material-arrow-right: [Adressage IP](../reseau/concept-adressage-ip.md)

**Permissions** (`rwx`) · 🖥️
: Droits de lecture, écriture et exécution sur un fichier, pour le propriétaire, le groupe et les autres. :material-arrow-right: [Permissions Linux](../systeme/concept-permissions-linux.md)

**Ping** · 🌐
: Commande qui vérifie qu'une machine répond (ICMP). :material-arrow-right: [Mémo commandes réseau](../reseau/memo-commandes-reseau.md)

**Port** · 🌐
: Numéro qui identifie un service sur une machine (22 pour SSH, 443 pour HTTPS). :material-arrow-right: [Mémo ports et protocoles](memo-ports-protocoles.md)

**Port access / trunk** · 🌐
: Sur un switch, un port **access** appartient à un seul VLAN, un port **trunk** transporte plusieurs VLAN étiquetés. :material-arrow-right: [VLAN](../reseau/concept-vlan.md)

**Protocole** · 🌐
: Ensemble de règles qu'utilisent deux machines pour se comprendre (HTTP, DNS, SSH…).

**Proxy / reverse proxy** · 🌐
: Le proxy sort sur Internet à la place des postes. Le reverse proxy reçoit les visiteurs à la place des serveurs web.

## R

**RAID** · 💾
: Combinaison de plusieurs disques pour la performance ou la tolérance aux pannes (RAID 1 = miroir, RAID 5 = parité). Un RAID n'est **pas** une sauvegarde.

**RDP** · 🖥️
: Bureau à distance de Windows (port 3389).

**Redirection de port** (*port forwarding*) · 🌐
: Règle de NAT qui renvoie un port de l'IP publique vers un serveur interne. :material-arrow-right: [Pare-feu et NAT](../reseau/concept-pare-feu-nat.md)

**root** · 🖥️
: Super-administrateur de Linux, qui a tous les droits. On l'utilise via `sudo`. :material-arrow-right: [Permissions Linux](../systeme/concept-permissions-linux.md)

**Routeur** · 🌐
: Équipement qui relie des réseaux différents et choisit le chemin des paquets (couche 3).

**RPO / RTO** · 💾
: RPO = quantité de données qu'on accepte de perdre. RTO = temps acceptable pour redémarrer. :material-arrow-right: [Sauvegardes](../transverse/concept-sauvegardes.md)

## S

**Samba** · 💾🖥️
: Logiciel libre qui fait parler SMB à Linux : partager des dossiers avec des postes Windows, et même rejoindre un domaine Active Directory. :material-arrow-right: [TP infra PME](../systeme/tp-infra-pme-proxmox.md)

**Sauvegarde 3-2-1** · 💾
: 3 copies des données, sur 2 supports différents, dont 1 hors site. :material-arrow-right: [Sauvegardes](../transverse/concept-sauvegardes.md)

**SID** · 🖥️
: Security Identifier : identifiant unique d'un compte, d'un groupe ou d'une machine Windows. Deux clones Windows non passés par `sysprep` ont le même SID machine.

**SMB / CIFS** · 💾
: Protocole de partage de fichiers et d'imprimantes de Windows (port 445).

**Snapshot** · 📦💾
: Photo de l'état d'une VM à un instant, pour revenir en arrière vite. Ce n'est **pas** une sauvegarde : il reste sur le même disque. :material-arrow-right: [Snapshot ou sauvegarde](../transverse/concept-sauvegardes.md)

**SNMP** · 🌐
: Protocole de supervision qui lit l'état des équipements réseau (ports 161/162).

**Sous-réseau** · 🌐
: Découpage d'un réseau en morceaux plus petits. :material-arrow-right: [Découper en sous-réseaux](../reseau/tuto-sous-reseaux.md)

**SSH** · 🔒🌐
: Connexion chiffrée en ligne de commande à une machine distante (port 22). :material-arrow-right: [SSH](../reseau/concept-ssh.md)

**SSO** · 🔒
: Authentification unique : on se connecte une fois et on accède à tous les services.

**sudo** · 🖥️
: Commande Linux pour lancer une action avec les droits root, en étant tracé. :material-arrow-right: [Permissions Linux](../systeme/concept-permissions-linux.md)

**Supervision** (*monitoring*) · 🖥️
: Surveillance continue des serveurs et services avec alertes en cas de panne (Zabbix, Uptime Kuma).

**Switch** (commutateur) · 🌐
: Équipement qui relie les machines d'un même réseau local et aiguille les trames selon les adresses MAC (couche 2). :material-arrow-right: [VLAN](../reseau/concept-vlan.md)

## T

**TCP / UDP** · 🌐
: Protocoles de transport (couche 4). TCP est fiable et vérifie la livraison (web, SSH), UDP est rapide sans vérification (DNS, vidéo). :material-arrow-right: [TCP ou UDP](../reseau/concept-modele-osi-tcpip.md)

**Template** · 📦
: VM modèle, préparée une fois, dont on tire des clones (ex. `tpl-debian01`). :material-arrow-right: [Préparer un clone](../systeme/tuto-preparer-clone-vm.md)

**TLS / SSL** · 🔒
: Protocole qui chiffre une connexion (le « S » de HTTPS). SSL est l'ancien nom, aujourd'hui obsolète.

**TTL** · 🌐
: Durée de vie : en IP, nombre de routeurs qu'un paquet peut traverser ; en DNS, durée de mise en cache d'une réponse.

## V

**VSS** (clichés instantanés) · 💾
: Volume Shadow Copy Service : copies ponctuelles d'un volume Windows, qui permettent de récupérer un fichier via *Versions précédentes*.

**Virtualisation** · 📦
: Faire tourner plusieurs machines virtuelles sur une seule machine physique. :material-arrow-right: [Virtualisation](../systeme/concept-virtualisation.md)

**VLAN** · 🌐
: Réseau local virtuel : découpe un switch physique en plusieurs réseaux isolés. :material-arrow-right: [VLAN](../reseau/concept-vlan.md)

**VM** (machine virtuelle) · 📦
: Ordinateur simulé par un hyperviseur, avec son propre OS, ses disques et sa carte réseau. :material-arrow-right: [Virtualisation](../systeme/concept-virtualisation.md)

**vmbr** (pont réseau) · 📦
: Switch virtuel de Proxmox qui relie les VM entre elles et au réseau (ex. `vmbr2` pour le réseau interne).

**VPN** · 🔒
: Tunnel chiffré qui relie un poste ou un site distant au réseau de l'entreprise (WireGuard, OpenVPN, IPsec).

## Z

**Zone DNS** · 🌐
: Partie de l'espace de noms gérée par un serveur DNS, qui contient ses enregistrements (A, CNAME, MX…). :material-arrow-right: [Types d'enregistrements](memo-references-reseau.md#types-denregistrements-dns)

**Zero trust** · 🔒
: Principe de sécurité : ne faire confiance à personne par défaut, même à l'intérieur du réseau, et vérifier chaque accès.
