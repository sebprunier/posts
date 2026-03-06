---
layout: single
title: "Last Night Otoroshi Saved My Life — #3 : sécurité & performance"
excerpt: "Basic Auth pour protéger les environnements hors production, OpenID Connect pour authentifier les sociétaires MAIF sur certaines fonctionnalités d'Aux Alentours, et cache HTTP pour soulager l'API de tiles."
date: 2026-03-23
lang: fr
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, basic-auth, openid-connect, oidc, cache, security]
---

{% include lang-switcher.html %}

Troisième article de la [série Otoroshi + Clever Cloud]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). Deux cas de sécurité et un cas de performance — tous résolus au niveau d'Otoroshi, sans modifier les applications.

---

## Basic Auth — protéger les environnements hors production

### Le problème

Un environnement de développement ou de recette exposé sur Internet sans protection, c'est un risque : données de test accessibles à tous, fonctionnalités en cours de développement visibles, risque de confusion pour des utilisateurs qui tomberaient dessus par hasard.

Ajouter une authentification directement dans le code de l'application est possible, mais lourd : il faut le faire pour chaque app, gérer les credentials quelque part, penser à l'enlever (ou l'activer) selon l'environnement. Otoroshi le fait en dehors du code, en quelques secondes.

### Le plugin SimpleBasicAuth

```json
{
  "plugin": "cp:otoroshi.next.plugins.SimpleBasicAuth",
  "config": {
    "realm": "authentication-mon-app-dev",
    "users": {
      "squad.dev": "$2a$10$uK8Edv0HCj465r4enmwPAu..."
    }
  }
}
```

Le plugin intercepte toutes les requêtes. Si aucune credential valide n'est fournie, le navigateur affiche sa boîte de dialogue d'authentification native. Les mots de passe sont stockés sous forme de hash bcrypt — jamais en clair.

<pre class="mermaid">
graph LR
    Browser["🌐 Navigateur"]
    Oto["⚙️ Otoroshi<br/>SimpleBasicAuth"]
    App["🖥️ App<br/>hors prod"]

    Browser -->|"GET /"| Oto
    Oto -->|"401 WWW-Authenticate"| Browser
    Browser -->|"Authorization: Basic ..."| Oto
    Oto -->|"✅ relais"| App
</pre>

### Ce que ça change concrètement

Sur les environnements hors production de plusieurs applications, le plugin `SimpleBasicAuth` est activé sur chaque route Otoroshi. Un seul credential partagé par l'équipe suffit pour tous les environnements concernés. Pas de modification de code, pas de variable d'environnement à gérer dans l'app, pas de risque d'oublier de désactiver un mode "dev" en production.

---

## OpenID Connect — authentifier les sociétaires MAIF

### Le problème

[Aux Alentours par MAIF](https://auxalentours.maif.fr) est majoritairement un site public. Mais certaines fonctionnalités — notamment la vérification d'éligibilité à des offres MAIF — nécessitent que l'utilisateur soit identifié en tant que sociétaire MAIF. Ces fonctionnalités sont accessibles sur des paths spécifiques (`/eligibilite`, `/se-proteger/inondations/eligibilite`).

Implémenter un flux OpenID Connect dans l'application elle-même est possible, mais complexe : gestion des redirections, stockage du token, rafraîchissement de session, gestion des erreurs... Otoroshi prend en charge l'intégralité du flux à la place de l'application.

### Le plugin AuthModule

Otoroshi dispose d'un concept de **module d'authentification** : une configuration OIDC centralisée (URL du provider, client ID/secret, scopes...) réutilisable sur plusieurs routes. Le plugin `AuthModule` associe une route à l'un de ces modules.

```json
{
  "plugin": "cp:otoroshi.next.plugins.AuthModule",
  "include": [
    "/eligibilite/.*",
    "/auth/logout"
  ],
  "config": {
    "pass_with_apikey": false,
    "module": "auth_mod_5f4ae282-b762-4e59-974c-a8eed77e850c"
  }
}
```

Le paramètre `include` est clé : l'authentification n'est déclenchée que sur les pages `/eligibilite/*`, réservées aux sociétaires. Le reste du site reste public. Otoroshi gère la redirection vers le fournisseur d'identité MAIF, le callback, la validation du token, et la session utilisateur.

<pre class="mermaid">
sequenceDiagram
    participant B as Navigateur
    participant O as Otoroshi
    participant IDP as Fournisseur d'identité MAIF
    participant A as App Aux Alentours

    B->>O: GET /eligibilite
    O->>B: Redirect vers IDP
    B->>IDP: Authentification
    IDP->>O: Callback + code
    O->>IDP: Échange code → token
    O->>B: Session établie
    B->>O: GET /eligibilite (avec session)
    O->>A: Requête + header utilisateur
    A->>B: Réponse
</pre>

### Transmettre les infos utilisateur au backend

Une fois authentifié, l'application backend a besoin de connaître l'identité de l'utilisateur pour personnaliser la réponse. Le plugin `OtoroshiInfos` transmet ces informations dans un header JWT signé :

```json
{
  "plugin": "cp:otoroshi.next.plugins.OtoroshiInfos",
  "config": {
    "version": "Latest",
    "ttl": 30,
    "algo": {
      "type": "HSAlgoSettings",
      "size": 512,
      "secret": "..."
    }
  }
}
```

L'application reçoit dans chaque requête un header contenant les claims de l'utilisateur (identifiant, email, etc.), signés et datés. Elle n'a pas à valider le token OIDC elle-même — Otoroshi l'a déjà fait.

### Bonus : headers de sécurité

La route du site Aux Alentours par MAIF utilise également le plugin `AdditionalHeadersOut` pour ajouter des headers de sécurité à toutes les réponses, sans toucher à l'application :

```json
{
  "plugin": "cp:otoroshi.next.plugins.AdditionalHeadersOut",
  "config": {
    "headers": {
      "X-XSS-Protection": "1; mode=block",
      "X-Content-Type-Options": "nosniff",
      "Referrer-Policy": "origin-when-cross-origin, strict-origin-when-cross-origin",
      "X-Permitted-Cross-Domain-Policies": "master-only"
    }
  }
}
```

Même pattern : la politique de sécurité est définie dans Otoroshi, pas dans l'app.

---

## Cache — soulager l'API de tiles

### Le problème

Les tuiles d'Aux Alentours par MAIF sont des **tuiles vectorielles au format MVT** (Mapbox Vector Tiles), générées à la volée depuis une base **PostgreSQL/PostGIS** hébergée comme add-on Clever Cloud. L'API interroge directement PostGIS via [`ST_AsMVTGeom`](https://postgis.net/docs/ST_AsMVTGeom.html) pour construire chaque tuile à partir des données géographiques stockées en base.

C'est une architecture élégante — pas de fichiers de tuiles pré-générés à stocker, les données sont toujours fraîches — mais coûteuse à la requête : chaque tuile implique une requête SQL avec des calculs géométriques. Or une tuile `/{z}/{x}/{y}` pour un zoom et des coordonnées donnés est identique pour tous les utilisateurs et ne change pas d'une requête à l'autre (sauf mise à jour des données en base).

Sans cache, chaque affichage de carte déclenche des dizaines de requêtes vers le backend, qui exécute autant de requêtes PostGIS. Le cache permet de court-circuiter ce chemin pour les tuiles déjà calculées, sans toucher ni à l'API ni au schéma de base de données.

Otoroshi propose deux approches selon le besoin.

### Option 1 — ResponseCache : cache centralisé dans Redis

```json
{
  "plugin": "cp:otoroshi.next.plugins.ResponseCache",
  "config": {
    "enabled": true,
    "ttl": 3600000,
    "maxSize": 50,
    "autoClean": true,
    "filter": {
      "statuses": [200],
      "methods": ["GET"],
      "not_found": false
    }
  }
}
```

Le plugin stocke les réponses dans le datastore Redis d'Otoroshi. La clé de cache est l'URL complète — chaque combinaison `/{z}/{x}/{y}` a sa propre entrée. Une tuile mise en cache est servie directement par Otoroshi sans atteindre le backend, pour tous les utilisateurs.

```
Requête 1 : GET /tiles/12/1056/723.png  →  backend (génération) → 120ms
Requête 2 : GET /tiles/12/1056/723.png  →  cache Otoroshi       →   2ms
```

**Attention** : les tuiles sont des images binaires, et leur nombre peut être très élevé (toutes les combinaisons zoom/x/y). Il faut dimensionner `maxSize` avec soin pour ne pas saturer le Redis d'Otoroshi.

### Option 2 — HttpClientCache : déléguer au navigateur

```json
{
  "plugin": "cp:otoroshi.next.plugins.HttpClientCache",
  "config": {
    "enabled": true,
    "ttl": 3600
  }
}
```

Ce plugin ajoute des headers `Cache-Control` aux réponses, déléguant entièrement la mise en cache au navigateur (ou à tout proxy HTTP intermédiaire). Plus simple à opérer — pas de stockage côté Otoroshi — mais le cache est local à chaque client. Deux utilisateurs différents qui chargent la même tuile déclencheront chacun une requête vers le backend.

### Quelle option choisir ?

| | ResponseCache | HttpClientCache |
|---|---|---|
| Cache partagé entre tous les clients | ✅ | ❌ |
| Stockage côté Otoroshi (Redis) | ✅ à surveiller | ❌ |
| Simplicité opérationnelle | ⚠️ | ✅ |

Pour des tiles publiques à fort trafic, `HttpClientCache` est souvent le bon point de départ : il réduit significativement la charge sur les utilisateurs récurrents sans risque de saturer le Redis. `ResponseCache` est pertinent si l'on veut garantir un cache chaud même pour les nouveaux utilisateurs.

---

## Ce qu'on retient

Ces trois cas ont un point commun : **des besoins transverses résolus une fois pour toutes dans Otoroshi**, indépendamment du cycle de développement des applications.

- Basic Auth : protéger un environnement hors prod ne nécessite pas de modifier l'app ni de gérer des credentials dans le code
- OIDC : le flux d'authentification complet est géré par Otoroshi — l'app reçoit juste les infos utilisateur dans un header
- Cache : une tuile générée une fois n'est plus générée pour les requêtes suivantes — sans aucune logique de cache dans le backend

Dans le prochain et dernier article : servir des contenus sans application dédiée (ZIP, S3, assets statiques) et exposer une UI Swagger depuis un simple fichier `openapi.json`.
