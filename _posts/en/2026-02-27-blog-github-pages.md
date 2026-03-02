---
layout: single
title: "Building a Technical Blog with Jekyll and GitHub Pages"
excerpt: "I wanted a simple place to publish technical experience reports without having to manage any infrastructure. GitHub Pages + Jekyll quickly became the obvious choice"
date: 2026-02-27
lang: en
categories: [blog, jekyll]
tags: [jekyll, github-pages, minimal-mistakes, jekyll-polyglot, github-actions]
---

{% include lang-switcher.html %}

I wanted a simple place to publish technical experience reports without having to manage any infrastructure. [GitHub Pages](https://docs.github.com/fr/pages) + [Jekyll](https://jekyllrb.com/) quickly became the obvious choice: free, versioned, and automatically deployed on every push.

This blog was built in collaboration with [Claude Code](https://claude.ai/code), Anthropic’s CLI. From project initialization to debugging the language switcher, Claude Code acted as the technical counterpart throughout the session — a good example of what can be achieved through pair programming with an AI assistant. Here are the main steps.

## Initialization

The starting point is a standard GitHub repository. Jekyll generates a static website from Markdown files. The minimal structure looks like this:

```
_posts/        ← articles
assets/        ← images, CSS
_config.yml    ← site configuration
Gemfile        ← Ruby dependencies
index.md       ← homepage
```

The `_config.yml` file defines the essentials: title, URL, theme, and plugins.

## Choosing a theme: Minimal Mistakes

The default theme (Minima) is functional but quite minimal. I chose [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/), a very complete theme offering multiple color schemes. It integrates via `remote_theme`, meaning there is no need to fork the repository:

```yaml
remote_theme: mmistakes/minimal-mistakes
minimal_mistakes_skin: "air"
```

The `air` skin provides a clean and readable layout.

## Tags and navigation

Minimal Mistakes natively supports tags through a dedicated layout. You simply create a `_pages/tags.md` page with `layout: tags`, declare the archive in `_config.yml`, and configure navigation in `_data/navigation.yml`:

```yaml
tag_archive:
  type: liquid
  path: /tags/
```

## Footer customization

Minimal Mistakes provides hooks to customize the footer. I overrode `_includes/footer.html` to add links to my social networks (LinkedIn, X, GitHub) using Font Awesome icons, along with an RSS link.

One important detail: Minimal Mistakes loads `footer/custom.html` *before* `footer.html`. If both are populated, icons appear twice. You must choose one or the other.

## Multilingual support with jekyll-polyglot

This was the most ambitious part. [jekyll-polyglot](https://polyglot.untra.io/) allows generating multiple language versions of a site from a single source. Configuration in `_config.yml`:

```yaml
languages: ["fr", "en"]
default_lang: "fr"
exclude_from_localization: ["assets"]
parallel_localization: false
```

Posts are organized into language-specific subfolders:

```
_posts/
  fr/  ← French articles
  en/  ← English articles
```

Each article includes a `lang: fr` or `lang: en` attribute in its front matter.

### GitHub Actions required

`jekyll-polyglot` is not included in the list of plugins allowed by GitHub Pages. Therefore, the site must be built manually using GitHub Actions and deployed as an artifact.

In the repository settings, the GitHub Pages source must be set to **"GitHub Actions"** (not "Deploy from a branch").

The workflow `.github/workflows/jekyll.yml` installs gems, builds the site with Jekyll, and deploys using `actions/deploy-pages`.

### Interface translations

Minimal Mistakes uses `site.locale` for interface strings (reading time, dates, etc.). With polyglot, some includes must be overridden to use `page.locale` instead — especially `page__meta.html` and `page__date.html`.

Dates also need to be formatted using Liquid filters to avoid dependency on the system locale.

For the homepage, `_layouts/home.html` also requires filtering to display only posts from the active language:

{% raw %}
```liquid
{% assign posts = site.posts | where: "lang", site.active_lang %}
```
{% endraw %}

## The language switcher: an unexpected challenge

Adding a simple "FR / EN" link to every page sounds easy. In practice, polyglot introduces two complications:

1. **URL post-processing**: polyglot scans the final HTML output of the EN build and automatically adds `/en/` to all relative links. Any Liquid-generated URL pointing to the FR version gets rewritten to EN.

2. **jekyll-include-cache**: Minimal Mistakes caches includes using {% raw %}`{% include_cached %}`{% endraw %}. If the masthead uses `page.url`, the first cached value is reused for every page — meaning the language link always points to the first rendered article.

The solution in both cases: **JavaScript**. By reading `window.location.href` at runtime in the browser, we bypass both polyglot’s post-processing and include caching:

```javascript
// FR → EN: insert /en/ after baseurl
document.getElementById('masthead-en-link').href =
  window.location.href.replace('/posts/', '/posts/en/');

// EN → FR: remove /en/
document.getElementById('masthead-fr-link').href =
  window.location.href.replace('/en/', '/');
```

## Conclusion

Jekyll + GitHub Pages remains an excellent combination for a simple technical blog. Adding multilingual support with jekyll-polyglot is powerful but requires a few adjustments, especially around include caching and URL post-processing. Once these pitfalls are understood, the result is clean and fully static.

---