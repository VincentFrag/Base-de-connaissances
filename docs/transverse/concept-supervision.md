---
icon: material/book-open-variant
title: La supervision (métriques, logs, alertes)
tags: [transverse, bases, concept, supervision, monitoring]
---

# 🧠 La supervision : métriques, logs et alertes

!!! abstract "En une phrase"
    **Superviser**, c'est faire surveiller ses machines et ses services **en continu par un outil**, pour être **prévenu d'une panne avant les utilisateurs** et avoir de quoi comprendre ce qui s'est passé.

## L'analogie

Le tableau de bord d'une voiture. La **jauge d'essence** et le compteur, ce sont les **métriques** (des chiffres qui évoluent). Le **voyant rouge** qui s'allume quand l'huile manque, c'est l'**alerte**. Le **carnet d'entretien** et la valise de diagnostic du garagiste, ce sont les **logs** (le récit détaillé de ce qui s'est passé). Sans tableau de bord, tu découvres la panne quand la voiture s'arrête sur l'autoroute.

## Pourquoi superviser ?

| Sans supervision | Avec supervision |
|---|---|
| On apprend la panne par un utilisateur qui râle | On reçoit une alerte, souvent avant que quelqu'un s'en rende compte |
| Le disque plein fait planter la base de données | Alerte à 85 % : on fait le ménage à temps |
| « Ça marchait hier, je crois ? » | Un historique : *le site est tombé à 3 h 12, il est revenu à 3 h 20* |
| La sauvegarde nocturne échoue en silence depuis 2 semaines | Pas de signal de la sauvegarde ce matin → alerte |
| Impossible de prouver la disponibilité du service | Un taux de disponibilité chiffré (ex. 99,9 % ce mois-ci) |

## Métriques, logs et alertes : les trois briques

| | Métriques | Logs (journaux) | Alertes |
|---|---|---|---|
| **C'est quoi** | Des **valeurs chiffrées** mesurées à intervalle régulier | Des **messages texte horodatés** écrits par les programmes | Une **notification** envoyée quand une condition est remplie |
| **Exemple** | CPU 72 %, disque 91 %, ping 3 ms, site en ligne = 1 | `Failed password for admin from 10.0.0.99 port 52144 ssh2` | « 🔴 srv-web01 ne répond plus depuis 2 min » |
| **Question à laquelle ça répond** | **Est-ce que** ça va ? Combien ? Depuis quand ? | **Pourquoi** ça ne va pas ? Que s'est-il passé ? | **Qui** doit agir, **maintenant** ? |
| **Volume** | Faible (quelques nombres) | Gros (des milliers de lignes par jour) | Très faible (idéalement rare) |
| **Où on les regarde** | Graphiques, tableaux de bord | `journalctl`, `/var/log/`, Observateur d'événements Windows | Mail, Discord, Telegram, SMS |
| **Outils typiques** | Uptime Kuma, Zabbix, Prometheus + Grafana | journald, rsyslog, Graylog, ELK, Loki | Intégrées aux outils précédents |

**Retenir :** les **métriques** disent *que* quelque chose ne va pas, les **logs** disent *pourquoi*, et l'**alerte** fait en sorte que quelqu'un s'en occupe.

## Ce qu'on surveille, du plus simple au plus fin

| Niveau | Question | Exemples de sondes |
|---|---|---|
| **Disponibilité** | La machine est-elle joignable ? | Ping (ICMP) |
| **Service** | Le service répond-il ? | Port TCP 22 (SSH), 389 (LDAP), requête DNS, page web qui renvoie `200 OK` |
| **Contenu** | Répond-il **correctement** ? | La page contient bien un mot-clé, le certificat TLS n'expire pas dans 15 jours |
| **Ressources** | A-t-il de quoi tenir ? | CPU, RAM, espace disque, charge |
| **Tâches** | Les travaux planifiés ont-ils tourné ? | La sauvegarde de la nuit a envoyé son signal |

## Deux façons de récolter l'information

| | Sonde **active** (*pull*) | Signal **passif** (*push*) |
|---|---|---|
| **Principe** | Le serveur de supervision **va vérifier** la cible | La cible **envoie** elle-même un « je suis vivant » |
| **Exemple** | Uptime Kuma pingue `srv-web01` toutes les 60 s | Le script de sauvegarde appelle une URL à la fin de son travail |
| **Idéal pour** | Services réseau, sites web | Tâches planifiées, machines derrière un pare-feu |

Et pour les ressources internes (CPU, disque), il faut **un agent** installé sur la machine (agent Zabbix, *node exporter* de Prometheus) ou un protocole comme **SNMP** (switchs, routeurs, imprimantes). Une simple sonde réseau ne voit pas le disque qui se remplit.

## Quel outil pour un homelab ?

| Outil | Ce qu'il fait bien | Mise en route | Pour qui |
|---|---|---|---|
| **Uptime Kuma** | Disponibilité : ping, HTTP, TCP, DNS, signal *push*, page de statut, alertes faciles | 1 conteneur, 10 min | Homelab, petite structure : **le meilleur point de départ** |
| **Zabbix** | Tout : agents, CPU/RAM/disque, SNMP, modèles prêts pour Linux, Windows, Proxmox | Serveur + base de données + agents, 1 à 2 h | Entreprise, BTS/CESI, quand on veut les ressources |
| **Prometheus + Grafana** | Métriques très fines et beaux graphiques | Plusieurs briques à assembler | Conteneurs, Kubernetes, profils DevOps |
| **Centreon** (héritier de Nagios) | Supervision par SNMP ou agent, très répandue en France (entreprises, collectivités) | Serveur central + base, 2 h | Pour le croiser en stage ou en entreprise |

**Le parcours proposé :** on commence par **Uptime Kuma**, simple et visuel, qui couvre déjà 80 % du besoin (« est-ce que c'est en ligne ? »). Puis **Zabbix** pour suivre l'intérieur des VM (CPU, RAM, disques) avec des agents, et **Centreon**, très répandu dans les entreprises françaises, pour découvrir la supervision par SNMP.

## À quoi ça sert concrètement

- Savoir en un coup d'œil quelles VM du lab sont en ligne (`srv-dc01`, `srv-web01`, `srv-lb01`…).
- Être prévenu sur son téléphone quand un site tombe, et quand il revient.
- Vérifier que la sauvegarde de la nuit a vraiment tourné.
- Anticiper : certificat qui expire, disque qui se remplit.
- Montrer une page de statut et un taux de disponibilité (utile en entreprise, et en soutenance).

!!! warning "Pièges fréquents"
    - **Superviser depuis la machine supervisée** : si le serveur tombe, la supervision tombe avec et personne n'est prévenu. Elle doit tourner **ailleurs** (autre VM au minimum, idéalement autre machine).
    - **Trop d'alertes** : à la vingtième fausse alerte, plus personne ne les lit. Seuils réalistes, plusieurs échecs avant d'alerter, et une alerte = une action à faire.
    - **Ping OK ≠ service OK** : la VM répond au ping alors qu'Apache est arrêté. Toujours ajouter une sonde **du service** (HTTP, port).
    - **Oublier le pare-feu** : Windows bloque le ping par défaut, la sonde est rouge alors que le serveur va bien.
    - **Aucune alerte configurée** : un beau tableau de bord que personne ne regarde ne prévient de rien.
    - **Ne jamais tester l'alerte** : on découvre le jour de la panne que le mail part dans les spams.

## Pour aller plus loin

- 🛠️ **[Superviser ses VM avec Uptime Kuma](tuto-uptime-kuma.md)** : la mise en pratique (installation, sondes ping/HTTP/TCP/DNS, alertes, signal de sauvegarde)
- 🛠️ [Superviser ses VM avec Zabbix](tuto-zabbix.md) : agents Linux et Windows, modèles, API Proxmox
- 🛠️ [Superviser ses VM avec Centreon](tuto-centreon.md) : SNMP, connecteurs de supervision, export de configuration
- 🧠 [Les sauvegardes et la règle 3-2-1](concept-sauvegardes.md) : la supervision sert aussi à vérifier qu'elles tournent
- 🛠️ [Diagnostiquer une connexion](../reseau/tuto-diagnostic-connexion.md) : quand l'alerte tombe, par où commencer
