---
icon: material/wrench-outline
title: Accéder à son homelab depuis l'extérieur avec WireGuard
tags: [réseau, tuto, vpn, wireguard, proxmox, accès distant, debian]
---

# 🛠️ Accéder à son homelab depuis l'extérieur avec WireGuard

!!! abstract "Objectif"
    Depuis l'école, le travail ou ton téléphone en 4G, allumer un VPN et joindre **toutes les VM du lab** (`ssh srv-web01`, interface web Proxmox, partages…) comme si tu étais chez toi, en n'ouvrant **qu'un seul port UDP** sur la box.

**Niveau :** 🟡 Intermédiaire · **Durée :** 45 min

## Prérequis

- Comprendre le principe d'un VPN, des clés et d'`AllowedIPs` → 🧠 [VPN](concept-vpn.md)
- Savoir se connecter en SSH aux VM du réseau interne → 🛠️ [SSH par rebond](tuto-ssh-rebond.md)
- Savoir ce qu'est une redirection de port (NAT) → 🧠 [Pare-feu et NAT](concept-pare-feu-nat.md)
- Une VM Debian 12 dans le lab (clonée depuis le template → 🛠️ [Préparer une VM clonée](../systeme/tuto-preparer-clone-vm.md))
- Un accès à l'interface d'administration de ta box Internet

## Le principe

```
 [PC portable / smartphone]          Internet           [Box]           [pve01]                [srv-vpn01]
  client WireGuard                                  IP publique      192.168.1.20            10.0.0.41
  wg0 : 10.8.0.2   ════════ tunnel UDP 51820 ═══════▶ redirige ──▶  redirige vers ──▶   wg0 : 10.8.0.1
                                                     UDP 51820      10.0.0.41               │
                                                                                            ▼
                                                                                 réseau du lab 10.0.0.0/24
                                                                         srv-dc01, srv-web01, srv-files01…
```

1. La **box** redirige le port UDP 51820 vers l'hôte Proxmox `pve01`.
2. **`pve01`** (qui est la passerelle du réseau interne) le redirige à son tour vers la VM `srv-vpn01`.
3. **`srv-vpn01`** déchiffre le trafic et le dépose sur le réseau du lab, en le **masquant** derrière sa propre IP (NAT), pour que les VM sachent répondre sans rien configurer.

!!! info "Pourquoi une VM dédiée plutôt que WireGuard sur Proxmox ?"
    On peut installer WireGuard directement sur l'hôte Proxmox (une étape de redirection en moins), mais on évite d'ajouter des services sur l'hyperviseur : une VM se sauvegarde, se restaure et se casse sans risque pour le reste du lab.

## Le lab d'exemple

| Machine | Rôle | IP lab | IP tunnel |
|---|---|---|---|
| `pve01` | Hôte Proxmox, passerelle | `192.168.1.20` et `10.0.0.1` | — |
| `srv-vpn01` | Serveur WireGuard (nouvelle VM) | `10.0.0.41` | `10.8.0.1` |
| `srv-dc01` | DNS du lab (`lab.local`) | `10.0.0.10` | — |
| PC portable | Client | réseau de l'école, de l'hôtel… | `10.8.0.2` |
| Smartphone | Client | 4G | `10.8.0.3` |

Réseau du tunnel : `10.8.0.0/24` · port : **UDP 51820** · adresse publique de la box : `vpn.example.net` (nom DynDNS) ou une IP comme `203.0.113.10`.

!!! tip "Choisir le réseau du tunnel"
    Prends une plage qui n'existe **ni** dans le lab, **ni** chez toi, **ni** là où tu te connecteras. `10.8.0.0/24` est un choix classique.

---

## Étape 1 — Créer la VM `srv-vpn01`

Clone le template Debian (`tpl-debian01`) dans Proxmox, sur le pont interne `vmbr2`, puis fixe son IP.

```bash
# Sur srv-vpn01 (console Proxmox ou ssh)
sudo hostnamectl set-hostname srv-vpn01
sudo nano /etc/network/interfaces
```

```text
auto ens18
iface ens18 inet static
    address 10.0.0.41/24
    gateway 10.0.0.1
    dns-nameservers 10.0.0.10
```

```bash
sudo systemctl restart networking
ping -c 2 10.0.0.1
```

## Étape 2 — Installer WireGuard et activer le routage

```bash
sudo apt update
sudo apt install -y wireguard iptables qrencode

# Autoriser la VM à faire passer des paquets d'une carte à l'autre
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system | grep ip_forward
```

Tu dois voir `net.ipv4.ip_forward = 1`.

## Étape 3 — Générer les clés

Une paire pour le serveur, une paire **par appareil** client. On les génère ici sur le serveur pour aller vite.

```bash
sudo -i
cd /etc/wireguard
umask 077

wg genkey | tee serveur.key | wg pubkey > serveur.pub
wg genkey | tee pc.key      | wg pubkey > pc.pub
wg genkey | tee tel.key     | wg pubkey > tel.pub

ls -l          # les fichiers .key doivent être en -rw------- (lisibles par root seul)
```

!!! warning "Les fichiers `.key` sont des secrets"
    Comme une clé privée SSH : ne jamais les publier, les envoyer par mail ou les mettre dans un dépôt Git. Une fois les configs clientes installées, supprime `pc.key` et `tel.key` du serveur.

## Étape 4 — Configurer le serveur

```bash
nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
# IP du serveur dans le tunnel
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <contenu de serveur.key>

# NAT : le trafic des clients sort sur le lab avec l'IP de srv-vpn01
PostUp   = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens18 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o ens18 -j MASQUERADE

# PC portable
[Peer]
PublicKey = <contenu de pc.pub>
AllowedIPs = 10.8.0.2/32

# Smartphone
[Peer]
PublicKey = <contenu de tel.pub>
AllowedIPs = 10.8.0.3/32
```

!!! tip "Remplir les clés sans copier-coller"
    `cat serveur.key` affiche la clé à recopier. Côté serveur, chaque `[Peer]` n'a droit qu'à **une seule IP** (`/32`) : c'est ce qui empêche un client de se faire passer pour un autre.

Démarre l'interface et active-la au démarrage :

```bash
systemctl enable --now wg-quick@wg0
wg show
```

`wg show` doit afficher `interface: wg0`, `listening port: 51820` et les deux pairs.

## Étape 5 — Ouvrir le chemin depuis Internet

### 5a. Sur la box Internet

Dans l'interface de la box (souvent `192.168.1.1`), menu **NAT / PAT**, **Redirection de ports** ou **Pare-feu** :

| Champ | Valeur |
|---|---|
| Protocole | **UDP** |
| Port externe | `51820` |
| IP interne | `192.168.1.20` (l'hôte `pve01`) |
| Port interne | `51820` |

!!! tip "IP publique dynamique"
    Crée un nom gratuit chez un service DynDNS (DuckDNS, No-IP…) ou active l'option DynDNS de la box. Ton client visera ce nom (`vpn.example.net`) et suivra les changements d'IP.

### 5b. Sur l'hôte Proxmox `pve01`

`pve01` reçoit le paquet sur sa patte `192.168.1.20` (pont `vmbr0`) et doit le renvoyer vers `srv-vpn01`. On ajoute une règle de **DNAT** dans la déclaration du pont interne :

```bash
# Sur pve01
nano /etc/network/interfaces
```

Sous le bloc `iface vmbr2 inet static` existant (celui du réseau interne, qui porte souvent déjà la ligne `MASQUERADE` donnant Internet aux VM), ajoute :

```text
    post-up   iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 51820 -j DNAT --to-destination 10.0.0.41:51820
    post-down iptables -t nat -D PREROUTING -i vmbr0 -p udp --dport 51820 -j DNAT --to-destination 10.0.0.41:51820
```

Applique sans redémarrer, puis vérifie :

```bash
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 51820 -j DNAT --to-destination 10.0.0.41:51820
iptables -t nat -L PREROUTING -n -v | grep 51820
```

!!! info "Si le pare-feu Proxmox est activé"
    Au niveau **Datacenter** ou **Nœud**, ajoute une règle entrante *ACCEPT UDP 51820*. Au niveau de la **VM** `srv-vpn01`, idem si la case *Firewall* de sa carte réseau est cochée.

## Étape 6 — Configurer les clients

Le fichier client est le **miroir** de celui du serveur.

```ini
[Interface]
PrivateKey = <contenu de pc.key>
Address = 10.8.0.2/32
# Pour utiliser les noms du lab (srv-web01.lab.local)
DNS = 10.0.0.10, lab.local

[Peer]
PublicKey = <contenu de serveur.pub>
Endpoint = vpn.example.net:51820
# Split tunnel : seuls le lab et le tunnel passent par le VPN
AllowedIPs = 10.0.0.0/24, 10.8.0.0/24
PersistentKeepalive = 25
```

=== "Windows"
    1. Installe le client officiel depuis **wireguard.com/install**.
    2. **Ajouter un tunnel** → **Ajouter un tunnel vide…**, nomme-le `homelab`.
    3. Remplace le contenu par le fichier ci-dessus (garde la clé privée générée à l'étape 3, pas celle proposée par l'application).
    4. **Enregistrer**, puis **Activer**.

=== "Linux"
    ```bash
    sudo apt install -y wireguard resolvconf
    sudo nano /etc/wireguard/homelab.conf     # coller le fichier ci-dessus
    sudo chmod 600 /etc/wireguard/homelab.conf

    sudo wg-quick up homelab       # allumer
    sudo wg-quick down homelab     # éteindre
    ```

=== "Smartphone (Android / iOS)"
    Sur `srv-vpn01`, écris le fichier du téléphone (`tel.key`, `Address = 10.8.0.3/32`) dans `/etc/wireguard/tel.conf`, puis affiche-le en QR code :
    ```bash
    qrencode -t ansiutf8 < /etc/wireguard/tel.conf
    ```
    Dans l'application WireGuard : **+** → **Scanner un QR code**. Supprime ensuite `tel.conf` et `tel.key` du serveur.

!!! warning "Tester depuis l'extérieur"
    Depuis chez toi, sur le même réseau que la box, le test peut échouer alors que tout est bon (beaucoup de box ne gèrent pas le « retour » vers leur propre IP publique). Teste avec le **téléphone en 4G** (Wi-Fi coupé) ou en partage de connexion.

---

## ✅ Vérification

Client connecté, depuis l'extérieur :

=== "Windows"
    ```powershell
    ping 10.8.0.1                # le serveur dans le tunnel
    ping 10.0.0.31               # une VM du lab
    Resolve-DnsName srv-web01.lab.local
    ssh admin@10.0.0.31
    ```

=== "Linux"
    ```bash
    ping -c 2 10.8.0.1
    ping -c 2 10.0.0.31
    resolvectl query srv-web01.lab.local   # ou : dig srv-web01.lab.local
    ssh admin@10.0.0.31
    ```

Puis ouvre `https://10.0.0.1:8006` : l'interface Proxmox répond par le tunnel.

Sur le serveur, `sudo wg show` doit indiquer pour le pair un **`latest handshake`** récent (quelques secondes) et des compteurs **`transfer`** qui augmentent.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| Pas de `latest handshake` sur le serveur, `transfer: 0 B received` | Le paquet n'arrive **jamais** à `srv-vpn01` | Vérifier la redirection de la box, le DNAT sur `pve01` (`iptables -t nat -L -n -v`, le compteur doit bouger), l'`Endpoint` côté client, un éventuel CGNAT |
| Handshake OK, `ping 10.8.0.1` OK, mais `ping 10.0.0.31` échoue | Le tunnel marche, le **retour depuis le lab** non | `sysctl net.ipv4.ip_forward` = 1 ? Règle `MASQUERADE` présente (`iptables -t nat -L POSTROUTING -n`) ? Bon nom de carte (`ip -br a`) ? |
| Handshake qui ne se fait jamais, clés « correctes » | Clé publique mal recopiée ou inversée | Côté client : `PublicKey` = **serveur.pub** ; côté serveur : `PublicKey` = **pc.pub** |
| Les IP marchent, pas les noms `lab.local` | DNS du tunnel non pris en compte | Vérifier la ligne `DNS =` ; sous Linux installer `resolvconf` ; tester `nslookup srv-web01.lab.local 10.0.0.10` |
| Tout marche puis coupe au bout de quelques minutes | La box ou le réseau d'en face oublie la connexion UDP | `PersistentKeepalive = 25` côté client |
| `RTNETLINK answers: Address already in use` | Une autre interface utilise déjà cette plage | Changer le réseau du tunnel, ou couper l'autre VPN |

## 🔧 Dépannage courant

!!! question "Ça marche en 4G mais pas depuis le Wi-Fi de l'école"
    Certains réseaux bloquent tout l'UDP sortant sauf les ports usuels. Essaie un port moins filtré : `ListenPort = 443` côté serveur (et redirection UDP 443 sur la box et `pve01`), `Endpoint = vpn.example.net:443` côté client. Si l'école a la même plage que ton lab, voir le piège « même plage » de la 🧠 [fiche VPN](concept-vpn.md).

!!! question "Comment ajouter un nouvel appareil ?"
    Nouvelle paire de clés, nouvelle IP libre du tunnel (`10.8.0.4`), un bloc `[Peer]` de plus dans `wg0.conf`, puis rechargement sans couper les autres :
    ```bash
    sudo bash -c 'wg syncconf wg0 <(wg-quick strip wg0)'
    ```

!!! question "J'ai perdu mon téléphone"
    Supprime son bloc `[Peer]` dans `wg0.conf` et recharge (`wg syncconf` ci-dessus). Sans sa clé publique sur le serveur, l'appareil ne peut plus rien.

!!! question "Et sur pfSense ?"
    Installe le paquet **WireGuard** (*System → Package Manager*). Crée un tunnel (*VPN → WireGuard → Tunnels*, port 51820, réseau `10.8.0.1/24`), ajoute les pairs avec leur clé publique, assigne l'interface (*Interfaces → Assignments*) puis ajoute deux règles de pare-feu : *WAN* « UDP 51820 vers This Firewall » et *WireGuard* « source 10.8.0.0/24 vers LAN ». Les clients sont configurés exactement comme à l'étape 6. Voir 🛠️ [Filtrer et faire du NAT avec pfSense](tuto-pfsense-pare-feu-nat.md).

!!! question "Et pour relier deux sites (site-à-site) ?"
    Même outil, mais chaque côté est un **routeur** : dans le `[Peer]` de chaque site, `AllowedIPs` contient le **réseau de l'autre site** (ex. `192.168.20.0/24`), pas une seule IP. Pas de `MASQUERADE` cette fois : on ajoute une **route** vers le réseau distant sur la passerelle de chaque site, pour que les postes passent par le tunnel sans rien installer.

## Voir aussi

- 🧠 [VPN](concept-vpn.md)
- 🛠️ [SSH par rebond vers un réseau interne](tuto-ssh-rebond.md) (une fois le VPN allumé, `ssh 10.0.0.31` marche sans rebond)
- 🛠️ [Sauvegarder et restaurer ses VM avec Proxmox](../transverse/tuto-proxmox-backup.md) (pense à inclure `srv-vpn01` dans le job)

??? quote "Sources"
    - [WireGuard : démarrage rapide](https://www.wireguard.com/quickstart/)
    - [Page de manuel wg(8)](https://man7.org/linux/man-pages/man8/wg.8.html)
    - [Page de manuel wg-quick(8)](https://man7.org/linux/man-pages/man8/wg-quick.8.html)
    - [Wiki Debian : WireGuard](https://wiki.debian.org/WireGuard)
