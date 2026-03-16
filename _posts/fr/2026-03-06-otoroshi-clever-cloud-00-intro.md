---
layout: single
title: "Last Night Otoroshi Saved My Life — Introduction"
excerpt: "Otoroshi est une API gateway open source bourrée de plugins. Clever Cloud est un PaaS européen. Ensemble, ils forment un duo redoutablement efficace pour gérer des APIs en production. Présentation."
date: 2026-03-06
lang: fr
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, api-gateway, api-management]
---

{% include lang-switcher.html %}

*Last Night a DJ Saved My Life* — Indeep, 1982. Le DJ, c'est Otoroshi : il s'intercale discrètement entre le monde et vos applications, et c'est souvent lui qui règle les problèmes avant qu'ils atteignent le code.

Cette série d'articles explore 9 cas d'usage concrets d'Otoroshi déployé devant des applications hébergées sur Clever Cloud. Pas de théorie abstraite : des situations réelles, des configurations qui fonctionnent.

---

## Otoroshi — un reverse proxy HTTP pour la prod

[Otoroshi](https://maif.github.io/otoroshi/) est un reverse proxy HTTP open source développé par [MAIF](https://www.maif.fr/), une mutuelle française qui l'utilise elle-même en production pour gérer des centaines de services. Ce n'est pas un projet de vitrine — c'est un outil né d'un besoin réel et durci par des années d'usage intensif.

### Ce que fait un reverse proxy

Un reverse proxy est un point d'entrée unique devant vos services. Toutes les requêtes transitent par lui avant d'atteindre vos backends. Ce positionnement lui permet d'intercepter, transformer, enrichir ou bloquer le trafic — sans toucher au code des applications.

Otoroshi va bien au-delà du simple proxying : il intègre des fonctionnalités d'API management (API keys, quotas, plans), d'authentification (Basic Auth, OIDC, JWT...), de sécurité et bien d'autres via son système de plugins. Concrètement, plutôt que de modifier chaque application pour y ajouter de l'authentification, du rate limiting ou du CORS, on le configure une fois dans Otoroshi. Les applications n'ont pas besoin de le savoir.

### Les principales fonctionnalités

**Routing puissant** — Otoroshi route les requêtes selon l'hôte, le path, les headers, les méthodes HTTP. Une même application peut être exposée sous plusieurs routes avec des comportements différents.

**Système de plugins** — C'est le cœur d'Otoroshi. Chaque route est une chaîne de plugins qui transforment les requêtes et réponses : authentification, CORS, cache, headers de sécurité, robots.txt, rate limiting, circuit breaker... Des plugins WASM permettent même d'embarquer de la logique métier en n'importe quel langage compilable en WebAssembly.

**Gestion des API keys** — Création, rotation, quotas, restrictions par IP ou domaine. Les clés peuvent être associées à des plans de consommation.

**Authentification** — Basic Auth, OAuth2/OIDC (Keycloak, Google, GitHub...), JWT, LDAP. La sécurisation d'une application peut se faire entièrement dans Otoroshi, sans modifier l'app.

**Backends "sans backend"** — Otoroshi peut servir directement des fichiers statiques depuis une archive ZIP, un bucket S3, ou des assets HTTP distants. Pratique pour des pages statiques ou de la documentation sans avoir à déployer un serveur dédié.

**Admin UI & REST API** — Une interface d'administration complète et une API REST pour tout piloter, y compris par code ou CI/CD.

---

## Clever Cloud — le PaaS européen

[Clever Cloud](https://www.clever-cloud.com/) est une plateforme cloud française fondée en 2010. C'est un PaaS (Platform as a Service) : vous déployez votre code, la plateforme s'occupe du reste — serveurs, mise à l'échelle, disponibilité.

### Déployer sur Clever Cloud

Le déploiement se fait par `git push` ou via le CLI `clever`. Clever Cloud supporte Java, Node.js, PHP, Python, Scala, Ruby, Go, Rust, et d'autres. Pas de Dockerfile à écrire, pas de Kubernetes à configurer — vous déclarez le type d'application et la plateforme choisit le runtime approprié.

### Les add-ons

L'écosystème Clever Cloud inclut des add-ons managés : PostgreSQL, MySQL, MongoDB, Redis, Pulsar, Keycloak, Elasticsearch... et **Otoroshi**. Chaque add-on est provisionné en quelques clics, rattaché à une application ou à une organisation, et géré par Clever Cloud (mises à jour, sauvegardes, monitoring).

---

## Otoroshi + Clever Cloud — le lien

Clever Cloud propose Otoroshi comme add-on managé. En quelques clics dans la console, vous provisionnez une instance Otoroshi prête à l'emploi, avec son interface d'administration accessible directement. Pas de serveur à installer, pas de Redis à configurer séparément — c'est inclus.

Le cas d'usage naturel : vous avez une ou plusieurs applications sur Clever Cloud, et vous voulez les exposer proprement vers l'extérieur. Otoroshi devient le point d'entrée de votre infrastructure, configuré pour router, sécuriser et enrichir le trafic avant qu'il atteigne vos apps.

C'est exactement ce que cette série illustre.

---

## Le fil rouge — Aux Alentours par MAIF

La plupart des exemples de cette série s'appuient sur un projet réel : [**Aux Alentours par MAIF**](https://auxalentours.maif.fr), une application qui permet de consulter les risques naturels et technologiques à partir d'une adresse, et d'obtenir des conseils et solutions de prévention adaptés.

L'infrastructure se compose de deux applications déployées sur Clever Cloud :

- **Le site web** — `auxalentours.maif.fr`, l'interface utilisateur
- **L'API** — le backend qui expose les données géographiques, les tuiles cartographiques et le rendu de cartes

Otoroshi se positionne en frontal de toute la plateforme : aussi bien devant le site web que devant l'API. C'est cette brique qui permet d'exposer le même backend de quatre façons différentes (article 1), de gérer le CORS pour les intégrations web (article 2), de contrôler l'indexation par les moteurs de recherche (article 2), de sécuriser les environnements hors production (article 3), ou encore d'authentifier les sociétaires MAIF pour certaines fonctionnalités (article 3).

Les articles 4 est indépendant du fil rouge — il couvre des use-cases utiles dans d'autres contextes (backends statiques, documentation Swagger).

---

## La série

**[Article 1 — Une app Clever Cloud, trois expositions différentes]({{ "/2026/03/06/otoroshi-clever-cloud-01-one-app-three-routes/" | relative_url }})**
Une seule API déployée sur Clever Cloud (l'API d'Aux Alentours par MAIF), exposée sous trois routes avec des profils radicalement différents : API sécurisée par API keys, documentation publique et API de tiles géographiques.

**[Article 2 — HTTP au quotidien]({{ "/2026/03/16/otoroshi-clever-cloud-02-http-daily/" | relative_url }})**
Quatre cas qui peuvent être rencontrés : CORS pour les intégrations Leaflet/web, plugin robots.txt pour contrôler l'indexation, redirections HTTP pour une migration de domaine, suppression de headers de sécurité pour une iframe en développement.

**Article 3 — Sécurité & performance**
Sécuriser les environnements hors production sans toucher au code (Basic Auth), authentifier les utilisateurs via OpenID Connect (sociétaires MAIF sur Aux Alentours), et mettre en cache les réponses d'une API de tiles pour soulager le backend.

**Article 4 — Funny Features**
Servir des contenus sans application dédiée (ZIP, S3, assets statiques), et exposer une UI Swagger complète à partir d'un simple fichier `openapi.json`.
