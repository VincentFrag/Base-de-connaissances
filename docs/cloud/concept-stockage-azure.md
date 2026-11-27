---
icon: material/book-open-variant
title: Le stockage dans Azure (comptes, blobs, niveaux, SAS)
tags: [cloud, intermédiaire, concept, azure, stockage, blob, sas, redondance, immuabilité]
---

# 🧠 Le stockage dans Azure : comptes, niveaux d'accès et SAS

!!! abstract "En une phrase"
    Un **compte de stockage** Azure héberge des fichiers (**blobs**), des partages SMB, des files et des tables ; on y choisit **combien de copies** garder (redondance), **à quel prix** selon la fréquence d'accès (niveaux), et **qui y accède** (rôles, liens **SAS**, réseau privé).

## L'analogie

Un garde-meubles :

- le **compte de stockage** = le bâtiment, avec son adresse unique ;
- le **conteneur** = un box loué dans le bâtiment ;
- le **blob** = un carton dans le box ;
- les **niveaux d'accès** = le box au rez-de-chaussée (cher, accès immédiat) ou à la cave (pas cher, il faut prévenir pour récupérer ses cartons) ;
- le **SAS** = un bon d'enlèvement nominatif, valable deux heures, pour un seul carton.

---

## Les services d'un compte de stockage

| Service | Ce que c'est | Équivalent sur site |
|---|---|---|
| **Blob** | Stockage **objet** : fichiers adressés par une URL `https://<compte>.blob.core.windows.net/<conteneur>/<nom>` | Un bucket S3, MinIO |
| **Files** | Partage de fichiers **SMB** ou NFS, montable comme un lecteur réseau | Un serveur de fichiers Samba ou Windows |
| **Queue** | File de messages entre applications | RabbitMQ |
| **Table** | Base NoSQL clé-valeur simple | — |
| **Disques managés** | Les disques des VM (service à part, pas dans un compte) | Un volume LVM ou Ceph |

## La redondance : combien de copies, où

| Option | Copies | Survit à | Usage |
|---|---|---|---|
| **LRS** | 3 dans un datacenter | Une panne de disque ou de serveur | Labs, données reconstructibles |
| **ZRS** | 3 réparties sur 3 zones | La perte d'un datacenter | Production dans une région |
| **GRS** | LRS + 3 copies dans la région jumelle | La perte d'une région | Sauvegardes, archives |
| **GZRS** | ZRS + 3 copies dans la région jumelle | Tout ce qui précède | Données critiques |

!!! info "Redondance n'est pas sauvegarde"
    Les copies sont **synchrones** : un fichier supprimé ou chiffré par un rançongiciel l'est partout. Contre ça, il faut la **suppression réversible** (*soft delete*), le **versionnage** et l'**immuabilité**.

## Les niveaux d'accès des blobs

| Niveau | Stockage | Lecture | Durée minimale | Pour quoi |
|---|---|---|---|---|
| **Chaud** (*Hot*) | Le plus cher | Le moins cher | — | Données lues souvent |
| **Froid** (*Cool*) | Moins cher | Plus cher | 30 jours | Sauvegardes récentes |
| **Très froid** (*Cold*) | Encore moins cher | Encore plus cher | 90 jours | Sauvegardes anciennes |
| **Archive** | Quelques fractions de centime par Go | Très cher, **plusieurs heures** de réhydratation | 180 jours | Conservation légale |

Une **règle de cycle de vie** déplace les blobs tout seuls : par exemple chaud → froid après 30 jours, archive après 180 jours, suppression après 3 ans.

## Qui peut accéder aux données

| Méthode | Principe | Recommandation |
|---|---|---|
| **Clé du compte** | Deux clés maîtresses qui donnent **tous** les droits | À éviter ; désactivable |
| **Entra ID + RBAC** | Rôle *Storage Blob Data Reader/Contributor* sur un compte ou un conteneur | ✅ Pour les personnes et les VM (identité managée) |
| **SAS de délégation d'utilisateur** | Lien signé par une identité Entra ID, droits et durée limités | ✅ Pour partager ponctuellement un fichier |
| **SAS de compte ou de service** | Lien signé avec la clé du compte | Seulement si rien d'autre n'est possible |
| **Accès public anonyme** | Le conteneur est lisible par tous | ❌ Désactivé par défaut, à laisser ainsi |

Un **SAS** est une URL avec des paramètres : droits (`sp=r` lecture), début et fin de validité (`st`, `se`), protocole (`spr=https`), éventuellement IP autorisée (`sip`), et la **signature** (`sig`). Qui a le lien a l'accès, jusqu'à expiration.

## Protéger contre la suppression

| Mécanisme | Ce qu'il fait |
|---|---|
| **Suppression réversible** | Un blob supprimé reste récupérable N jours |
| **Versionnage** | Chaque écrasement garde l'ancienne version |
| **Immuabilité** (WORM) | Pendant la durée fixée, **personne** ne peut modifier ni supprimer, même un Propriétaire ; une fois **verrouillée**, la règle ne peut plus être raccourcie |
| **Verrou de ressource** | Empêche la suppression du compte lui-même |

## Garder le trafic privé

| Option | Principe |
|---|---|
| **Pare-feu du compte** | N'accepte que certaines IP publiques et certains sous-réseaux |
| **Point de terminaison de service** | Le sous-réseau atteint le stockage par le réseau Azure ; le compte garde son IP publique mais n'accepte que ce sous-réseau |
| **Point de terminaison privé** (*Private Endpoint*) | Le compte reçoit une **IP privée dans ton VNet** ; on peut alors couper tout accès public |

---

## À quoi ça sert concrètement

- **Copie hors site** des sauvegardes du homelab, en niveau froid, avec immuabilité contre les rançongiciels.
- **Partager un fichier** volumineux avec un client pendant 48 h, par un lien SAS qui expire tout seul.
- **Héberger les fichiers statiques** d'un site (images, PDF) sans les mettre sur la VM.
- **Remplacer un serveur de fichiers** par un partage Azure Files monté sur les postes.

!!! warning "Pièges fréquents"
    - Le nom du compte est **mondial** : 3 à 24 caractères, minuscules et chiffres uniquement, unique sur tout Azure.
    - Partager un SAS **sans date d'expiration courte** ou signé avec la clé du compte : impossible à révoquer sans changer la clé.
    - Mettre des sauvegardes en **Archive** sans savoir qu'une restauration prend des heures et coûte cher : vérifier le RTO.
    - Oublier les **durées minimales** : supprimer un blob froid au bout de 10 jours fait payer les 30.
    - Verrouiller une règle d'immuabilité de plusieurs années **sur un lab** : on ne pourra plus supprimer le compte avant la fin.

## Pour aller plus loin

- 🛠️ **[Stocker et partager des fichiers dans Azure](tuto-stockage-azure-sas.md)** : la mise en pratique (compte, SAS, cycle de vie, immuabilité, Private Endpoint)
- 🧠 [Sauvegardes](../transverse/concept-sauvegardes.md) : la règle 3-2-1 et la copie hors site
- 🧠 [Identités et droits dans Azure](concept-identite-azure.md) : plan de contrôle et plan de données
