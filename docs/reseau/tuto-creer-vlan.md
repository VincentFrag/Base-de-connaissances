---
icon: material/wrench-outline
title: Créer et configurer des VLAN
tags: [réseau, tuto, vlan, cisco, pfsense, switch]
---

# 🛠️ Créer et configurer des VLAN

!!! abstract "Objectif"
    Créer deux VLAN sur un switch, y brancher des postes, les faire remonter en **trunk** jusqu'au routeur ou au pare-feu, puis les faire communiquer.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 30 min

## Prérequis

- Comprendre access, trunk et VLAN natif → 🧠 [Les VLAN](concept-vlan.md)
- Un switch Cisco (réel, **Packet Tracer** ou GNS3) et un routeur Cisco **ou** un pfSense
- Un accès console ou SSH au switch

## Le plan qu'on va monter

| VLAN | Nom | Réseau | Passerelle | Ports du switch |
|---|---|---|---|---|
| 10 | UTILISATEURS | `192.168.10.0/24` | `192.168.10.254` | Fa0/1 à Fa0/10 |
| 20 | SERVEURS | `192.168.20.0/24` | `192.168.20.254` | Fa0/11 à Fa0/20 |
| 99 | NATIF | (aucun hôte) | — | VLAN natif du trunk |

```
[PC VLAN 10]──Fa0/1──┐
                     [Switch]──Gi0/1 (trunk 10,20)──[Routeur / pfSense]
[Srv VLAN 20]─Fa0/11─┘
```

---

## Étape 1 — Créer les VLAN sur le switch

```text
enable
configure terminal
vlan 10
 name UTILISATEURS
vlan 20
 name SERVEURS
vlan 99
 name NATIF
exit
```

## Étape 2 — Affecter les ports d'accès

```text
interface range fa0/1 - 10
 switchport mode access
 switchport access vlan 10
 exit
interface range fa0/11 - 20
 switchport mode access
 switchport access vlan 20
 exit
```

## Étape 3 — Configurer le trunk vers le routeur

```text
interface gi0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 switchport trunk native vlan 99
 exit
```

!!! note "Erreur « An interface whose trunk encapsulation is Auto… »"
    Sur certains switchs (ex. Catalyst 3560), ajoute d'abord `switchport trunk encapsulation dot1q`.

## Étape 4 — Sauvegarder

```text
end
copy running-config startup-config
```

---

## Étape 5 — Router entre les VLAN

=== "Routeur Cisco (router on a stick)"
    Une **sous-interface** par VLAN sur le port relié au trunk :

    ```text
    enable
    configure terminal
    interface gi0/0
     no shutdown
     exit
    interface gi0/0.10
     encapsulation dot1Q 10
     ip address 192.168.10.254 255.255.255.0
     exit
    interface gi0/0.20
     encapsulation dot1Q 20
     ip address 192.168.20.254 255.255.255.0
     exit
    interface gi0/0.99
     encapsulation dot1Q 99 native
     exit
    end
    copy running-config startup-config
    ```

    Si le serveur DHCP est dans un autre VLAN (ex. `192.168.20.10`), ajoute un **relais** sur chaque sous-interface cliente :

    ```text
    interface gi0/0.10
     ip helper-address 192.168.20.10
    ```

=== "pfSense"
    1. **Interfaces → Assignments → VLANs → Add**
        - *Parent Interface* : la carte reliée au trunk (ex. `igb1`)
        - *VLAN Tag* : `10` · *Description* : `UTILISATEURS`
        - Recommencer pour le VLAN `20`
    2. **Interfaces → Assignments** : dans *Available network ports*, choisir chaque VLAN puis **Add**.
    3. Cliquer sur chaque nouvelle interface (`OPT1`, `OPT2`…) :
        - ☑ *Enable interface* · *Description* : `UTILISATEURS`
        - *IPv4 Configuration Type* : **Static IPv4** → `192.168.10.254` / **24**
        - **Save** puis **Apply Changes**
    4. **Services → DHCP Server → UTILISATEURS** (facultatif) : cocher *Enable*, plage `192.168.10.100` → `192.168.10.200`.
    5. **Firewall → Rules → UTILISATEURS** : par défaut, une nouvelle interface **bloque tout**. Ajouter une règle *Pass* (ex. source `UTILISATEURS subnets`, destination `any`) pour commencer, puis la restreindre.

=== "Serveur Linux en trunk"
    Pour qu'un serveur Linux (ou un hyperviseur) ait une patte dans chaque VLAN sur un seul câble :

    ```bash
    sudo ip link add link eth0 name eth0.10 type vlan id 10
    sudo ip addr add 192.168.10.5/24 dev eth0.10
    sudo ip link set eth0.10 up
    ```

    ⚠️ Non persistant au redémarrage : à déclarer ensuite dans la configuration réseau (Netplan, NetworkManager ou `/etc/network/interfaces`).

---

## ✅ Vérification

**Sur le switch :**

```text
show vlan brief              ! les VLAN existent et les bons ports sont dedans
show interfaces trunk        ! Gi0/1 est en trunk, VLAN autorisés et natif corrects
show interfaces fa0/1 switchport
```

**Sur le routeur Cisco :**

```text
show ip interface brief      ! les sous-interfaces sont « up / up »
```

**Depuis un poste du VLAN 10 :**

```bash
ping 192.168.10.254     # sa passerelle
ping 192.168.20.254     # la passerelle de l'autre VLAN
ping 192.168.20.10      # un serveur du VLAN 20
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| Le port n'apparaît pas dans `show vlan brief` | C'est normal pour un port **trunk** | Vérifier avec `show interfaces trunk` |
| Ping de la passerelle KO | Mauvais VLAN sur le port, trunk ou sous-interface absente | Étapes 2, 3 et 5 |
| Passerelle OK mais l'autre VLAN KO | Routage ou pare-feu | Sous-interface manquante, ou règle pfSense absente |
| Poste en `169.254.x.x` | Pas de DHCP dans ce VLAN | Serveur DHCP ou `ip helper-address` |

## 🔧 Dépannage courant

!!! question "« %CDP-4-NATIVE_VLAN_MISMATCH » dans les logs ?"
    Le VLAN natif n'est pas le même des deux côtés du trunk. Mettre le même partout (`switchport trunk native vlan 99`).

!!! question "Le VLAN marche sur un switch mais pas sur le second ?"
    Il faut aussi **créer le VLAN** sur le second switch et l'**autoriser** sur le trunk entre les deux (`switchport trunk allowed vlan add 30`).

!!! question "J'ai écrasé la liste des VLAN autorisés ?"
    `switchport trunk allowed vlan 30` **remplace** la liste. Pour ajouter un VLAN, utilise `switchport trunk allowed vlan add 30`.

!!! question "Tout est bon mais pfSense ne laisse rien passer ?"
    Une interface VLAN neuve n'a **aucune règle** : tout est bloqué. Ajoute une règle dans *Firewall → Rules*.

## Voir aussi

- 🧠 [Les VLAN](concept-vlan.md)
- 🛠️ [Calculer et découper un réseau en sous-réseaux](tuto-sous-reseaux.md)
- 🛠️ [Diagnostiquer un problème de connexion](tuto-diagnostic-connexion.md)
- 🛠️ [Filtrer et faire du NAT avec pfSense](tuto-pfsense-pare-feu-nat.md) : la suite, pour restreindre les flux entre VLAN

??? quote "Sources"
    *Commandes Cisco IOS : d'après les guides de configuration Cisco Catalyst (documentation éditeur, non liée page par page).*

    - [Netgate : VLAN sur pfSense](https://docs.netgate.com/pfsense/en/latest/vlan/index.html)
    - [Netplan : référence de configuration (VLAN)](https://netplan.readthedocs.io/en/stable/netplan-yaml/)
    - [Page de manuel ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html)
