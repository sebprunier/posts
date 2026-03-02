# Blog — sebprunier/posts

Blog statique généré par **Jekyll** et hébergé sur **GitHub Pages**.

URL : `https://sebprunier.github.io/posts/`

## Stack

- **Thème** : [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) (`mmistakes/minimal-mistakes`), skin `air`, via `jekyll-remote-theme`
- **Multilingue** : [jekyll-polyglot](https://polyglot.untra.io/) — FR (défaut) + EN
- **Déploiement** : GitHub Actions (`.github/workflows/jekyll.yml`) — obligatoire car jekyll-polyglot n'est pas dans la whitelist GitHub Pages

## Structure

```
_config.yml                  ← config Jekyll (thème, plugins, polyglot, defaults)
Gemfile                      ← dépendances Ruby
.github/workflows/jekyll.yml ← build & deploy via GitHub Actions

_posts/
  fr/                        ← articles en français
  en/                        ← articles en anglais

_pages/
  tags.md                    ← page tags FR
  tags-en.md                 ← page tags EN
  index-en.md                ← home EN

_data/
  navigation.yml             ← navigation FR
  en/navigation.yml          ← navigation EN (override polyglot)
  ui-text.yml                ← (dans le gem MM, pas surchargé)

_layouts/
  home.html                  ← surcharge MM : filtre posts par langue

_includes/
  lang-switcher.html         ← switcher FR/EN inline (dans les articles)
  masthead.html              ← surcharge MM : ajoute switcher + hamburger mobile
  footer.html                ← surcharge MM : liens sociaux + copyright bilingue
  footer/custom.html         ← vide intentionnellement (évite double rendu)
  page__meta.html            ← surcharge MM : utilise page.locale
  page__date.html            ← surcharge MM : utilise page.locale + custom_date.html
  custom_date.html           ← formatage de date en Liquid (FR/EN)
  post_pagination.html       ← vide intentionnellement (supprime liens précédent/suivant)

assets/
  css/main.scss              ← styles custom (lang-switcher, footer, masthead mobile)
  images/                    ← images des articles (sous-dossier par article)
```

## Ajouter un article

1. Créer les deux fichiers (FR + EN) :
   - `_posts/fr/YYYY-MM-DD-slug.md`
   - `_posts/en/YYYY-MM-DD-slug.md`

2. Front matter obligatoire :

```yaml
---
layout: single
title: "Titre de l'article"
excerpt: "Résumé affiché sur la home."
date: YYYY-MM-DD
lang: fr          # ou 'en'
categories: [cat]
tags: [tag1, tag2]
---

{% include lang-switcher.html %}
```

3. Les deux fichiers doivent avoir **le même slug et la même date** pour que le switcher de langue fonctionne.

4. Commiter et pusher → GitHub Actions rebuild et déploie automatiquement.

## Images

- Placer les images dans `assets/images/YYYY-MM-DD-slug/`
- Les référencer dans le Markdown :

```liquid
![alt]({{ "/assets/images/2026-02-27-slug/image.png" | relative_url }})
```

## Multilingue — points d'attention

- **`site.active_lang`** : variable polyglot (langue du build en cours). Utiliser de préférence à `page.lang` dans les includes mis en cache.
- **URLs dans les includes cachés** : ne jamais utiliser `page.url` dans un include appelé via `{% include_cached %}` (valeur figée au premier rendu). Utiliser JavaScript + `window.location.href` à la place.
- **Post-traitement polyglot** : dans le build EN, polyglot ajoute `/en/` à tous les liens relatifs du HTML final. Les liens cross-langue doivent être construits en JS au runtime pour échapper à ce post-traitement.
- **Filtre posts par langue** : `site.posts | where: "lang", site.active_lang`
- **Dates** : utiliser `{% include custom_date.html date=page.date lang=page.lang %}` pour un formatage localisé sans dépendance à la locale système.

## Déploiement local

```bash
bundle exec jekyll serve
# → http://localhost:4000/posts/
```
