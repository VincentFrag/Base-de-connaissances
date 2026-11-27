---
icon: material/flask-outline
title: "Exercices papier : penser cloud avant de cliquer"
tags: [cloud, azure, tp, dimensionnement, sla, adressage, nsg, migration, 7r, sauvegarde]
---

# 🧪 Exercices papier : penser cloud avant de cliquer

!!! abstract "Ce que tu vas faire"
    Sept exercices sur papier ou dans un tableur, sans consommer de crédit : lire une offre cloud, dimensionner, calculer un SLA, poser un plan d'adressage, lire des règles NSG, choisir une stratégie de migration et bâtir un plan de sauvegarde.

**Niveau :** 🟢 Bases · **Durée :** 5 h au total (30 min à 1 h par exercice) · **Coût :** 0 $

Ces sept exercices ne consomment aucun crédit et fixent les notions que les TP mettent en œuvre. Fais-les sur papier ou dans un tableur, puis ouvre le corrigé replié sous l'exercice.

---

## E1 — Comprendre l'offre OVHcloud

Objectif : savoir lire le catalogue d'un fournisseur et le comparer à Azure. Utilise uniquement ovhcloud.com.

1. Classe les gammes OVHcloud (Bare Metal, Public Cloud, Hosted Private Cloud, Managed Kubernetes, Object Storage, hébergement web) en IaaS / PaaS / SaaS et en cloud public / privé / bare metal.
2. Trouve l'instance Public Cloud la plus proche d'une VM Azure « 2 vCPU / 8 Go ». Compare le prix à l'heure et au mois avec la calculatrice Azure en France Central.
3. Liste les classes de stockage objet OVHcloud et mets-les en face des tiers Azure Hot / Cool / Cold / Archive.
4. Comment chaque fournisseur facture-t-il le trafic sortant (egress) ? Relie ta réponse à la slide sur la réversibilité (p. 115).
5. Quelles offres OVHcloud sont qualifiées SecNumCloud, et pourquoi une mutuelle de santé s'y intéresserait-elle ?

??? success "✅ Corrigé"

    Pas de corrigé chiffré : prix et gammes changent souvent, l'intérêt est d'aller les lire. Repères pour vérifier ta copie :

    - **Classement** : Bare Metal = serveurs dédiés (ni IaaS virtualisé ni cloud au sens strict) ; Public Cloud = IaaS public (instances, réseau, volumes) plus du PaaS (Kubernetes managé, bases managées) ; Hosted Private Cloud = cloud privé hébergé, dédié à un client ; hébergement web = PaaS/SaaS mutualisé.
    - **Comparaison de prix** : compare à configuration égale (vCPU, RAM, disque, trafic inclus) et sur le même engagement (à l'heure vs au mois).
    - **Egress** : c'est souvent là que la comparaison bascule. Un trafic sortant inclus ou peu facturé rend la réversibilité (p. 115) beaucoup moins chère qu'un egress facturé au Go.
    - **SecNumCloud** : qualification ANSSI qui garantit notamment l'immunité aux lois extraterritoriales (p. 107). Les données de santé d'une mutuelle exigent en plus un hébergeur certifié HDS : vérifie quelles offres ont l'une et l'autre.

## E2 — Dimensionnement cloud privé vs public

Un hyperviseur a 2 processeurs de 14 cœurs et 512 Go de RAM. Le fournisseur applique un surengagement CPU ×5, aucun surengagement RAM, et réserve 10 % de la RAM à l'hyperviseur.

1. Combien de vCPU et de Go de RAM sont louables ?
2. Combien de VM « standard » (2 vCPU / 8 Go) peut-on y placer ? Et de VM « RAM intense » (2 vCPU / 32 Go) ? Quelle ressource limite dans chaque cas ?
3. À partir de quel facteur de surengagement CPU la RAM devient-elle le seul goulot ? Qu'en déduis-tu sur le « ratio vCPU / RAM entre 3 et 4 » de la slide ?
4. Un cluster CEPH a 6 serveurs de 10 To bruts. Calcule la capacité utile en réplication ×3 et en erasure coding 4+2. Combien de pannes de disque ou de serveur chaque mode tolère-t-il ?
5. Pourquoi une instance SPOT coûte-t-elle beaucoup moins cher, et pour quels usages est-elle adaptée ?

??? success "✅ Corrigé"

    1. CPU : 2 × 14 = 28 cœurs × 5 = **140 vCPU**. RAM : 512 × 0,9 = **460,8 Go**.
    2. VM standard 2/8 : 70 par le CPU, 57 par la RAM → **57**, limité par la RAM. VM RAM intense 2/32 : 70 par le CPU, 14 par la RAM → **14**, limité par la RAM.
    3. Avec un catalogue en ratio 1 vCPU : 4 Go, la RAM supporte 460,8 / 4 = 115 vCPU, soit un surengagement de 115 / 28 ≈ **×4,1**. Au-delà, les vCPU supplémentaires sont invendables : la RAM est épuisée avant. L'hôte offre 140 vCPU pour 460 Go, soit environ 1:3,3 ; c'est le sens du « ratio vCPU/RAM entre 3 et 4 » : on dimensionne la RAM des hôtes pour qu'elle s'épuise au même rythme que le CPU vendu.
    4. 6 × 10 = 60 To bruts. Réplication ×3 : **20 To utiles** (33 %), tolère la perte de 2 copies, donc de 2 serveurs. Erasure coding 4+2 : 60 × 4/6 = **40 To utiles** (67 %), tolère 2 fragments perdus, donc 2 serveurs. Mais avec exactement 6 serveurs, il n'y a aucun serveur libre pour reconstruire : le cluster reste dégradé jusqu'à réparation. L'EC coûte aussi plus de CPU et ralentit écritures et reconstructions.
    5. SPOT = capacité inutilisée revendue à prix cassé, récupérable par le fournisseur avec un préavis très court. Adapté aux traitements interruptibles et sans état : batch, rendu, CI, tests de charge.

## E3 — SLA composites

1. Convertis en minutes d'indisponibilité par mois : 99,9 %, 99,95 %, 99,99 %.
2. Trois VM indépendantes à 99,9 % derrière un répartiteur (une seule suffit pour rendre le service) : quelle disponibilité théorique ? Compare à une VM seule à 99,95 %. Quelle hypothèse rend ce calcul optimiste ?
3. Chaîne en série : Load Balancer 99,99 % → deux VM à 99,9 % en parallèle → base de données managée 99,99 %. Disponibilité globale et minutes d'arrêt par mois ?
4. « Data 99,999999999 % » (11 neufs) sur la slide : est-ce une disponibilité ? Explique la différence avec la durabilité.
5. Si Azure ne tient pas son SLA, que reçois-tu concrètement ? Relie à la slide « délégation du risque » (p. 107).

??? success "✅ Corrigé"

    Base : un mois moyen = 730 h = 43 800 min.

    | Disponibilité | Arrêt maximal par mois |
    | --- | --- |
    | 99,9 % | 43,8 min |
    | 99,95 % | 21,9 min |
    | 99,98 % | 8,8 min |
    | 99,99 % | 4,4 min |

    2. Parallèle : 1 − 0,001³ = 1 − 10⁻⁹ = **99,9999999 %**, contre 99,95 % pour une seule VM. Hypothèse optimiste : les pannes sont supposées indépendantes. Trois VM dans la même zone, avec la même image et le même bug, tombent ensemble. Contractuellement, Azure garantit 99,99 % pour deux VM ou plus réparties sur deux zones.
    3. Série : 0,9999 × (1 − 0,001²) × 0,9999 = 0,9999 × 0,999999 × 0,9999 ≈ 0,99980 → **99,98 %, environ 8,8 min par mois**. Le maillon le plus faible n'est plus la VM mais les composants uniques en série.
    4. Non : 11 neufs, c'est la **durabilité**, la probabilité de ne pas perdre la donnée sur un an. La **disponibilité**, pouvoir y accéder à un instant donné, est bien plus basse (99,9 % à 99,99 % selon la redondance).
    5. Des **crédits de service** : un pourcentage de la facture mensuelle du service concerné, à réclamer toi-même. Pas de dommages et intérêts sur ta perte d'exploitation : le risque n'est délégué qu'en partie, d'où l'architecture redondante, les clauses contractuelles et l'assurance (p. 107).

## E4 — Plan d'adressage

1. Combien d'adresses utilisables dans un sous-réseau Azure en /24, /28 et /29 ? Quelles adresses Azure réserve-t-il dans 10.10.1.0/24 ?
2. Peut-on appairer un VNet 10.0.0.0/16 avec un VNet 10.0.128.0/17 ? Pourquoi ?
3. A est appairé à B, B à C. A peut-il joindre C ? Cite trois façons de le permettre.
4. Propose un plan d'adressage pour : un hub, trois spokes (prod, recette, dev) et un site on-premise en 192.168.0.0/16. Le hub doit accueillir `GatewaySubnet`, `AzureBastionSubnet` et une passerelle ; chaque spoke a un sous-réseau front et un back. Laisse de la place pour grandir.

??? success "✅ Corrigé"

    1. /24 : 256 − 5 = **251** ; /28 : **11** ; /29 : **3**. Réservées dans 10.10.1.0/24 : .0 (réseau), .1 (passerelle), .2 et .3 (DNS Azure), .255 (diffusion).
    2. **Non** : 10.0.128.0/17 est inclus dans 10.0.0.0/16, les plages se chevauchent.
    3. **Non**, l'appairage n'est pas transitif. Solutions : appairage direct A↔C ; passerelle (NVA ou Azure Firewall) dans B avec des routes personnalisées ; Azure Virtual WAN (p. 71).
    4. Exemple de plan, tout Azure dans 10.0.0.0/12 pour être résumé en une seule route côté on-premise :

    | Réseau | Plage | Sous-réseaux |
    | --- | --- | --- |
    | Hub | 10.0.0.0/16 | GatewaySubnet 10.0.0.0/27, AzureBastionSubnet 10.0.0.64/26, snet-nva 10.0.1.0/24 |
    | Spoke prod | 10.1.0.0/16 | front 10.1.1.0/24, back 10.1.2.0/24 |
    | Spoke recette | 10.2.0.0/16 | front 10.2.1.0/24, back 10.2.2.0/24 |
    | Spoke dev | 10.3.0.0/16 | front 10.3.1.0/24, back 10.3.2.0/24 |
    | Clients VPN P2S | 172.16.201.0/24 | — |
    | On-premise | 192.168.0.0/16 | existant, ne chevauche rien |

## E5 — Lecture de règles NSG

Un NSG est associé au sous-réseau 10.10.1.0/24 avec ces règles entrantes, en plus des règles par défaut (65000 AllowVnetInBound, 65001 AllowAzureLoadBalancerInBound, 65500 DenyAllInBound) :

| Priorité | Source | Port destination | Protocole | Action |
| --- | --- | --- | --- | --- |
| 100 | 203.0.113.10 | 22 | TCP | Allow |
| 110 | Internet | 22 | TCP | Deny |
| 120 | Internet | 80, 443 | TCP | Allow |
| 200 | 10.10.2.0/24 | \* | \* | Deny |

Pour chaque flux, donne la décision et la règle qui la prend : (a) SSH depuis 203.0.113.10 ; (b) SSH depuis 198.51.100.7 ; (c) HTTPS depuis n'importe où ; (d) SSH depuis 10.10.2.5 ; (e) ping depuis 10.10.3.4 ; (f) RDP depuis Internet ; (g) la réponse d'un serveur web du sous-réseau à un client Internet. Question bonus : la VM a aussi un NSG sur sa carte réseau qui n'autorise que le port 22. Le flux (c) passe-t-il encore ?

??? success "✅ Corrigé"

    | Flux | Décision | Règle |
    | --- | --- | --- |
    | (a) SSH depuis 203.0.113.10 | Autorisé | 100 |
    | (b) SSH depuis 198.51.100.7 | Refusé | 110 |
    | (c) HTTPS depuis n'importe où | Autorisé | 120 |
    | (d) SSH depuis 10.10.2.5 | Refusé | 200 (la 110 ne s'applique pas : une IP du VNet n'est pas « Internet ») |
    | (e) Ping depuis 10.10.3.4 | Autorisé | 65000 AllowVnetInBound |
    | (f) RDP depuis Internet | Refusé | 65500 DenyAllInBound |
    | (g) Réponse du serveur web | Autorisée | aucune : le NSG est à états, le retour d'un flux autorisé passe |

    Bonus : en entrée, le NSG du sous-réseau est évalué puis celui de la carte, et les deux doivent autoriser. Le flux (c) est donc **bloqué** par le NSG de la carte. D'où le conseil : un seul niveau de NSG, de préférence au sous-réseau.

## E6 — Migration : les 7R

Une mutuelle fictive, « MutuOuest », doit quitter son datacenter dans 9 mois. Attribue un R à chaque application (Retire, Retain, Relocate, Rehost, Repurchase, Replatform, Refactor) et justifie en une phrase :

1. Exchange 2016 sur site.
2. Intranet SharePoint 2013, presque plus consulté depuis l'arrivée de Teams.
3. Application de gestion des adhérents (.NET + SQL Server sur VM), cœur de métier, l'éditeur supporte Azure.
4. Cluster VMware de 40 VM hétérogènes, à sortir vite.
5. Moteur de tarification COBOL sur mainframe, critique, aucune compétence pour le réécrire.
6. Site vitrine PHP + MySQL.
7. Batch mensuel de calcul des remboursements, très gourmand deux jours par mois et inutilisé le reste du temps.
8. Serveur de fichiers Windows de 2 To.

Puis choisis trois KPI de la slide p. 107 pour piloter la migration et dis comment tu les mesurerais.

??? success "✅ Corrigé"

    | Application | R | Justification |
    | --- | --- | --- |
    | Exchange 2016 | Repurchase | Exchange Online fait mieux, sans serveur à maintenir. |
    | Intranet SharePoint 2013 | Retire | Plus utilisé ; archiver le contenu et éteindre. |
    | Gestion des adhérents | Rehost, puis Replatform | Lift-and-shift pour tenir le délai, puis SQL Server vers Azure SQL Managed Instance. |
    | Cluster VMware de 40 VM | Relocate | Azure VMware Solution : on déplace sans rien convertir, délai court. |
    | Tarification COBOL | Retain | Trop risqué et sans compétences ; à revisiter plus tard. |
    | Site vitrine PHP | Replatform | App Service + base MySQL managée : plus d'OS à gérer. |
    | Batch de remboursements | Refactor | Réécrit en traitement élastique (fonctions, conteneurs, batch) : on ne paie que deux jours par mois. |
    | Serveur de fichiers | Replatform | Azure Files (+ File Sync), ou Repurchase vers SharePoint/OneDrive. |

    KPI possibles : taux de réussite (applications migrées sans retour arrière), continuité de service (minutes d'arrêt réelles vs fenêtre prévue), coût (TCO avant/après sur 12 mois), accompagnement au changement (tickets support dans le mois qui suit).

## E7 — Plan de sauvegarde

PME de 50 personnes : un contrôleur AD, un serveur de fichiers de 2 To (environ 5 % de modifications par jour), un ERP sur SQL Server de 200 Go modifié en continu. Exigences : ERP RPO 1 h / RTO 4 h ; fichiers RPO 24 h / RTO 24 h ; rétention 30 jours.

1. Pour chaque système, choisis le type et la fréquence de sauvegarde (complète, incrémentale, différentielle, synthétique, journaux de transactions).
2. Estime le volume stocké pour les fichiers sur 30 jours avec « complète hebdomadaire + incrémentale quotidienne », avant compression. Que change une complète synthétique ?
3. Applique la règle 3-2-1-1-0 : où sont les copies, sur quels supports, laquelle est hors ligne ou immuable ?
4. Comment tester la restauration sans exploser les coûts d'egress ?
5. Rédige le sommaire du document de l'étape 8 (p. 113).

??? success "✅ Corrigé"

    1. **AD** : état système quotidien, et surtout un second contrôleur de domaine. **Fichiers** : complète hebdomadaire + incrémentale quotidienne (RPO 24 h respecté). **ERP** : complète quotidienne + sauvegarde des journaux de transactions toutes les 15 min (RPO ≤ 15 min, donc < 1 h) ; RTO 4 h tenable si une copie est sur site.
    2. Pour garder 30 jours restaurables, il faut environ 5 complètes (la chaîne la plus ancienne doit rester entière) : 5 × 2 To = 10 To, plus environ 26 incrémentales × 100 Go = 2,6 To, soit **≈ 12,6 To**. Une complète synthétique est fabriquée dans le dépôt à partir de la précédente et des incrémentales : aucune relecture de la production, et avec le clonage de blocs (ReFS/XFS) les complètes ne coûtent presque plus d'espace (≈ 2 To + 2,6 To).
    3. **3** copies (production, dépôt local, cloud) ; **2** supports (disques locaux, stockage objet) ; **1** hors site (cloud) ; **1** immuable ou hors ligne (bucket Object Lock ou blob immuable, ou bande) ; **0** erreur aux tests de restauration vérifiés.
    4. Tester à partir de la copie locale, ou restaurer *dans* le cloud (vers une VM de la même région) pour ne payer aucun egress ; ne rapatrier en masse qu'en vrai sinistre.
    5. Sommaire : objectifs ; périmètre et inventaire ; criticité, RPO et RTO par système ; politique (types, fréquences, rétention) ; emplacements 3-2-1-1-0 ; procédure de sauvegarde ; procédure de restauration pas à pas ; plan de tests et résultats ; supervision et alertes ; rôles ; révision annuelle.
