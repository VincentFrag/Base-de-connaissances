---
icon: material/book-open-variant
title: DHCP et DNS
tags: [réseau, bases, concept, dhcp, dns]
---

# 🧠 DHCP et DNS

!!! abstract "En une phrase"
    Le **DHCP** donne automatiquement à chaque poste sa configuration réseau (IP, masque, passerelle, DNS) ; le **DNS** traduit les noms (`google.com`) en adresses IP.

## L'analogie

Tu arrives à l'hôtel :

- **DHCP** = l'accueil, qui te donne un **numéro de chambre** pour la durée de ton séjour (le *bail*).
- **DNS** = l'annuaire, qui te dit « le restaurant *Le Lilas* ? C'est au **12 rue des Fleurs** ».

---

## DHCP : obtenir sa configuration

### Ce que le DHCP distribue

| Paramètre | Exemple |
|---|---|
| Adresse IP | `192.168.1.42` |
| Masque | `255.255.255.0` |
| Passerelle par défaut | `192.168.1.1` |
| Serveur(s) DNS | `192.168.1.10` |
| Durée du bail | 8 heures, 1 jour… |

### Les 4 étapes : DORA

| Étape | Qui → qui | Message |
|---|---|---|
| **D**iscover | Poste → tout le monde (broadcast) | « Y a-t-il un serveur DHCP ? » |
| **O**ffer | Serveur → poste | « Je te propose `192.168.1.42` » |
| **R**equest | Poste → tout le monde | « J'accepte `192.168.1.42` » |
| **A**ck | Serveur → poste | « C'est validé, pour 8 h » |

Le poste renouvelle son bail à **50 % de sa durée**, sans repasser par toute la séquence.

### Vocabulaire utile

- **Étendue** (*scope*) : la plage d'adresses distribuables, ex. `192.168.1.100` à `.200`.
- **Exclusion** : adresses de la plage à ne jamais distribuer (serveurs, imprimantes).
- **Réservation** : une IP toujours attribuée à la même machine, identifiée par son **adresse MAC**.
- **Relais DHCP** (*IP helper*) : le broadcast ne traverse pas les routeurs ; le relais transmet les demandes d'un VLAN vers un serveur DHCP central.

---

## DNS : résoudre les noms

### Comment se passe une résolution

```
Poste → cache local ? → serveur DNS configuré (résolveur)
                              ↓ s'il ne sait pas
               serveurs racine (.) → serveurs .fr → serveur de exemple.fr
                              ↓
                    réponse : 93.184.216.34
```

Chaque réponse est gardée en **cache** pendant une durée appelée **TTL**.

### Les principaux types d'enregistrements

| Type | Rôle | Exemple |
|---|---|---|
| **A** | Nom → IPv4 | `srv01` → `192.168.1.10` |
| **AAAA** | Nom → IPv6 | `srv01` → `2001:db8::10` |
| **CNAME** | Alias vers un autre nom | `intranet` → `srv01` |
| **PTR** | IP → nom (résolution inverse) | `192.168.1.10` → `srv01` |
| **MX** | Serveur de mail du domaine | `exemple.fr` → `mail.exemple.fr` |
| **TXT** | Texte libre (SPF, vérifications) | `v=spf1 …` |
| **SRV** | Localiser un service | `_ldap._tcp` → contrôleur de domaine |

!!! info "DNS et Active Directory"
    Dans un domaine AD, les postes trouvent les contrôleurs de domaine grâce aux enregistrements **SRV**. Si un poste utilise un DNS externe (ex. `8.8.8.8`) au lieu du DNS interne, **il ne trouve plus le domaine** : ouverture de session, GPO et partages échouent.

---

## À quoi ça sert concrètement

- **DHCP** : configurer des centaines de postes sans intervention, et changer un paramètre (DNS, passerelle) pour tout le parc en un seul endroit.
- **DNS** : utiliser des noms stables même si les IP changent, et faire fonctionner Active Directory, la messagerie et le web.

!!! warning "Pièges fréquents"
    - Une IP en `169.254.x.x` = **pas de réponse DHCP** (serveur arrêté, étendue pleine, relais absent sur le VLAN).
    - Deux serveurs DHCP sur le même réseau = conflits d'adresses.
    - Attribuer une IP fixe à la main **dans** l'étendue DHCP, sans exclusion = conflit tôt ou tard.
    - « Internet marche par IP mais pas par nom » = problème **DNS**, pas réseau.
    - Après une modification DNS, l'ancien résultat peut rester en cache jusqu'à expiration du **TTL** : penser à `ipconfig /flushdns`.

## Pour aller plus loin

- 🛠️ **[Installer et configurer DHCP et DNS](tuto-dhcp-dns.md)** : la mise en pratique (Windows Server, Linux)
- 🛠️ [Diagnostiquer un problème de connexion de A à Z](tuto-diagnostic-connexion.md) (étapes 1 et 5)
- 📋 [Mémo commandes réseau](memo-commandes-reseau.md) (sections Configuration IP et DNS)
- 🧠 [Active Directory : domaine, OU et GPO](../systeme/concept-active-directory.md) : pourquoi AD dépend du DNS

??? quote "Sources"
    - [RFC 2131 : DHCP](https://www.rfc-editor.org/rfc/rfc2131)
    - [RFC 1034 : DNS, concepts](https://www.rfc-editor.org/rfc/rfc1034)
    - [RFC 1035 : DNS, implémentation](https://www.rfc-editor.org/rfc/rfc1035)
    - [Microsoft Learn : DHCP dans Windows Server](https://learn.microsoft.com/fr-fr/windows-server/networking/technologies/dhcp/dhcp-top)
    - [Microsoft Learn : DNS dans Windows Server](https://learn.microsoft.com/fr-fr/windows-server/networking/dns/dns-top)
