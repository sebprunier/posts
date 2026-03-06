---
layout: single
title: "Last Night Otoroshi Saved My Life — #2: everyday HTTP"
excerpt: "CORS for integrating tiles in third-party JS code, robots.txt without touching the app, HTTP redirects with path param capture, removing security headers to embed an iframe in dev."
date: 2026-03-16
lang: en
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, cors, robots, redirect, iframe, http]
---

{% include lang-switcher.html %}

Second article in the [Otoroshi + Clever Cloud series]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). Four everyday HTTP cases — concrete situations any project can encounter, solved in a few clicks in Otoroshi without modifying application code.

---

## CORS — integrating tiles in third-party JS code

### The problem

The [Aux Alentours par MAIF](https://auxalentours.maif.fr) API exposes a map tile service at `tiles.auxalentours.innovation.maif`. The Aux Alentours par MAIF frontend itself has no CORS issue: tile requests go through a server-side proxy (a backend pass-through), which completely bypasses browser restrictions.

But for a third party wanting to integrate these tiles directly in JavaScript — a [Leaflet](https://leafletjs.com/) map for example — the problem is immediate. The browser blocks the request because the application's origin (`my-app.example.com`) differs from the tile domain (`tiles.auxalentours.innovation.maif`).

```
Access to fetch 'https://tiles.auxalentours.innovation.maif/1/2/3.png'
from origin 'https://my-app.example.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

### The solution with Otoroshi

The **`Cors`** plugin adds the appropriate CORS headers to responses, without touching the backend application.

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

With this configuration on the tiles route, every response includes:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, OPTIONS
```

The browser accepts the request, and Leaflet can load tiles directly.

### Adapting as needed

`allow_origin: "*"` is suitable for public tiles. For more restricted access — only allowing integration from known partner domains — replace with an explicit value:

```json
"allow_origin": "https://partner.example.com"
```

Otoroshi also handles `OPTIONS` preflight requests automatically.

---

## Robots — controlling indexing without touching the app

### The problem

By default, search engines crawl everything they can reach. For APIs, dev/staging environments, or internal tools, being indexed by Google is at best useless, at worst problematic (test data exposed, internal URLs referenced, etc.).

The classic solution is to add a `robots.txt` file and meta tags in the application — but that means modifying each app. On Aux Alentours par MAIF, there are multiple applications and multiple environments. Otoroshi handles it for all of them, in one place.

### The Robots plugin

The **`Robots`** plugin acts on three levels simultaneously:

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

- **`robots.txt`** — Otoroshi responds directly to requests to `/robots.txt` with the configured content. The application does not need to expose this file.
- **Meta tag** — injects `<meta name="robots" content="noindex,nofollow,noarchive">` into HTML responses.
- **HTTP header** — adds `X-Robots-Tag: noindex, nofollow, noarchive` to all responses.

### One exception: the public site in production

On Aux Alentours par MAIF, the Robots plugin is enabled on **every route** with `Disallow: /` — with one exception: the public site `auxalentours.maif.fr` in production, which does need to be indexed by search engines.

For the API, blocking crawlers makes sense: its responses are JSON, not web pages. For dev and staging environments, it matters even more: a staging URL that gets indexed can surface in search results and confuse users — without any developer having intended it.

Result: zero `robots.txt` files to maintain in applications, zero risk of forgetting on a new environment. Crawling is only allowed in one place, explicitly.

---

## HTTP redirects — migrating a domain without downtime

### The context

`evenements.maif.fr` is the new domain for a MAIF events site. Flyers were printed in advance with these URLs — but at the time of distribution, the old site (`maif-evenements.fr`) was still active and the new domain was not yet operational.

The solution: configure redirects in Otoroshi so that all traffic arriving on `evenements.maif.fr` is forwarded to `maif-evenements.fr` until the migration was complete — including URLs with specific paths, such as department pages (`evenements.maif.fr/42`).

### Route 1 — global redirect

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

All traffic on `evenements.maif.fr` is redirected to `https://maif-evenements.fr`. The `Redirection` plugin short-circuits the request at the `pre_route` phase — it never reaches the backend. That is why the configured backend is `request.otoroshi.io`, Otoroshi's internal dummy backend, used when no real target is needed.

### Route 2 — redirect with path param capture

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

Department pages (`evenements.maif.fr/42`) need to redirect to their equivalent on the old domain (`maif-evenements.fr/dep42`). The syntax `$departement<[0-9]+>` in the frontend captures the numeric path value, and `${req.pathparams.departement}` re-injects it into the target URL.

This route is more specific than the previous one and must be defined first — Otoroshi evaluates routes in order and stops at the first match.

Once the migration was complete, removing the redirects simply meant disabling the two routes in Otoroshi. Nothing more.

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

## iFrame — embedding a third-party app in local development

### The context

A site (let's call it **Calling Site**) embeds a partner application in an iframe (let's call it **Partner iFrame**). In production, everything works: the Calling Site is hosted on a known HTTPS domain, and the Partner iFrame loads without issue.

But in local development, the Calling Site runs on `localhost`. The Partner iFrame sends strict security headers — notably a `Content-Security-Policy` with `frame-ancestors` — that only allow embedding from the production domain. Result: the browser blocks the iframe, making it impossible to develop and test the page locally.

Two problems to solve:

1. **Security headers** — the Partner iFrame sends headers (`Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options`...) that prevent loading in an iframe from `localhost`.
2. **Assets** — the Partner iFrame loads its own CSS files from its domain. In local dev, we want to replace them with a local version to iterate quickly.

### The solution with Otoroshi

An Otoroshi route acts as a proxy between `localhost` and the Partner iFrame. The Calling Site in local dev points to this route rather than directly to the partner URL. It solves both problems:

**Removing security headers** with the `RemoveHeadersOut` plugin:

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

The headers are stripped from responses before they reach the browser. The iframe loads normally.

**Replacing a CSS URL** with the `NgHtmlPatcher` plugin:

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

Otoroshi injects a small script into the HTML that replaces the Partner iFrame's CSS references on the fly with local URLs. The page then loads the CSS currently under development.

### What this avoids

Without Otoroshi, the options would have been: ask the partner team to modify their security headers (out of reach), disable browser protections (unsafe, not reproducible across the team), or build a custom proxy in-house (dev time, maintenance). Otoroshi sorts it out in a few minutes, the time it takes to configure two plugins.

---

## Key takeaways

These four cases illustrate a common pattern: **Otoroshi intercepts and transforms HTTP traffic where application code cannot or should not step in**.

- CORS: the backend doesn't know who consumes its tiles — Otoroshi adds the headers based on the usage context
- Robots: apps don't need to know about indexing policies — Otoroshi applies them uniformly across all environments
- Redirects: the origin server no longer exists or can't be modified — Otoroshi takes over
- iFrame: the partner application is out of reach — Otoroshi adapts its responses for the development context

Next article: securing environments with Basic Auth, OpenID Connect authentication for MAIF members, and caching tile responses.
