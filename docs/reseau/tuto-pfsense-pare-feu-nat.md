---
icon: material/wrench-outline
title: Filtrer et faire du NAT avec pfSense
tags: [réseau, tuto, pare-feu, nat, pfsense, vlan]
---

# 🛠️ Filtrer et faire du NAT avec pfSense

!!! abstract "Objectif"
    Sur un pfSense qui route déjà deux VLAN, **restreindre** ce que les utilisateurs peuvent joindre côté serveurs, **donner accès à Internet** aux deux VLAN grâce au NAT sortant, puis **publier** le serveur web interne sur le WAN avec une redirection de port.

**Niveau :** 🟡 Intermédiaire · **Durée :** 45 min

## Prérequis

- Comprendre règles, états et types de NAT → 🧠 [Pare-feu et NAT](concept-pare-feu-nat.md)
- Avoir monté les VLAN 10 et 20 sur pfSense → 🛠️ [Créer et configurer des VLAN](tuto-creer-vlan.md) (onglet pfSense)
- Un pfSense CE 2.7 ou plus (VM Proxmox, par exemple) avec une interface **WAN** qui sort sur Internet
- Un serveur web dans le VLAN 20 (ex. Debian avec `nginx`) et un poste client dans le VLAN 10

## Le plan qu'on va monter

| Élément | Adresse | Rôle |
|---|---|---|
| pfSense WAN | `192.168.1.250` (derrière la box du homelab) ou une IP publique | Sortie Internet |
| pfSense VLAN 10 `UTILISATEURS` | `192.168.10.254/24` | Passerelle des postes |
| pfSense VLAN 20 `SERVEURS` | `192.168.20.254/24` | Passerelle des serveurs |
| Serveur web | `192.168.20.10` (ports 80, 443, SSH 22) | Service à protéger puis publier |
| Poste client | `192.168.10.50` | Pour les tests |

```
                 Internet / box
                       │
                    [WAN] 192.168.1.250
                   [pfSense]
        VLAN 10 ─────┘   └───── VLAN 20
  192.168.10.254             192.168.20.254
  [PC 192.168.10.50]         [Serveur web 192.168.20.10]
```

**La politique visée pour le VLAN 10 :**

| # | Action | Source | Destination | Port | Pourquoi |
|---|---|---|---|---|---|
| 1 | ✅ Pass | UTILISATEURS subnets | UTILISATEURS address | 53 (TCP/UDP) | Résolution DNS par pfSense |
| 2 | ✅ Pass | UTILISATEURS subnets | UTILISATEURS address | ICMP | Pinger la passerelle (dépannage) |
| 3 | ✅ Pass | UTILISATEURS subnets | `SRV_WEB` | `PORTS_WEB` (80, 443) | Accès au site interne |
| 4 | ⛔ Reject + log | UTILISATEURS subnets | `RESEAUX_PRIVES` | tous | Rien d'autre en interne (SSH, interface de pfSense…) |
| 5 | ✅ Pass | UTILISATEURS subnets | any | tous | Internet |

---

## Étape 1 — Créer des alias

Un **alias** est un nom donné à une liste d'adresses ou de ports : les règles restent lisibles, et on modifie la liste à un seul endroit.

**Firewall → Aliases → IP → Add** :

| Name | Type | Contenu |
|---|---|---|
| `SRV_WEB` | Host(s) | `192.168.20.10` |
| `RESEAUX_PRIVES` | Network(s) | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |

**Firewall → Aliases → Ports → Add** :

| Name | Type | Contenu |
|---|---|---|
| `PORTS_WEB` | Port(s) | `80`, `443` |

Cliquer sur **Save** puis **Apply Changes** après chaque alias.

## Étape 2 — Supprimer la règle temporaire « tout autoriser »

Si tu as suivi le tuto VLAN, l'interface `UTILISATEURS` a une règle *Pass* `UTILISATEURS subnets → any`. **Firewall → Rules → UTILISATEURS** : supprime-la (icône 🗑️) pour repartir d'une page vide. Sans règle, tout est bloqué : c'est le *default deny*.

## Étape 3 — Écrire les règles du VLAN 10

**Firewall → Rules → UTILISATEURS → Add** (flèche vers le bas, pour ajouter en fin de liste), une règle à la fois, dans l'ordre du tableau :

1. **DNS vers pfSense**
    - *Action* : **Pass** · *Protocol* : **TCP/UDP**
    - *Source* : `UTILISATEURS subnets`
    - *Destination* : `UTILISATEURS address` · *Destination Port Range* : **DNS (53)**
    - *Description* : `DNS vers pfSense`
2. **Ping de la passerelle**
    - *Action* : **Pass** · *Protocol* : **ICMP** · *ICMP Subtypes* : **Echo request**
    - *Source* : `UTILISATEURS subnets` · *Destination* : `UTILISATEURS address`
    - *Description* : `Ping passerelle`
3. **Accès au serveur web**
    - *Action* : **Pass** · *Protocol* : **TCP**
    - *Source* : `UTILISATEURS subnets`
    - *Destination* : **Address or Alias** → `SRV_WEB`
    - *Destination Port Range* : **(other)** → `PORTS_WEB` dans *From* et *To*
    - *Description* : `Web interne`
4. **Bloquer le reste de l'interne**
    - *Action* : **Reject** · *Protocol* : **Any**
    - *Source* : `UTILISATEURS subnets` · *Destination* : **Address or Alias** → `RESEAUX_PRIVES`
    - ☑ *Log packets that are handled by this rule*
    - *Description* : `Interdit vers les réseaux internes`
5. **Internet**
    - *Action* : **Pass** · *Protocol* : **Any**
    - *Source* : `UTILISATEURS subnets` · *Destination* : **any**
    - *Description* : `Internet`

Puis **Apply Changes**. Vérifie l'ordre dans la liste : il doit être **1 → 5** de haut en bas (glisser-déposer puis **Save** si besoin).

!!! info "Pourquoi la règle 4 bloque aussi l'interface web de pfSense"
    `192.168.10.254` est dans `192.168.0.0/16` : les utilisateurs ne peuvent plus ouvrir la page d'administration de pfSense. C'est voulu : administre pfSense depuis le **LAN** ou un VLAN d'administration (la règle *anti-lockout* du LAN reste active). Le DHCP continue de fonctionner : pfSense ajoute ses propres règles cachées pour lui.

!!! tip "Et le VLAN 20 ?"
    Les serveurs doivent au moins sortir pour leurs mises à jour. Sur **Firewall → Rules → SERVEURS**, applique le même modèle : DNS, ping de la passerelle, *Reject* vers `RESEAUX_PRIVES`, puis *Pass* vers `any`. Ils ne pourront pas **initier** de connexion vers le VLAN 10, mais ils **répondent** aux postes grâce à la table d'états.

## Étape 4 — Vérifier le NAT sortant

**Firewall → NAT → Outbound**.

- Le mode par défaut **Automatic outbound NAT rule generation** crée tout seul une règle pour **chaque réseau interne** (dont tes VLAN) vers le WAN : il n'y a rien à faire.
- Contrôle dans *Automatic Rules* qu'une ligne couvre `192.168.10.0/24` et `192.168.20.0/24` avec la traduction **WAN address**.

!!! note "Quand passer en mode Hybrid ou Manual"
    Pour un besoin précis (un serveur qui doit sortir avec une autre IP publique, un réseau qui ne doit **pas** être NATé vers un VPN…), passe en **Hybrid** et ajoute ta règle **au-dessus** des règles automatiques.

## Étape 5 — Publier le serveur web (redirection de port)

**Firewall → NAT → Port Forward → Add** :

- *Interface* : **WAN** · *Address Family* : **IPv4** · *Protocol* : **TCP**
- *Destination* : **WAN address**
- *Destination port range* : **HTTPS** (*From* et *To*)
- *Redirect target IP* : **Address or Alias** → `SRV_WEB`
- *Redirect target port* : **HTTPS**
- *Description* : `Publication web`
- *Filter rule association* : **Add associated filter rule**

**Save** puis **Apply Changes**. Dans **Firewall → Rules → WAN**, une règle *Pass* vers `192.168.20.10:443` est apparue toute seule : c'est elle qui **autorise** le flux que la redirection **traduit**.

!!! warning "Homelab : pfSense derrière ta box"
    Si le WAN de pfSense a une IP privée (`192.168.1.x`), va dans **Interfaces → WAN** et décoche **Block private networks and loopback addresses**, sinon les clients du réseau de la box sont bloqués avant même la redirection. Pour ouvrir vraiment le service sur Internet, il faut en plus une redirection du port 443 **sur la box** vers `192.168.1.250`.

---

## ✅ Vérification

**Depuis le poste du VLAN 10 :**

=== "Linux"
    ```bash
    ping -c 2 192.168.10.254             # passerelle : OK (règle 2)
    nslookup debian.org 192.168.10.254   # DNS : OK (règle 1)
    curl -kI https://192.168.20.10       # web interne : OK (règle 3)
    nc -zv -w 3 192.168.20.10 22         # SSH serveur : refusé (règle 4)
    curl -I https://debian.org           # Internet : OK (règle 5 + NAT sortant)
    ```

=== "Windows"
    ```powershell
    ping 192.168.10.254
    Resolve-DnsName debian.org -Server 192.168.10.254
    Test-NetConnection 192.168.20.10 -Port 443   # TcpTestSucceeded : True
    Test-NetConnection 192.168.20.10 -Port 22    # TcpTestSucceeded : False
    Test-NetConnection debian.org -Port 443
    ```

**Depuis une machine côté WAN** (ex. un PC branché sur la box) :

```bash
curl -kI https://192.168.1.250       # répond comme le serveur web : redirection OK
```

**Sur pfSense :**

- **Status → System Logs → Firewall** : la tentative SSH apparaît en rouge, avec la description `Interdit vers les réseaux internes`.
- **Diagnostics → States** : filtre sur `192.168.20.10` pour voir les connexions web ouvertes.
- Dans le shell (option **8** de la console) :

    ```bash
    pfctl -sr | less       # règles de filtrage chargées
    pfctl -s nat           # règles NAT (sortant et redirections)
    pfctl -ss | grep 192.168.20.10   # états en cours vers le serveur
    ```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| Plus rien ne passe depuis le VLAN 10 | Règles absentes ou pas appliquées | **Apply Changes** ; vérifier l'interface choisie (étape 3) |
| SSH vers le serveur passe encore | État ouvert avant la règle, ou règle 5 placée avant la 4 | Vider les états, vérifier l'ordre |
| Web interne bloqué | Alias mal rempli ou port hors `PORTS_WEB` | **Firewall → Aliases**, contrôler l'IP et les ports |
| Internet KO mais passerelle OK | DNS bloqué ou NAT sortant absent | Règle 1 ; **Firewall → NAT → Outbound** en mode automatique |
| Redirection : délai dépassé depuis le WAN | *Block private networks* coché, ou règle WAN absente | Étape 5 |
| Redirection : page de connexion de pfSense | Test fait depuis l'intérieur, ou mauvais port | Tester depuis le WAN ; voir *NAT reflection* ci-dessous |

## 🔧 Dépannage courant

!!! question "La redirection marche depuis l'extérieur mais pas depuis le VLAN 10 ?"
    Normal : un client interne qui vise l'IP du WAN n'est pas traduit par défaut. Soit tu utilises l'IP interne (ou un nom DNS interne qui pointe vers `192.168.20.10`, le plus propre), soit tu actives **System → Advanced → Firewall & NAT → NAT Reflection mode : NAT + proxy**.

!!! question "J'ai durci une règle mais la connexion passe toujours ?"
    La connexion était déjà dans la **table d'états**. **Diagnostics → States → Reset States** (ou supprime uniquement les états du client) puis reteste.

!!! question "Je ne vois pas mes blocages dans les logs ?"
    Seules les règles avec la case **Log** cochée journalisent. Le blocage par défaut est journalisé, mais pas un *Reject* sans cette case.

!!! question "Je me suis enfermé dehors de l'interface web ?"
    Depuis la console de la VM, ouvre le shell (option **8**) et tape `pfctl -d` : le filtrage est **désactivé** le temps de corriger ta règle dans l'interface web. Le prochain **Apply Changes** le réactive (ou `pfctl -e`).

---

## Équivalents sur d'autres plateformes

La même politique (filtrage du VLAN 10, NAT sortant, publication du port 443) sur un routeur Cisco ou un routeur Linux :

=== "Cisco IOS"
    ```text
    enable
    configure terminal
    ! Filtrage à l'entrée du VLAN 10 (ACL sans état : les réponses passent car rien ne filtre en entrée côté serveurs)
    ip access-list extended UTILISATEURS-IN
     permit udp 192.168.10.0 0.0.0.255 host 192.168.10.254 eq 53
     permit icmp 192.168.10.0 0.0.0.255 host 192.168.10.254 echo
     permit tcp 192.168.10.0 0.0.0.255 host 192.168.20.10 eq 80
     permit tcp 192.168.10.0 0.0.0.255 host 192.168.20.10 eq 443
     deny   ip 192.168.10.0 0.0.0.255 192.168.0.0 0.0.255.255 log
     permit ip 192.168.10.0 0.0.0.255 any
     exit
    interface gi0/0.10
     ip access-group UTILISATEURS-IN in
     ip nat inside
     exit
    interface gi0/0.20
     ip nat inside
     exit
    interface gi0/1
     ip nat outside
     exit
    ! NAT sortant (PAT) pour les deux VLAN
    access-list 1 permit 192.168.10.0 0.0.0.255
    access-list 1 permit 192.168.20.0 0.0.0.255
    ip nat inside source list 1 interface gi0/1 overload
    ! Redirection de port 443 vers le serveur web
    ip nat inside source static tcp 192.168.20.10 443 interface gi0/1 443
    end
    copy running-config startup-config
    ```

    Vérifier avec `show access-lists` (compteurs de correspondances) et `show ip nat translations`.

=== "Linux (nftables)"
    Routeur Linux avec `eth0` côté WAN, `eth1.10` et `eth1.20` côté VLAN :

    ```bash
    # Activer le routage
    echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-routage.conf
    sudo sysctl --system
    ```

    `/etc/nftables.conf` :

    ```text
    flush ruleset

    table inet filtre {
      chain forward {
        type filter hook forward priority 0; policy drop;
        ct state established,related accept
        # VLAN 10 -> serveur web
        iifname "eth1.10" ip daddr 192.168.20.10 tcp dport { 80, 443 } accept
        # VLAN 10 -> reste de l'interne : refusé et journalisé
        iifname "eth1.10" ip daddr { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 } log prefix "REFUS-VLAN10 " reject
        # VLAN 10 et 20 -> Internet
        iifname { "eth1.10", "eth1.20" } oifname "eth0" accept
        # Redirection de port entrante
        iifname "eth0" ip daddr 192.168.20.10 tcp dport 443 ct status dnat accept
      }
    }

    table ip nat {
      chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "eth0" tcp dport 443 dnat to 192.168.20.10
      }
      chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "eth0" masquerade
      }
    }
    ```

    ```bash
    sudo nft -c -f /etc/nftables.conf   # vérifier la syntaxe
    sudo systemctl enable --now nftables
    sudo nft list ruleset
    ```

    Ce fichier ne filtre que le trafic **routé** (chaîne `forward`) : protéger le routeur lui-même demanderait une chaîne `input`.

## Voir aussi

- 🧠 [Pare-feu et NAT](concept-pare-feu-nat.md)
- 🛠️ [Créer et configurer des VLAN](tuto-creer-vlan.md)
- 🛠️ [Diagnostiquer une connexion de A à Z](tuto-diagnostic-connexion.md)
