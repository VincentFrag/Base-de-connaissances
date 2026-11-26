# Base de connaissances — consignes de rédaction et de publication

Site MkDocs Material publié sur GitHub Pages par `.github/workflows/deploy.yml` à chaque push sur `main`.
URL : https://vincentfrag.github.io/base-de-connaissances/

## Arborescence
- `docs/reseau/`, `docs/systeme/`, `docs/transverse/` : un dossier par bloc, chacun avec un `index.md` (parcours conseillé)
- Noms de fichiers : `concept-<sujet>.md`, `tuto-<sujet>.md`, `memo-<sujet>.md`

## Types d'articles (en français, digestes, pédagogiques)
- 🧠 Fiche concept (1 page max) : encadré « En une phrase », analogie, explication avec tableaux, « À quoi ça sert concrètement », encadré « Pièges fréquents », liens « Pour aller plus loin »
- 🛠️ Tuto de A à Z : encadré « Objectif », niveau et durée, prérequis (liens vers les fiches concept), étapes numérotées avec commandes (onglets `=== "Windows"` / `=== "Linux"` si besoin), tableau « ce que tu vois / ce que ça veut dire / action », vérification finale, dépannage courant
- 📋 Mémo : tableaux commande par besoin (colonnes Windows / Linux si pertinent)
- Niveaux : 🟢 Bases · 🟡 Intermédiaire. Front matter : `title` et `tags`.

## Publier un article
1. Écrire le fichier dans le bon dossier
2. L'ajouter à `nav` dans `mkdocs.yml` et au `index.md` du bloc
3. Vérifier : `pip install -r requirements.txt && mkdocs build --strict`
4. Commit en français (`Ajout : <titre>`) puis push sur `main`
