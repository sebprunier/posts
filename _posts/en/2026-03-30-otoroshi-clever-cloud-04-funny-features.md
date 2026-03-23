---
layout: single
title: "Last Night Otoroshi Saved My Life — #4: Funny Features"
excerpt: "Serving static content without a dedicated application (ZIP, S3, HTTP assets), and exposing a full Swagger UI from a simple openapi.json file — all without deploying an extra server."
date: 2026-03-30
lang: en
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, backendless, swagger, static, s3]
cover: /assets/images/2026-03-30-otoroshi-clever-cloud-04-funny-features/cover.png
---

{% include lang-switcher.html %}

Fourth and final article in the [Otoroshi + Clever Cloud series]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). We step away from the Aux Alentours par MAIF running example to explore two *funny features* of Otoroshi that allow serving content without deploying a dedicated backend application.

---

## Backends without a backend

An Otoroshi backend is not necessarily a running application. Otoroshi can itself act as a content server for several types of sources. These modes are particularly useful for one-off needs: maintenance pages, static documentation, shared assets, prototypes — without spinning up a full Clever Cloud application.

### ZIP backend — serving a static archive

The ZIP backend serves the contents of a ZIP archive hosted at an accessible URL. Otoroshi downloads the archive, extracts it in memory, and serves its files like a static file server.

```json
{
  "targets": [{
    "hostname": "zip-backend.otoroshi.io",
    "port": 443,
    "tls": true
  }],
  "root": "/local/zip/https://storage.example.com/my-site.zip"
}
```

Typical use case: quickly deploying a maintenance page or a static landing page. Generate the site with a tool like Hugo or Vite, upload the ZIP to an S3 bucket (or a Clever Cloud FSBucket), and Otoroshi serves it directly. Zero runtime to manage.

### S3 backend — serving from a bucket

The S3 backend connects Otoroshi directly to an S3-compatible bucket. Files in the bucket are served as static assets.

```json
{
  "targets": [{
    "hostname": "s3-backend.otoroshi.io",
    "port": 443,
    "tls": true
  }],
  "root": "/s3/my-bucket/my-prefix"
}
```

The configuration includes S3 credentials (access key, secret key, region, endpoint). Clever Cloud offers **Cellar** add-ons: S3-compatible buckets provisionable in a few clicks. One Cellar bucket + one Otoroshi route = a working static hosting setup with no server at all.

<pre class="mermaid">
graph LR
    Browser["🌐 Browser"]
    Oto["⚙️ Otoroshi"]
    Cellar["🗄️ Cellar S3<br/>Clever Cloud add-on"]

    Browser -->|"GET /assets/..."| Oto
    Oto -->|"read"| Cellar
    Cellar -->|"file"| Oto
    Oto -->|"200 OK"| Browser
</pre>

---

## Swagger UI — exposing API documentation without a server

### The need

An API has an `openapi.json` file describing its endpoints, models, and parameters. The goal is to expose a browsable Swagger UI for the developers consuming the API — without deploying a Node.js server, without embedding Swagger UI in the application, and without maintaining dedicated infrastructure.

### The solution with Otoroshi

Otoroshi includes a **`SwaggerUi`** plugin that serves a full Swagger UI from a URL pointing to an OpenAPI file. No server to provision: Otoroshi handles the Swagger rendering directly.

```json
{
  "plugin": "cp:otoroshi.next.plugins.SwaggerUi",
  "config": {
    "openapi_url": "https://api.my-project.example.com/doc/openapi.json"
  }
}
```

A dedicated route is created:

```json
"frontend": {
  "domains": ["api.my-project.example.com/swagger"]
}
```

By visiting `https://api.my-project.example.com/swagger`, developers get a full, interactive Swagger UI pointing to the API's OpenAPI file.

<pre class="mermaid">
graph LR
    Dev["👩‍💻 Developer"]
    Oto["⚙️ Otoroshi<br/>SwaggerUi plugin"]
    Json["📄 openapi.json"]

    Dev -->|"GET /swagger"| Oto
    Oto -->|"Swagger UI + fetch"| Dev
    Dev -->|"fetch openapi.json"| Json
</pre>

### What this replaces

Without this plugin, the classic options would be:
- Embedding Swagger UI in the backend application (dependency, configuration, maintenance)
- Deploying a dedicated Swagger UI container (an extra Clever Cloud app, a deployment to maintain)
- Using Swagger Editor online (no customisation, no custom domain)

With the `SwaggerUi` plugin, documentation is available seconds after configuring the route — and the `openapi.json` file remains the only source of truth to maintain.

---

## Key takeaways

These two features illustrate an often underestimated aspect of Otoroshi: its ability to **replace entire applications** for static content or presentation needs.

On Clever Cloud, every deployed application has a cost in resources and maintenance. For simple needs — documentation, assets, static pages — Otoroshi backends avoid a full deployment while benefiting from the same routing, the same plugins (auth, cache, CORS), and the same domains as application routes.

---

## Series wrap-up

Nine use cases, one Otoroshi instance, dozens of applications served. Key takeaways from this series:

**Otoroshi as a control layer** — routing, security, HTTP transformation, caching, authentication: everything is configured in one place, without modifying application code.

**Clever Cloud as the foundation** — applications and add-ons (PostgreSQL/PostGIS, Cellar S3, Otoroshi itself) are provisioned and operated by the platform. The team focuses on business value, not infrastructure.

**The duo in production** — the use cases in this series are not theoretical examples. They come from real projects, in production, with real availability and performance constraints.
