# Base de connaissances — consignes de rédaction et de publication

Site MkDocs Material publié sur GitHub Pages par `.github/workflows/deploy.yml` à chaque push sur `main`.
URL : https://vincentfrag.github.io/Base-de-connaissances/

## Arborescence
- `docs/reseau/`, `docs/systeme/`, `docs/transverse/` : un dossier par bloc, chacun avec un `index.md`
- **Organisation par thème** : chaque bloc est découpé en thèmes numérotés (ex. Réseau : Fondamentaux, DHCP et DNS, VLAN, Accès distant, Dépannage, S'entraîner — « S'entraîner » toujours en dernier).
  - `index.md` du bloc : légende + `grid cards` (une carte par thème : icône numérotée, titre, une phrase, liste de liens avec icône de type `:material-book-open-variant:` Comprendre / `:material-wrench-outline:` Pratiquer / `:material-flask-outline:` TP / `:material-clipboard-text-outline:` Mémo, et badge `<span class="kb-badge kb-bases">Bases</span>` ou `kb-inter` Intermédiaire)
  - `nav` de `mkdocs.yml` : une sous-section par thème ; pour un couple concept/tuto, libellés courts « Comprendre » et « Mettre en pratique »
- Noms de fichiers : `concept-<sujet>.md`, `tuto-<sujet>.md`, `memo-<sujet>.md`, `tp-<sujet>.md`
- Couleurs d'encadrés adoucies dans `docs/stylesheets/extra.css` (question, success)

## Types d'articles (en français, digestes, pédagogiques)
- 🧠 Fiche concept (1 page max) : encadré « En une phrase », analogie, explication avec tableaux, « À quoi ça sert concrètement », encadré « Pièges fréquents », liens « Pour aller plus loin »
- 🛠️ Tuto de A à Z : encadré « Objectif », niveau et durée, prérequis (liens vers les fiches concept), étapes numérotées avec commandes (onglets `=== "Windows"` / `=== "Linux"` si besoin), tableau « ce que tu vois / ce que ça veut dire / action », vérification finale, dépannage courant
- 🧪 TP (`tp-<sujet>.md`, en fin de parcours d'un bloc) : contexte d'entreprise, cahier des charges, matériel, tâches numérotées ; sous chaque tâche `??? tip "💡 Indice"` puis `??? success "✅ Solution"` (commandes complètes) ; grille de validation finale ; « Si un test échoue ». Carte « S'entraîner » dans l'index du bloc.
- 📋 Mémo : tableaux commande par besoin (colonnes Windows / Linux si pertinent)
- **Règle : toute fiche concept doit avoir son tuto pratique associé** (ex. concept-vlan → tuto-creer-vlan), avec les commandes réelles par plateforme en onglets (Cisco IOS, pfSense, Windows Server/PowerShell, Linux). La fiche concept renvoie vers son tuto en premier dans « Pour aller plus loin ».
- Niveaux (dans les articles) : 🟢 Bases · 🟡 Intermédiaire. Front matter : `icon` (concept `material/book-open-variant`, tuto `material/wrench-outline`, tp `material/flask-outline`, memo `material/clipboard-text-outline`), `title` et `tags`.

## Anonymisation (site public)
Ne jamais publier les vrais noms d'hôtes, domaine, IP, noms de VM ou comptes du lab de Vincent. Utiliser toujours ce plan générique, identique dans tous les articles :

| Élément | Nom générique | IP |
|---|---|---|
| Hôte Proxmox (rebond, passerelle) | `pve01` | `192.168.1.20` (réseau du PC) et `10.0.0.1` |
| Contrôleur de domaine / DNS | `srv-dc01` (domaine `lab.local`, NetBIOS `LAB`) | `10.0.0.10` |
| Serveur de fichiers | `srv-files01` | `10.0.0.20` |
| Répartiteur de charge | `srv-lb01` | `10.0.0.30` |
| Serveurs web | `srv-web01`, `srv-web02` | `10.0.0.31`, `10.0.0.32` |
| Template Debian | `tpl-debian01` | `10.0.0.200` |
| Postes clients | `PC-01`, `PC-02`… | DHCP |

Réseau interne `10.0.0.0/24` sur le pont `vmbr2` · compte d'admin des VM `admin` · utilisateurs d'exemple `alice`, `bruno`, `chloe`. Nouvelle machine : suivre le même schéma (`srv-<rôle>01`, IP libre du plan). Les plans VLAN pédagogiques (192.168.10.0/24, 192.168.20.0/24) restent valables.

## Publier un article
1. Écrire le fichier dans le bon dossier
2. L'ajouter dans la bonne sous-section thème de `nav` (`mkdocs.yml`) et dans la bonne carte de l'`index.md` du bloc (créer thème/carte si nouveau)
3. Vérifier : `pip install -r requirements.txt && mkdocs build --strict`
4. Commit en français (`Ajout : <titre>`) puis push sur `main`
