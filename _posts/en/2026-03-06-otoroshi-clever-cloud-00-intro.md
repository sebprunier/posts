---
layout: single
title: "Last Night Otoroshi Saved My Life — Introduction"
excerpt: "Otoroshi is an open source HTTP reverse proxy packed with plugins. Clever Cloud is a European PaaS. Together, they make a remarkably effective duo for managing APIs in production. An introduction."
date: 2026-03-06
lang: en
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, api-gateway, api-management]
---

{% include lang-switcher.html %}

*Last Night a DJ Saved My Life* — Indeep, 1982. Otoroshi is the DJ: it quietly slots in between the outside world and your applications, and it's often the one that fixes problems before they ever reach your code.

This series of articles explores 9 concrete use cases of Otoroshi deployed in front of applications hosted on Clever Cloud. No abstract theory — real situations, working configurations.

---

## Otoroshi — an HTTP reverse proxy built for production

[Otoroshi](https://maif.github.io/otoroshi/) is an open source HTTP reverse proxy developed by [MAIF](https://www.maif.fr/), a French insurance company that uses it in production to manage hundreds of services. This is not a showcase project — it's a tool born from a real need and hardened by years of intensive use.

### What a reverse proxy does

A reverse proxy is a single entry point in front of your services. Every request passes through it before reaching your backends. This position allows it to intercept, transform, enrich, or block traffic — without touching application code.

Otoroshi goes well beyond simple proxying: it includes API management features (API keys, quotas, plans), authentication (Basic Auth, OIDC, JWT...), security, and much more through its plugin system. In practice, rather than modifying each application to add authentication, rate limiting, or CORS, you configure it once in Otoroshi. The applications don't need to know.

### Key features

**Powerful routing** — Otoroshi routes requests based on host, path, headers, and HTTP methods. A single application can be exposed through multiple routes with completely different behaviors.

**Plugin system** — This is the heart of Otoroshi. Each route is a plugin pipeline that transforms requests and responses: authentication, CORS, caching, security headers, robots.txt, rate limiting, circuit breaker... WASM plugins even allow embedding custom logic in any language that compiles to WebAssembly.

**API key management** — Creation, rotation, quotas, IP or domain restrictions. Keys can be associated with consumption plans.

**Authentication** — Basic Auth, OAuth2/OIDC (Keycloak, Google, GitHub...), JWT, LDAP. An application can be fully secured through Otoroshi, without modifying the app itself.

**Backendless** — Otoroshi can serve static files directly from a ZIP archive, an S3 bucket, or remote HTTP assets. Useful for static pages or documentation without needing a dedicated server.

**Admin UI & REST API** — A complete administration interface and a REST API to manage everything, including from code or CI/CD pipelines.

---

## Clever Cloud — the European PaaS

[Clever Cloud](https://www.clever-cloud.com/) is a French cloud platform founded in 2010. It's a PaaS (Platform as a Service): you deploy your code, the platform handles the rest — servers, scaling, availability.

### Deploying on Clever Cloud

Deployment happens via `git push` or the `clever` CLI. Clever Cloud supports Java, Node.js, PHP, Python, Scala, Ruby, Go, Rust, and more. No Dockerfile to write, no Kubernetes to configure — you declare the application type and the platform picks the appropriate runtime.

### Add-ons

The Clever Cloud ecosystem includes managed add-ons: PostgreSQL, MySQL, MongoDB, Redis, Pulsar, Keycloak, Elasticsearch... and **Otoroshi**. Each add-on is provisioned in a few clicks, attached to an application or organization, and managed by Clever Cloud (updates, backups, monitoring).

---

## Otoroshi + Clever Cloud — the connection

Clever Cloud offers Otoroshi as a managed add-on. In a few clicks from the console, you provision a ready-to-use Otoroshi instance with its administration interface directly accessible. No server to install, no Redis to configure separately — it's all included.

The natural use case: you have one or more applications on Clever Cloud, and you want to expose them cleanly to the outside world. Otoroshi becomes the entry point of your infrastructure, configured to route, secure, and enrich traffic before it reaches your apps.

That's exactly what this series illustrates.

---

## The running example — Aux Alentours par MAIF

Most examples in this series are based on a real project: [**Aux Alentours par MAIF**](https://auxalentours.maif.fr), an application that lets users consult natural and technological risks from a given address, and get tailored prevention advice and solutions.

The infrastructure consists of two applications deployed on Clever Cloud:

- **The web frontend** — `auxalentours.maif.fr`, the user interface
- **The API** — the backend that exposes geographic data, map tiles, and map rendering

Otoroshi sits in front of the entire platform: both the web frontend and the API. It's this component that makes it possible to expose the same backend in four different ways (article 1), handle CORS for web integrations (article 2), control search engine indexing (article 2), secure non-production environments (article 3), and authenticate MAIF members for certain features (article 3).

Article 4 is independent from this running example — it covers use cases useful in other contexts (static backends, Swagger documentation).

---

## The series

**[Article 1 — One Clever Cloud app, three different exposures]({{ "/2026/03/06/otoroshi-clever-cloud-01-one-app-three-routes/" | relative_url }})**
A single API deployed on Clever Cloud (the Aux Alentours par MAIF API), exposed through three routes with radically different profiles: API key-secured endpoints, public documentation, and geographic tile API.

**Article 2 — Everyday HTTP**
Four situations that can come up on any project: CORS for Leaflet/web integrations, the robots.txt plugin to control indexing, HTTP redirects for a domain migration, and removing security headers to embed an iframe in development.

**Article 3 — Security & performance**
Securing non-production environments without touching the code (Basic Auth), authenticating users via OpenID Connect (MAIF members on Aux Alentours par MAIF), and caching tile API responses to offload the backend.

**Article 4 — Funny Features**
Serving content without a dedicated application (ZIP, S3, static assets), and exposing a full Swagger UI from a simple `openapi.json` file.
