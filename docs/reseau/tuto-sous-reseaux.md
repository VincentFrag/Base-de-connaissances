---
icon: material/wrench-outline
title: Calculer et découper un réseau en sous-réseaux
tags: [réseau, bases, tuto, subnetting, cidr]
---

# 🛠️ Calculer et découper un réseau en sous-réseaux

!!! abstract "Objectif"
    Savoir, **sans calculatrice** : trouver le réseau d'une adresse IP, et découper un réseau en sous-réseaux adaptés aux besoins.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 20 min + exercices

## Prérequis

- Savoir lire une IP, un masque et la notation CIDR → 🧠 [Adressage IP et masque](concept-adressage-ip.md)

## La méthode du « nombre magique »

Tout repose sur un seul calcul : la **taille du bloc**.

```
Taille du bloc = 256 − (valeur du masque dans l'octet « intéressant »)
```

L'**octet intéressant**, c'est celui où le masque n'est ni 255 ni 0.

## La table à connaître

| CIDR (4ᵉ octet) | CIDR (3ᵉ octet) | Valeur du masque | Taille du bloc | Hôtes (dans le 4ᵉ octet) |
|---|---|---|---|---|
| /25 | /17 | 128 | 128 | 126 |
| /26 | /18 | 192 | 64 | 62 |
| /27 | /19 | 224 | 32 | 30 |
| /28 | /20 | 240 | 16 | 14 |
| /29 | /21 | 248 | 8 | 6 |
| /30 | /22 | 252 | 4 | 2 |

Nombre d'hôtes utilisables : **2^(32 − CIDR) − 2**.

---

## Étape 1 — Trouver le réseau d'une adresse

**Exemple :** `172.16.45.200/20`

1. **Masque** : /20 → `255.255.240.0`. L'octet intéressant est le **3ᵉ** (240).
2. **Taille du bloc** : 256 − 240 = **16**.
3. **Multiples de 16** dans le 3ᵉ octet : 0, 16, 32, **48**… → 45 est entre **32** et 47.
4. **Résultat :**

| Élément | Valeur |
|---|---|
| Adresse réseau | `172.16.32.0` |
| Première IP utilisable | `172.16.32.1` |
| Dernière IP utilisable | `172.16.47.254` |
| Broadcast | `172.16.47.255` |
| Nb d'hôtes | 2^12 − 2 = **4 094** |

## Étape 2 — Découper en sous-réseaux de taille égale

**Besoin :** découper `192.168.10.0/24` en **4** sous-réseaux.

1. Combien de bits emprunter ? 2^**2** = 4 → on emprunte **2 bits** : /24 + 2 = **/26**.
2. Taille du bloc pour /26 : **64**.
3. On avance de 64 en 64 :

| Sous-réseau | Réseau | Plage utilisable | Broadcast |
|---|---|---|---|
| 1 | `192.168.10.0/26` | `.1` → `.62` | `.63` |
| 2 | `192.168.10.64/26` | `.65` → `.126` | `.127` |
| 3 | `192.168.10.128/26` | `.129` → `.190` | `.191` |
| 4 | `192.168.10.192/26` | `.193` → `.254` | `.255` |

## Étape 3 — Découper selon les besoins (VLSM)

En vrai, les services n'ont pas tous la même taille. Avec le **VLSM**, on donne à chacun un sous-réseau à sa mesure.

**Besoin** dans `192.168.1.0/24` : Utilisateurs **100** postes, Serveurs **50**, Imprimantes **20**, lien entre deux routeurs **2**.

!!! tip "La règle d'or"
    Toujours placer les **plus grands besoins en premier**, sinon les blocs se chevauchent.

| Service | Besoin | CIDR choisi (capacité) | Réseau | Plage utilisable | Broadcast |
|---|---|---|---|---|---|
| Utilisateurs | 100 | /25 (126) | `192.168.1.0` | `.1` → `.126` | `.127` |
| Serveurs | 50 | /26 (62) | `192.168.1.128` | `.129` → `.190` | `.191` |
| Imprimantes | 20 | /27 (30) | `192.168.1.192` | `.193` → `.222` | `.223` |
| Lien routeurs | 2 | /30 (2) | `192.168.1.224` | `.225` → `.226` | `.227` |

Il reste `192.168.1.228` → `.255` de libre pour plus tard.

---

## ✅ Vérification

Vérifie tes calculs avec un outil :

=== "Linux"
    ```bash
    sudo apt install ipcalc
    ipcalc 172.16.45.200/20
    ```

=== "Windows"
    Pas d'outil intégré : utilise un calculateur en ligne (rechercher « subnet calculator »), ou installe `ipcalc` dans WSL.

## 🏋️ Exercices

??? question "1. Quel est le réseau de `10.0.5.77/27` ?"
    /27 → bloc de **32**. Multiples : 0, 32, **64**, 96 → 77 est dans le bloc 64.
    Réseau `10.0.5.64` · plage `.65` → `.94` · broadcast `.95` · **30 hôtes**.

??? question "2. Découper `192.168.50.0/24` en 8 sous-réseaux. Quel masque ?"
    2^3 = 8 → 3 bits empruntés → **/27** (`255.255.255.224`), blocs de 32 : `.0`, `.32`, `.64`, `.96`, `.128`, `.160`, `.192`, `.224`. 30 hôtes chacun.

??? question "3. Quel CIDR pour un réseau de 500 postes ?"
    /24 = 254 (trop petit) → **/23** = 2^9 − 2 = **510** hôtes. Masque `255.255.254.0`.

??? question "4. `192.168.1.130/25` et `192.168.1.100/25` peuvent-ils se parler sans routeur ?"
    /25 → blocs de 128 : `.0`–`.127` et `.128`–`.255`. `.100` est dans le premier, `.130` dans le second → **non**, ils sont sur deux réseaux différents.

## 🔧 Dépannage courant

!!! question "Mes sous-réseaux se chevauchent ?"
    En VLSM, tu n'as sans doute pas commencé par le plus grand besoin. Recommence en triant par taille décroissante.

!!! question "Il me manque 2 adresses par rapport à mon calcul ?"
    Tu as oublié de retirer l'**adresse réseau** et le **broadcast** : c'est le « − 2 ».

!!! question "Combien d'hôtes pour une croissance future ?"
    Prévois une marge d'environ **30 %** : 100 postes aujourd'hui → vise 130, donc /24 plutôt que /25.

## Voir aussi

- 🧠 [Les VLAN](concept-vlan.md) (un VLAN = un sous-réseau)

??? quote "Sources"
    - [RFC 950 : sous-réseaux](https://www.rfc-editor.org/rfc/rfc950)
    - [RFC 4632 : CIDR](https://www.rfc-editor.org/rfc/rfc4632)
