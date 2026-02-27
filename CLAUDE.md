# Blog — Posts

Blog statique généré par **Jekyll** et hébergé sur **GitHub Pages**.

URL : `https://sebprunier.github.io/posts/`

## Structure

```
_config.yml              ← config Jekyll (thème, titre, baseurl)
_posts/                  ← un fichier Markdown par article
assets/images/           ← images référencées dans les articles
```

## Ajouter un article

1. Créer `_posts/YYYY-MM-DD-titre-de-l-article.md`
2. Ajouter le front matter en début de fichier :

```yaml
---
layout: post
title: "Titre de l'article"
date: YYYY-MM-DD
categories: [catégorie]
tags: [tag1, tag2]
---
```

3. Écrire le contenu en Markdown standard
4. Commiter et pusher → GitHub Pages rebuild automatiquement

## Images

- Placer les images dans `assets/images/`
- Les référencer dans le Markdown : `![alt]({{ "/assets/images/mon-image.png" | relative_url }})`

## Thème

Minima (thème Jekyll par défaut, supporté nativement par GitHub Pages).
Config dans `_config.yml`.
