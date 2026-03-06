---
layout: single
title: "Last Night Otoroshi Saved My Life — #3: security & performance"
excerpt: "Basic Auth to protect non-production environments, OpenID Connect to authenticate MAIF members on specific Aux Alentours features, and HTTP caching to offload the tile API."
date: 2026-03-23
lang: en
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, basic-auth, openid-connect, oidc, cache, security]
---

{% include lang-switcher.html %}

Third article in the [Otoroshi + Clever Cloud series]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). Two security use cases and one performance use case — all solved at the Otoroshi level, without modifying the applications.

---

## Basic Auth — protecting non-production environments

### The problem

A development or staging environment exposed on the internet without protection is a risk: test data accessible to anyone, in-progress features visible, risk of confusion for users who stumble upon it.

Adding authentication directly in application code is possible, but cumbersome: it needs to be done for each app, credentials need to be managed somewhere, and you need to remember to toggle it per environment. Otoroshi handles it outside the code, in seconds.

### The SimpleBasicAuth plugin

```json
{
  "plugin": "cp:otoroshi.next.plugins.SimpleBasicAuth",
  "config": {
    "realm": "authentication-my-app-dev",
    "users": {
      "squad.dev": "$2a$10$uK8Edv0HCj465r4enmwPAu..."
    }
  }
}
```

The plugin intercepts all requests. If no valid credentials are provided, the browser displays its native authentication dialog. Passwords are stored as bcrypt hashes — never in plain text.

<pre class="mermaid">
graph LR
    Browser["🌐 Browser"]
    Oto["⚙️ Otoroshi<br/>SimpleBasicAuth"]
    App["🖥️ App<br/>non-prod"]

    Browser -->|"GET /"| Oto
    Oto -->|"401 WWW-Authenticate"| Browser
    Browser -->|"Authorization: Basic ..."| Oto
    Oto -->|"✅ forward"| App
</pre>

### What this changes in practice

On non-production environments across multiple applications, the `SimpleBasicAuth` plugin is enabled on each Otoroshi route. A single shared credential for the team covers all concerned environments. No code changes, no environment variables to manage in the app, no risk of accidentally leaving a "dev mode" enabled in production.

---

## OpenID Connect — authenticating MAIF members

### The problem

[Aux Alentours par MAIF](https://auxalentours.maif.fr) is mostly a public site. But some features — particularly eligibility checks for MAIF offerings — require the user to be identified as a MAIF member. These features are available on `/eligibilite/*` paths.

Implementing an OpenID Connect flow in the application itself is possible, but complex: managing redirects, storing tokens, refreshing sessions, handling errors... Otoroshi takes over the entire flow on behalf of the application.

### The AuthModule plugin

Otoroshi has a concept of **authentication modules**: a centralized OIDC configuration (provider URL, client ID/secret, scopes...) reusable across multiple routes. The `AuthModule` plugin associates a route with one of these modules.

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

The `include` parameter is key: authentication is only triggered on `/eligibilite/*` pages, reserved for members. The rest of the site remains public. Otoroshi handles the redirect to the MAIF identity provider, the callback, token validation, and session management.

<pre class="mermaid">
sequenceDiagram
    participant B as Browser
    participant O as Otoroshi
    participant IDP as MAIF Identity Provider
    participant A as Aux Alentours App

    B->>O: GET /eligibilite
    O->>B: Redirect to IDP
    B->>IDP: Authentication
    IDP->>O: Callback + code
    O->>IDP: Exchange code for token
    O->>B: Session established
    B->>O: GET /eligibilite (with session)
    O->>A: Request + user header
    A->>B: Response
</pre>

### Passing user info to the backend

Once authenticated, the backend application needs to know the user's identity to personalize the response. The `OtoroshiInfos` plugin passes this information in a signed JWT header:

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

The application receives a header in every request containing the user's claims (identifier, email, etc.), signed and timestamped. It does not need to validate the OIDC token itself — Otoroshi has already done that.

### Bonus: security headers

The Aux Alentours par MAIF route also uses the `AdditionalHeadersOut` plugin to add security headers to all responses, without touching the application:

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

Same pattern: the security policy is defined in Otoroshi, not in the app.

---

## Cache — offloading the tile API

### The problem

The Aux Alentours par MAIF tiles are **vector tiles in MVT format** (Mapbox Vector Tiles), generated on the fly from a **PostgreSQL/PostGIS** database hosted as a Clever Cloud add-on. The API queries PostGIS directly via [`ST_AsMVTGeom`](https://postgis.net/docs/ST_AsMVTGeom.html) to build each tile from the geographic data stored in the database.

It's an elegant architecture — no pre-generated tile files to store, data is always fresh — but expensive per request: each tile involves a SQL query with geometric computations. Yet a tile `/{z}/{x}/{y}` for a given zoom level and coordinates is identical for all users and does not change between requests (unless the underlying data is updated).

Without caching, every map view triggers dozens of requests to the backend, each executing a PostGIS query. Caching short-circuits this path for already-computed tiles, without touching the API or the database schema.

Otoroshi offers two approaches depending on the need.

### Option 1 — ResponseCache: centralized cache in Redis

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

The plugin stores responses in Otoroshi's Redis datastore. The cache key is the full URL — each `/{z}/{x}/{y}` combination has its own entry. A cached tile is served directly by Otoroshi without reaching the backend, for all users.

```
Request 1: GET /tiles/12/1056/723.png  →  backend (generation) → 120ms
Request 2: GET /tiles/12/1056/723.png  →  Otoroshi cache       →   2ms
```

**Warning**: tiles are binary images, and their number can be very large (all zoom/x/y combinations). `maxSize` must be sized carefully to avoid saturating Otoroshi's Redis.

### Option 2 — HttpClientCache: delegate to the browser

```json
{
  "plugin": "cp:otoroshi.next.plugins.HttpClientCache",
  "config": {
    "enabled": true,
    "ttl": 3600
  }
}
```

This plugin adds `Cache-Control` headers to responses, delegating caching entirely to the browser (or any intermediate HTTP proxy). Simpler to operate — no storage on the Otoroshi side — but the cache is local to each client. Two different users loading the same tile will each trigger a request to the backend.

### Which option to choose?

| | ResponseCache | HttpClientCache |
|---|---|---|
| Shared cache across all clients | ✅ | ❌ |
| Storage on Otoroshi side (Redis) | ✅ monitor closely | ❌ |
| Operational simplicity | ⚠️ | ✅ |

For public tiles with high traffic, `HttpClientCache` is often the right starting point: it significantly reduces load for returning users without risking Redis saturation. `ResponseCache` is relevant when you want a warm cache even for first-time visitors.

---

## Key takeaways

These three cases share a common trait: **cross-cutting concerns resolved once in Otoroshi**, independently of the application development cycle.

- Basic Auth: protecting a non-prod environment requires no code changes and no credentials managed in the app
- OIDC: the complete authentication flow is handled by Otoroshi — the app just receives user info in a header
- Cache: a tile generated once is not generated again for subsequent requests — with no caching logic in the backend

Next and final article (*Funny Features*): serving content without a dedicated application (ZIP, S3, static assets) and exposing a Swagger UI from a simple `openapi.json` file.
