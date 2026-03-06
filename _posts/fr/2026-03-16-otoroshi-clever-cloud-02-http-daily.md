---
layout: single
title: "Last Night Otoroshi Saved My Life — #2 : HTTP au quotidien"
excerpt: "CORS pour l'intégration de tiles dans du code JS, robots.txt sans toucher à l'app, redirections HTTP avec capture de path params, suppression de headers de sécurité pour intégrer une iframe en dev."
date: 2026-03-16
lang: fr
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, cors, robots, redirect, iframe, http]
---

{% include lang-switcher.html %}

Deuxième article de la [série Otoroshi + Clever Cloud]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). Quatre cas HTTP du quotidien — des situations concrètes que tout projet peut rencontrer, résolues en quelques clics dans Otoroshi sans modifier le code des applications.

---

## CORS — intégrer les tiles dans du code JS tiers

### Le problème

L'API d'[Aux Alentours par MAIF](https://auxalentours.maif.fr) expose un service de tuiles cartographiques sur `tiles.auxalentours.innovation.maif`. Le site Aux Alentours par MAIF lui-même n'a pas de problème CORS : les requêtes vers les tiles transitent par un proxy côté serveur (un "passe-plat" backend), ce qui évite complètement les restrictions du navigateur.

Mais pour un tiers qui souhaite intégrer ces tiles directement dans du code JavaScript — une carte [Leaflet](https://leafletjs.com/) par exemple — le problème se pose immédiatement. Le navigateur bloque la requête car l'origine de l'application (`mon-app.example.com`) est différente du domaine des tiles (`tiles.auxalentours.innovation.maif`).

```
Access to fetch 'https://tiles.auxalentours.innovation.maif/1/2/3.png'
from origin 'https://mon-app.example.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

### La solution avec Otoroshi

Le plugin **`Cors`** d'Otoroshi ajoute les headers CORS appropriés aux réponses, sans toucher à l'application backend.

```json
{
  "plugin": "cp:otoroshi.next.plugins.Cors",
  "config": {
    "allow_origin": "*",
    "allow_methods": ["GET", "OPTIONS"],
    "allow_headers": ["Accept"],
    "expose_headers": [],
    "max_age": 86400
  }
}
```

Avec cette configuration sur la route tiles, chaque réponse inclut :

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS
```

Le navigateur accepte la requête, et Leaflet peut charger les tiles directement.

### Adapter selon les besoins

`allow_origin: "*"` convient pour des tiles publiques. Pour un accès plus restreint — ne permettre l'intégration qu'à des domaines partenaires connus — on peut remplacer par une liste explicite :

```json
"allow_origin": "https://partenaire.example.com"
```

Otoroshi gère aussi les requêtes préliminaires `OPTIONS` (`preflight`) automatiquement.

---

## Robots — contrôler l'indexation sans toucher à l'app

### Le problème

Par défaut, les moteurs de recherche explorent tout ce qu'ils peuvent atteindre. Pour des APIs, des environnements de dev/recette ou des outils internes, être indexé par Google est au mieux inutile, au pire problématique (données de test exposées, URLs internes référencées, etc.).

La solution classique est d'ajouter un fichier `robots.txt` et des meta-tags dans l'application — mais ça implique de modifier chaque app. Sur Aux Alentours par MAIF, on a plusieurs applications et plusieurs environnements. Otoroshi le fait pour toutes, en un seul endroit.

### Le plugin Robots

Le plugin **`Robots`** d'Otoroshi agit sur trois niveaux simultanément :

```json
{
  "plugin": "cp:otoroshi.next.plugins.Robots",
  "config": {
    "robot_txt_enabled": true,
    "robot_txt_content": "User-agent: *\nDisallow: /\n",
    "meta_enabled": true,
    "meta_content": "noindex,nofollow,noarchive",
    "header_enabled": true,
    "header_content": "noindex, nofollow, noarchive"
  }
}
```

- **`robots.txt`** — Otoroshi répond directement aux requêtes vers `/robots.txt` avec le contenu configuré. L'application n'a pas à exposer ce fichier.
- **Meta tag** — injecte `<meta name="robots" content="noindex,nofollow,noarchive">` dans les réponses HTML.
- **Header HTTP** — ajoute `X-Robots-Tag: noindex, nofollow, noarchive` à toutes les réponses.

### Une seule exception : le site public en prod

Sur Aux Alentours par MAIF, le plugin Robots est activé sur **toutes les routes** avec `Disallow: /` — à une exception près : le site public `auxalentours.maif.fr` en production, qui lui doit être indexé par les moteurs de recherche.

Pour l'API, bloquer les crawlers est logique : ses réponses sont du JSON, pas des pages web. Pour les environnements de développement et de recette, c'est encore plus important : une URL de recette indexée peut remonter dans les résultats de recherche et créer de la confusion — sans qu'aucun développeur ne l'ait voulu.

Résultat : zéro `robots.txt` à maintenir dans les applications, zéro risque d'oubli sur un nouvel environnement. Le crawling n'est autorisé qu'à un seul endroit, explicitement.

---

## Redirections HTTP — migrer un domaine sans downtime

### Le contexte

`evenements.maif.fr` est le nouveau domaine d'un site d'événements MAIF. Des flyers ont été imprimés en avance avec ces URLs — mais au moment de la distribution, l'ancien site (`maif-evenements.fr`) était toujours actif et le nouveau domaine n'était pas encore opérationnel.

La solution : configurer les redirections dans Otoroshi pour que tout le trafic arrivant sur `evenements.maif.fr` soit renvoyé vers `maif-evenements.fr`, le temps que la migration soit finalisée — y compris les URLs avec des paths spécifiques comme les pages par département (`evenements.maif.fr/42`).

### Route 1 — redirection globale

```json
"frontend": {
  "domains": ["evenements.maif.fr"]
},
"plugins": [{
  "plugin": "cp:otoroshi.next.plugins.Redirection",
  "config": {
    "code": 303,
    "to": "https://maif-evenements.fr"
  }
}]
```

Tout le trafic sur `evenements.maif.fr` est redirigé vers `https://maif-evenements.fr`. Le plugin `Redirection` court-circuite la requête en phase `pre_route` — elle n'atteint jamais le backend. C'est pourquoi le backend configuré est `request.otoroshi.io`, le backend fictif interne d'Otoroshi, utilisé quand aucune cible réelle n'est nécessaire.

### Route 2 — redirection avec capture de path param

```json
"frontend": {
  "domains": ["evenements.maif.fr/$departement<[0-9]+>"]
},
"plugins": [{
  "plugin": "cp:otoroshi.next.plugins.Redirection",
  "config": {
    "code": 303,
    "to": "https://maif-evenements.fr/dep${req.pathparams.departement}"
  }
}]
```

Les pages département (`evenements.maif.fr/42`) doivent rediriger vers leur équivalent sur le nouveau domaine (`maif-evenements.fr/dep42`). La syntaxe `$departement<[0-9]+>` dans le frontend capture la valeur numérique du path, et `${req.pathparams.departement}` la réinjecte dans l'URL cible.

Cette route est plus spécifique que la précédente et doit être définie en premier — Otoroshi évalue les routes dans l'ordre et s'arrête à la première correspondance.

Une fois la migration terminée, supprimer les redirections a consisté à désactiver les deux routes dans Otoroshi. Rien de plus simple.

<pre class="mermaid">
graph LR
    A["evenements.maif.fr"]
    B["evenements.maif.fr/42"]
    Oto["⚙️ Otoroshi"]
    C["maif-evenements.fr"]
    D["maif-evenements.fr/dep42"]

    A -->|"303"| Oto
    B -->|"303"| Oto
    Oto --> C
    Oto --> D
</pre>

---

## iFrame — intégrer une app tierce en développement local

### Le contexte

Un site (appelons-le **Site appelant**) intègre en iframe une application partenaire (appelons-la **iFrame partenaire**). En production, tout fonctionne : le Site appelant est hébergé sur un domaine HTTPS connu, et l'iFrame partenaire s'y charge sans problème.

Mais en développement local, le Site appelant tourne sur `localhost`. Or l'iFrame partenaire renvoie des headers de sécurité stricts — notamment une `Content-Security-Policy` avec `frame-ancestors` — qui autorisent uniquement l'intégration depuis le domaine de production. Résultat : l'iframe est bloquée par le navigateur, impossible de développer et tester la page localement.

Deux problèmes à résoudre :

1. **Les headers de sécurité** — l'iFrame partenaire envoie des headers (`Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`...) qui empêchent le chargement dans une iframe depuis `localhost`.
2. **Les assets** — l'iFrame partenaire charge ses propres fichiers CSS depuis son domaine. En dev local, on veut pouvoir les remplacer par une version locale pour itérer rapidement.

### La solution avec Otoroshi

Une route Otoroshi fait office de proxy entre `localhost` et l'iFrame partenaire. Le Site appelant en local pointe vers cette route plutôt que directement vers l'URL partenaire. Elle résout les deux problèmes :

**Suppression des headers de sécurité** avec le plugin `RemoveHeadersOut` :

```json
{
  "plugin": "cp:otoroshi.next.plugins.RemoveHeadersOut",
  "config": {
    "names": [
      "content-security-policy",
      "content-security-policy-report-only",
      "Report-To",
      "strict-transport-security",
      "x-content-security-policy",
      "x-content-type-options"
    ]
  }
}
```

Les headers sont supprimés des réponses avant qu'elles atteignent le navigateur. L'iframe peut se charger normalement.

**Remplacement d'une URL CSS** avec le plugin `NgHtmlPatcher` :

```json
{
  "plugin": "cp:otoroshi.next.plugins.NgHtmlPatcher",
  "config": {
    "append_body": [
      "<script>
        [...document.querySelectorAll('link')].forEach(link => {
          if (link.href.startsWith('https://widget.rec.external-partner.com')) {
            link.href = 'http://localhost:8080/css/widget.css';
          }
        });
      </script>"
    ]
  }
}
```

Otoroshi injecte un petit script dans le HTML qui remplace à la volée les références CSS de l'iFrame partenaire par des URLs locales. La page charge alors le CSS en cours de développement.

### Ce que ça évite

Sans Otoroshi, les options auraient été : demander à l'équipe partenaire de modifier leurs headers de sécurité (hors de portée), désactiver les protections dans le navigateur (dangereux, non reproductible en équipe), ou monter un proxy custom maison (temps de dev, maintenance). Otoroshi règle ça en quelques minutes, le temps de configurer deux plugins.

---

## Ce qu'on retient

Ces quatre cas illustrent un pattern commun : **Otoroshi intercepte et transforme le trafic HTTP là où le code de l'application ne peut pas ou ne doit pas intervenir**.

- CORS : le backend ne sait pas qui consomme ses tiles — Otoroshi ajoute les headers selon le contexte d'utilisation
- Robots : les apps n'ont pas à connaître les politiques d'indexation — Otoroshi les applique uniformément sur tous les environnements
- Redirections : le serveur d'origine n'existe plus ou n'est pas modifiable — Otoroshi prend le relais
- iFrame : l'application tierce est hors de portée — Otoroshi adapte ses réponses pour le contexte de développement

Dans le prochain article : sécurisation d'environnements avec Basic Auth, authentification OpenID Connect pour les sociétaires MAIF, et mise en cache des tiles.
