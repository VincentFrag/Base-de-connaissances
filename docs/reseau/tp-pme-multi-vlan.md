---
icon: material/flask-outline
title: "TP : Le réseau multi-VLAN d'une PME"
tags: [réseau, tp, vlan, dhcp, dns, sous-réseaux, acl, packet tracer]
---

# 🧪 TP : Le réseau multi-VLAN d'une PME

!!! abstract "Ce que tu vas faire"
    Concevoir et monter le réseau complet d'une PME : plan d'adressage, VLAN, routage entre VLAN, DHCP, DNS et isolement du Wi-Fi invités.

**Niveau :** 🟡 Intermédiaire · **Durée :** 1 h 30 à 2 h · **Outil :** Cisco Packet Tracer

## Avant de commencer

Ce TP mobilise :

- 🛠️ [Calculer et découper un réseau en sous-réseaux](tuto-sous-reseaux.md)
- 🛠️ [Créer et configurer des VLAN](tuto-creer-vlan.md)
- 🧠 [DHCP et DNS](concept-dhcp-dns.md)

!!! tip "Comment utiliser ce TP"
    Pour chaque tâche, essaie d'abord **seul**. Bloqué ? Ouvre l'**💡 Indice**. Toujours bloqué, ou pour vérifier ? Ouvre la **✅ Solution**.

---

## Le contexte

La **PME Dupont** (60 salariés) emménage dans de nouveaux locaux. Le gérant veut :

- séparer les **utilisateurs**, la **comptabilité**, les **serveurs** et le **Wi-Fi invités** ;
- que les postes reçoivent leur configuration **automatiquement** ;
- pouvoir joindre le serveur interne par son **nom** ;
- que les **invités n'accèdent à rien d'interne**, sauf la résolution DNS.

## Le cahier des charges

**Plage attribuée :** `172.16.0.0/24` · **Domaine :** `dupont.local`

| VLAN | Nom | Nb d'hôtes nécessaires | Ports du switch |
|---|---|---|---|
| 10 | UTILISATEURS | 100 | Fa0/1 – Fa0/10 |
| 20 | COMPTA | 25 | Fa0/11 – Fa0/15 |
| 30 | INVITES | 40 | Fa0/16 – Fa0/20 |
| 40 | SERVEURS | 10 | Fa0/21 – Fa0/22 |
| 99 | NATIF | 0 | VLAN natif du trunk |

**Règles :**

- La passerelle de chaque VLAN est la **dernière adresse utilisable** de son sous-réseau.
- Le serveur `srv01` a la **première adresse utilisable** du VLAN SERVEURS.
- Le routeur distribue les adresses en **DHCP** pour les VLAN 10, 20 et 30.

## Le matériel

| Équipement | Modèle Packet Tracer | Liaison |
|---|---|---|
| R1 | Routeur 2911 | Gi0/0 ↔ Gi0/1 de SW1 |
| SW1 | Switch 2960 | |
| PC-U1 | PC | Fa0/1 |
| PC-C1 | PC | Fa0/11 |
| PC-I1 | Laptop | Fa0/16 |
| srv01 | Server | Fa0/21 |

---

## Tâche 1 — Le plan d'adressage

Calcule pour chaque VLAN : le réseau, le masque, la plage utilisable, la passerelle et le broadcast.

??? tip "💡 Indice"
    Classe les besoins **du plus grand au plus petit** (100, 40, 25, 10), puis attribue les blocs dans cet ordre en partant de `172.16.0.0`. Pour 100 hôtes, il faut un bloc de 128.

??? success "✅ Solution"
    | VLAN | Réseau | Masque | Plage utilisable | Passerelle | Broadcast |
    |---|---|---|---|---|---|
    | 10 UTILISATEURS | `172.16.0.0/25` | `255.255.255.128` | `.1` → `.126` | `.126` | `.127` |
    | 30 INVITES | `172.16.0.128/26` | `255.255.255.192` | `.129` → `.190` | `.190` | `.191` |
    | 20 COMPTA | `172.16.0.192/27` | `255.255.255.224` | `.193` → `.222` | `.222` | `.223` |
    | 40 SERVEURS | `172.16.0.224/28` | `255.255.255.240` | `.225` → `.238` | `.238` | `.239` |

    `srv01` = **`172.16.0.225`**. Il reste `172.16.0.240` → `.255` pour plus tard.

## Tâche 2 — Créer les VLAN et affecter les ports (SW1)

??? tip "💡 Indice"
    Commandes `vlan`, `name`, puis `interface range` avec `switchport mode access` et `switchport access vlan`.

??? success "✅ Solution"
    ```text
    enable
    configure terminal
    hostname SW1
    vlan 10
     name UTILISATEURS
    vlan 20
     name COMPTA
    vlan 30
     name INVITES
    vlan 40
     name SERVEURS
    vlan 99
     name NATIF
    exit
    interface range fa0/1 - 10
     switchport mode access
     switchport access vlan 10
    interface range fa0/11 - 15
     switchport mode access
     switchport access vlan 20
    interface range fa0/16 - 20
     switchport mode access
     switchport access vlan 30
    interface range fa0/21 - 22
     switchport mode access
     switchport access vlan 40
    exit
    ```

## Tâche 3 — Configurer le trunk vers R1 (SW1)

??? tip "💡 Indice"
    Port Gi0/1 en mode trunk, VLAN autorisés 10, 20, 30, 40, 99, et VLAN natif 99.

??? success "✅ Solution"
    ```text
    interface gi0/1
     switchport mode trunk
     switchport trunk allowed vlan 10,20,30,40,99
     switchport trunk native vlan 99
    end
    copy running-config startup-config
    ```

## Tâche 4 — Le routage entre VLAN (R1)

??? tip "💡 Indice"
    *Router on a stick* : une sous-interface `gi0/0.X` par VLAN, avec `encapsulation dot1Q X` et l'adresse de la passerelle. N'oublie pas d'activer l'interface physique.

??? success "✅ Solution"
    ```text
    enable
    configure terminal
    hostname R1
    interface gi0/0
     no shutdown
    interface gi0/0.10
     encapsulation dot1Q 10
     ip address 172.16.0.126 255.255.255.128
    interface gi0/0.20
     encapsulation dot1Q 20
     ip address 172.16.0.222 255.255.255.224
    interface gi0/0.30
     encapsulation dot1Q 30
     ip address 172.16.0.190 255.255.255.192
    interface gi0/0.40
     encapsulation dot1Q 40
     ip address 172.16.0.238 255.255.255.240
    interface gi0/0.99
     encapsulation dot1Q 99 native
    exit
    ```

## Tâche 5 — Le serveur srv01 (DNS)

Configure l'IP fixe de `srv01`, active son service DNS et crée l'enregistrement `srv01.dupont.local`.

??? tip "💡 Indice"
    Dans Packet Tracer : onglet **Desktop → IP Configuration** pour l'adresse, puis onglet **Services → DNS**.

??? success "✅ Solution"
    1. **Desktop → IP Configuration** : *Static*
        - IP `172.16.0.225` · masque `255.255.255.240`
        - Passerelle `172.16.0.238` · DNS `172.16.0.225`
    2. **Services → DNS** : *DNS Service* **On**
        - *Name* `srv01.dupont.local` · *Type* `A Record` · *Address* `172.16.0.225` → **Add**
    3. **Services → HTTP** : laisser **On** (pour tester avec le navigateur à la fin).

## Tâche 6 — Le DHCP sur R1

Crée un pool DHCP pour chaque VLAN client (10, 20, 30) : réseau, passerelle, DNS et domaine. Les passerelles ne doivent jamais être distribuées.

??? tip "💡 Indice"
    `ip dhcp excluded-address` pour les passerelles, puis `ip dhcp pool` avec `network`, `default-router`, `dns-server` et `domain-name`. Le routeur sert directement les VLAN de ses sous-interfaces : **pas besoin de relais** ici.

??? success "✅ Solution"
    ```text
    ip dhcp excluded-address 172.16.0.126
    ip dhcp excluded-address 172.16.0.222
    ip dhcp excluded-address 172.16.0.190

    ip dhcp pool UTILISATEURS
     network 172.16.0.0 255.255.255.128
     default-router 172.16.0.126
     dns-server 172.16.0.225
     domain-name dupont.local
    ip dhcp pool COMPTA
     network 172.16.0.192 255.255.255.224
     default-router 172.16.0.222
     dns-server 172.16.0.225
     domain-name dupont.local
    ip dhcp pool INVITES
     network 172.16.0.128 255.255.255.192
     default-router 172.16.0.190
     dns-server 172.16.0.225
     domain-name dupont.local
    exit
    ```

    Puis sur chaque PC : **Desktop → IP Configuration → DHCP**.

## Tâche 7 — Isoler les invités (ACL)

Les invités doivent pouvoir faire des requêtes **DNS vers srv01**, mais ne doivent joindre **aucun autre** réseau interne.

??? tip "💡 Indice"
    Une ACL **étendue**, appliquée **en entrée** sur `gi0/0.30`. L'ordre compte : d'abord autoriser le DNS, puis interdire l'interne, puis tout autoriser (pour Internet). Masque générique d'un /26 : `0.0.0.63`.

??? success "✅ Solution"
    ```text
    ip access-list extended INVITES-ISOLES
     permit udp 172.16.0.128 0.0.0.63 host 172.16.0.225 eq 53
     deny   ip  172.16.0.128 0.0.0.63 172.16.0.0 0.0.0.127
     deny   ip  172.16.0.128 0.0.0.63 172.16.0.192 0.0.0.63
     permit ip any any
    exit
    interface gi0/0.30
     ip access-group INVITES-ISOLES in
    end
    copy running-config startup-config
    ```

    La 3ᵉ ligne (`172.16.0.192 0.0.0.63`) couvre d'un coup **COMPTA** (`.192`–`.223`) et **SERVEURS** (`.224`–`.255`).

---

## ✅ Grille de validation

| # | Test | Depuis | Résultat attendu |
|---|---|---|---|
| 1 | `show vlan brief` | SW1 | 5 VLAN, ports dans les bons VLAN |
| 2 | `show interfaces trunk` | SW1 | Gi0/1 en trunk, natif 99 |
| 3 | `show ip interface brief` | R1 | Sous-interfaces `up / up` |
| 4 | `ipconfig /all` | PC-U1 | IP en `172.16.0.x` (≤ .125), passerelle `.126`, DNS `.225` |
| 5 | `ping 172.16.0.193` (ou l'IP de PC-C1) | PC-U1 | ✅ Réponse (routage entre VLAN) |
| 6 | `ping srv01.dupont.local` | PC-U1 | ✅ Réponse (DNS) |
| 7 | Navigateur → `http://srv01.dupont.local` | PC-C1 | ✅ Page web du serveur |
| 8 | `nslookup srv01.dupont.local` | PC-I1 | ✅ Réponse (DNS autorisé) |
| 9 | `ping 172.16.0.225` | PC-I1 | ❌ Échec (invités isolés) |
| 10 | `show ip dhcp binding` | R1 | Un bail par PC client |

## 🔧 Si un test échoue

!!! question "PC en `169.254.x.x` (test 4) ?"
    Le port n'est pas dans le bon VLAN, le VLAN n'est pas autorisé sur le trunk, ou le pool DHCP a une erreur de `network`. Vérifie avec `show vlan brief` puis `show ip dhcp pool`.

!!! question "Ping entre VLAN KO (test 5) ?"
    Sous-interface manquante ou mauvais numéro dans `encapsulation dot1Q`. Vérifie que `gi0/0` est bien en `no shutdown`.

!!! question "Le ping fonctionne par IP mais pas par nom (test 6) ?"
    Service DNS éteint sur srv01, enregistrement mal saisi, ou DNS non distribué par le DHCP (`dns-server`).

!!! question "Les invités n'ont plus de DNS (test 8) ?"
    La ligne `permit udp … eq 53` doit être **avant** les `deny`. Dans une ACL, la première règle qui correspond l'emporte.

## Pour aller plus loin

- Remplacer R1 par un **pfSense** (voir l'onglet pfSense du 🛠️ [tuto VLAN](tuto-creer-vlan.md))
- Déplacer le DHCP vers un **Windows Server**, avec `ip helper-address` sur R1 (voir 🛠️ [Installer DHCP et DNS](tuto-dhcp-dns.md))

??? quote "Sources"
    Voir les sources des tutos liés : [VLAN](tuto-creer-vlan.md), [DHCP et DNS](tuto-dhcp-dns.md), [pare-feu et NAT](tuto-pfsense-pare-feu-nat.md).

