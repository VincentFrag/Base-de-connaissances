---
icon: material/wrench-outline
title: Diagnostiquer un problème de connexion de A à Z
tags: [réseau, bases, tuto, dépannage]
---

# 🛠️ Diagnostiquer un problème de connexion de A à Z

!!! abstract "Objectif"
    Face à un « Internet ne marche pas », trouver **où** ça coince en remontant méthodiquement de la machine jusqu'à Internet.

**Niveau :** 🟢 Bases · **Durée :** 5 à 10 min

## Prérequis

- Savoir lire une adresse IP et un masque → 🧠 [Adressage IP](concept-adressage-ip.md)
- Un terminal : PowerShell / CMD (Windows) ou un shell (Linux)

## Le principe

On teste **du plus proche au plus loin**. La première étape qui échoue indique la zone du problème.

```
[Poste] → [Carte réseau] → [Passerelle] → [Internet (IP)] → [DNS (noms)]
   1            2                3               4                 5
```

---

## Étape 1 — Le poste a-t-il une adresse IP valide ?

=== "Windows"
    ```powershell
    ipconfig /all
    ```

=== "Linux"
    ```bash
    ip a
    ip r          # affiche la passerelle par défaut
    ```

**À vérifier :** une IP cohérente, un masque, une passerelle et un serveur DNS.

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| IP en `169.254.x.x` | Pas de réponse du DHCP | Étape 1 bis |
| Pas d'IP / carte « déconnectée » | Problème physique ou Wi-Fi | Câble, Wi-Fi, port du switch |
| IP correcte | OK | Étape 2 |

??? tip "Étape 1 bis : redemander une IP au DHCP"
    === "Windows"
        ```powershell
        ipconfig /release
        ipconfig /renew
        ```
    === "Linux"
        ```bash
        sudo dhclient -r && sudo dhclient
        # ou, avec NetworkManager :
        nmcli connection down "<nom>" && nmcli connection up "<nom>"
        ```

## Étape 2 — La pile réseau locale répond-elle ?

```bash
ping 127.0.0.1
```

✅ Réponses → la pile TCP/IP fonctionne.
❌ Échec → problème logiciel sur le poste (pilote, pile réseau à réinitialiser).

## Étape 3 — La passerelle est-elle joignable ?

```bash
ping <IP_de_la_passerelle>     # ex. ping 192.168.1.1
```

✅ Réponses → le réseau local fonctionne.
❌ Échec → câble, switch, VLAN ou mauvaise configuration IP/masque.

## Étape 4 — Internet est-il joignable par IP ?

```bash
ping 1.1.1.1
```

✅ Réponses → la sortie Internet fonctionne.
❌ Échec → problème au niveau du routeur/pare-feu ou chez le FAI. Pour voir où ça s'arrête :

=== "Windows"
    ```powershell
    tracert 1.1.1.1
    ```
=== "Linux"
    ```bash
    traceroute 1.1.1.1
    ```

## Étape 5 — La résolution de noms (DNS) fonctionne-t-elle ?

```bash
ping google.com
nslookup google.com
```

| Résultat | Diagnostic |
|---|---|
| `ping 1.1.1.1` OK mais `ping google.com` KO | **Problème DNS** 🎯 |
| `nslookup` ne répond pas | Serveur DNS injoignable ou mal configuré |

Tester un autre serveur DNS pour confirmer :

```bash
nslookup google.com 1.1.1.1
```

Vider le cache DNS :

=== "Windows"
    ```powershell
    ipconfig /flushdns
    ```
=== "Linux"
    ```bash
    resolvectl flush-caches
    ```

---

## ✅ Vérification finale

Le problème est résolu quand ces trois commandes répondent :

```bash
ping <passerelle>
ping 1.1.1.1
ping google.com
```

## 🔧 Dépannage courant

!!! question "Le ping échoue mais le site s'ouvre ?"
    Certains équipements ou pare-feux **bloquent l'ICMP** (le ping). Teste plutôt un port :
    `Test-NetConnection google.com -Port 443` (PowerShell) ou `curl -I https://google.com` (Linux).

!!! question "Un seul site ne marche pas ?"
    Le réseau est bon : le souci vient du site, du DNS pour ce domaine ou d'un filtrage (proxy, pare-feu).

!!! question "Ça marche en filaire mais pas en Wi-Fi ?"
    Compare `ipconfig /all` sur les deux interfaces : réseau, VLAN ou DHCP différents.

## Voir aussi

- 📋 [Mémo commandes réseau](memo-commandes-reseau.md)

??? quote "Sources"
    - [Microsoft Learn : commandes Windows (ipconfig, ping, tracert, nslookup)](https://learn.microsoft.com/fr-fr/windows-server/administration/windows-commands/windows-commands)
    - [PowerShell : Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
    - [Page de manuel ip(8)](https://man7.org/linux/man-pages/man8/ip.8.html)
    - [Page de manuel ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
    - [NetworkManager : nmcli(1)](https://networkmanager.dev/docs/api/latest/nmcli.html)
