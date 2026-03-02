---
layout: single
title: "Créer un blog technique avec Jekyll et GitHub Pages"
excerpt: "J'avais envie d'un endroit simple pour publier des retours d'expérience tech, sans infrastructure à gérer. GitHub Pages + Jekyll s'est imposé naturellement."
date: 2026-02-27
lang: fr
categories: [blog, jekyll]
tags: [jekyll, github-pages, minimal-mistakes, jekyll-polyglot, github-actions]
---

{% include lang-switcher.html %}

J'avais envie d'un endroit simple pour publier des retours d'expérience tech, sans infrastructure à gérer. [GitHub Pages](https://docs.github.com/fr/pages) + [Jekyll](https://jekyllrb.com/) s'est imposé naturellement : gratuit, versionné, déployé automatiquement à chaque push.

Ce blog a été mis en place en collaboration avec [Claude Code](https://claude.ai/code), le CLI d'Anthropic. De l'initialisation du projet jusqu'au débogage du switcher de langue, Claude Code a été l'interlocuteur technique tout au long de la session — un bon exemple de ce qu'on peut accomplir en pair-programming avec un assistant IA. Voici les principales étapes.

## Initialisation

Le point de départ est un dépôt GitHub classique. Jekyll génère un site statique à partir de fichiers Markdown. La structure minimale :

```
_posts/        ← les articles
assets/        ← images, CSS
_config.yml    ← configuration du site
Gemfile        ← dépendances Ruby
index.md       ← page d'accueil
```

Le fichier `_config.yml` déclare l'essentiel : titre, URL, thème, plugins.

## Choix du thème : Minimal Mistakes

Le thème par défaut (Minima) est fonctionnel mais assez sobre. J'ai opté pour [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/), un thème très complet avec plusieurs variantes de couleurs. Il s'intègre via `remote_theme` sans avoir à forker le dépôt :

```yaml
remote_theme: mmistakes/minimal-mistakes
minimal_mistakes_skin: "air"
```

Le skin `air` donne un rendu clair et agréable pour la lecture.

## Tags et navigation

Minimal Mistakes supporte nativement les tags via un layout dédié. Il suffit de créer une page `_pages/tags.md` avec `layout: tags`, de déclarer l'archive dans `_config.yml`, et d'alimenter la navigation dans `_data/navigation.yml` :

```yaml
tag_archive:
  type: liquid
  path: /tags/
```

## Personnalisation du footer

MM propose des hooks pour personnaliser le pied de page. J'ai surchargé `_includes/footer.html` pour y mettre les liens vers mes réseaux sociaux (LinkedIn, X, GitHub) avec les icônes Font Awesome, et un lien RSS. Un point d'attention : MM appelle `footer/custom.html` *avant* `footer.html` — si on remplit les deux, les icônes apparaissent en double. Il faut choisir l'un ou l'autre.

## Gestion du multilingue avec jekyll-polyglot

C'est la partie la plus ambitieuse. [jekyll-polyglot](https://polyglot.untra.io/) permet de générer plusieurs versions linguistiques du site à partir d'une même source. La configuration dans `_config.yml` :

```yaml
languages: ["fr", "en"]
default_lang: "fr"
exclude_from_localization: ["assets"]
parallel_localization: false
```

Les articles sont organisés en sous-dossiers par langue :

```
_posts/
  fr/  ← articles en français
  en/  ← articles en anglais
```

Chaque article porte un attribut `lang: fr` ou `lang: en` dans son front matter.

### GitHub Actions obligatoire

jekyll-polyglot n'est pas dans la liste des plugins autorisés par GitHub Pages. Il faut donc passer par GitHub Actions pour builder soi-même et déployer l'artifact. Dans les paramètres du dépôt, la source de GitHub Pages doit être positionnée sur **"GitHub Actions"** (et non "Deploy from a branch").

Le workflow `.github/workflows/jekyll.yml` installe les gems, build avec Jekyll, puis déploie via `actions/deploy-pages`.

### Traductions de l'interface

MM utilise `site.locale` pour les chaînes de l'interface (temps de lecture, date, etc.). Avec polyglot, il faut surcharger les includes concernés pour utiliser `page.locale` à la place — notamment `page__meta.html` et `page__date.html` — et formater les dates en Liquid pour éviter toute dépendance à la locale système.

Pour la page d'accueil, `_layouts/home.html` a également besoin d'un filtre pour n'afficher que les articles de la langue courante :

{% raw %}
```liquid
{% assign posts = site.posts | where: "lang", site.active_lang %}
```
{% endraw %}

## Le switcher de langue : un parcours du combattant

Proposer un lien "FR / EN" sur chaque page semble simple. En pratique, polyglot crée deux complications :

1. **Post-traitement des URLs** : polyglot scanne le HTML final du build EN et ajoute `/en/` à tous les liens relatifs. Toute URL construite en Liquid pour pointer vers la version FR se retrouve transformée vers la version EN.

2. **jekyll-include-cache** : MM met en cache les includes avec {% raw %}`{% include_cached %}`{% endraw %}. Si le masthead (header global) utilise `page.url`, la première valeur mise en cache est réutilisée pour toutes les pages — le lien pointe alors toujours vers le premier article rendu, quelle que soit la page courante.

La solution dans les deux cas : **JavaScript**. En lisant `window.location.href` au runtime dans le navigateur, on échappe complètement au post-traitement de polyglot et au cache des includes :

```javascript
// FR → EN : insérer /en/ après le baseurl
document.getElementById('masthead-en-link').href =
  window.location.href.replace('/posts/', '/posts/en/');

// EN → FR : retirer /en/
document.getElementById('masthead-fr-link').href =
  window.location.href.replace('/en/', '/');
```

## Bilan

Jekyll + GitHub Pages reste une excellente combinaison pour un blog technique simple. L'ajout du multilingue avec jekyll-polyglot est puissant mais demande quelques ajustements, notamment autour du cache des includes et du post-traitement des URLs. Une fois ces pièges identifiés, le résultat est propre et entièrement statique.

---