---
layout: single
title: "Last Night Otoroshi Saved My Life — #1 : une app, trois expositions"
excerpt: "L'API d'Aux Alentours par MAIF déployée sur Clever Cloud, exposée sous trois routes Otoroshi radicalement différentes : API sécurisée par API keys, documentation publique et API de tiles."
date: 2026-03-06
lang: fr
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, api-gateway, api-management, api-keys, routing]
---

{% include lang-switcher.html %}

Premier article de la [série Otoroshi + Clever Cloud]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). On commence par le cas fondateur : une seule application backend, exposée de trois façons différentes — sans toucher au code.

---

## Aux Alentours par MAIF — le contexte

[Aux Alentours par MAIF](https://auxalentours.maif.fr) est une application qui permet de consulter les risques naturels et technologiques à partir d'une adresse, et d'obtenir des conseils et solutions de prévention adaptés. L'infrastructure repose sur deux applications déployées sur Clever Cloud :

- **Le site web** — `auxalentours.maif.fr`, l'interface utilisateur
- **L'API** — le backend qui alimente le site et expose plusieurs types de données

Otoroshi est déployé sur un scaler Clever Cloud de type JVM — une installation custom, car à l'époque où la plateforme a été mise en place, l'add-on Otoroshi n'existait pas encore. Il se positionne en frontal de toute la plateforme : aussi bien devant le site web que devant l'API.

Les applications sont déployées sur des domaines internes (`*.innovation.maif`) et ne sont jamais accessibles directement. Pour garantir cela, Otoroshi utilise l'**[exchange protocol](https://maif.github.io/otoroshi/manual/topics/otoroshi-protocol.html)** : un mécanisme qui permet aux backends de vérifier que les requêtes qu'ils reçoivent transitent bien par Otoroshi, et de les rejeter dans le cas contraire. À noter que Clever Cloud propose désormais [Request Flow](https://www.clever.cloud/developers/doc/develop/request-flow/), une fonctionnalité native qui remplit le même rôle sans avoir à implémenter le challenge côté application.

<pre class="mermaid">
graph LR
    Users["👤 Utilisateurs"]
    Partners["🤝 Partenaires"]
    Oto["⚙️ Otoroshi<br/>Clever Cloud JVM scaler"]
    Site["🌐 Site web<br/>Clever Cloud"]
    API["🖥️ API<br/>Clever Cloud"]

    Users --> Oto
    Partners -->|"API key dédiée"| Oto
    Oto -->|"exchange protocol"| Site
    Oto -->|"exchange protocol"| API
</pre>

Cet article se concentre sur l'API. Elle expose trois profils très différents : une API REST sécurisée, une documentation publique, et un service de tuiles cartographiques. Trois raisons de ne pas les exposer de la même façon.

---

## Trois routes, une seule application

Dans Otoroshi, chaque **route** définit un frontend (domaine + path entrant) et un backend (cible). Il n'est pas nécessaire de créer un objet backend séparé : la cible est déclarée directement dans la route. Les trois routes pointent toutes vers la même application Clever Cloud (`app-xxxxxxxx.innovation.maif`), mais avec des configurations différentes.

<pre class="mermaid">
graph TD
    R1["Route API<br/>api.auxalentours.innovation.maif<br/>🔑 API key obligatoire"]
    R2["Route doc<br/>api.auxalentours.innovation.maif/doc<br/>🌍 Accès public"]
    R3["Route tiles<br/>tiles.auxalentours.innovation.maif<br/>🌍 Accès public"]
    Backend["🖥️ app-xxxxxxxx.innovation.maif<br/>API Aux Alentours par MAIF"]

    R1 --> Backend
    R2 --> Backend
    R3 --> Backend
</pre>

### Plugins communs à toutes les routes

Quatre plugins sont présents sur chacune des routes :

- **`OverrideHost`** — remplace l'header `Host` de la requête sortante par le hostname du backend. Indispensable pour que l'application reçoive le bon host.
- **`ForceHttpsTraffic`** — redirige automatiquement les requêtes HTTP vers HTTPS.
- **`DisableHttp10`** — rejette les requêtes HTTP/1.0, obsolète et inadapté au volume de trafic moderne.
- **`Robots`** — bloque l'indexation par les moteurs de recherche (on y reviendra en détail dans l'article 2).

---

## Route 1 — API sécurisée par API keys

```json
"frontend": {
  "domains": ["api.auxalentours.innovation.maif"],
  "strip_path": true
},
"backend": {
  "targets": [{ "hostname": "app-xxxxxxxx.innovation.maif" }],
  "root": "/api/"
}
```

Les requêtes arrivent sur `api.auxalentours.innovation.maif`. Avec `strip_path: true` et `root: /api/`, Otoroshi relaie vers `/api/<path>` côté backend. La route ne fait pas de filtrage de path : elle couvre l'ensemble du domaine.

Le plugin **`ApikeyCalls`** est ajouté avec `mandatory: true` — toute requête sans clé valide reçoit un `401`.

### Comment fonctionnent les API keys dans Otoroshi

Une API key est composée d'un **client ID** et d'un **client secret**. Otoroshi supporte plusieurs façons de les transmettre ; on utilise ici le **bearer token**, via l'header HTTP standard :

```http
Authorization: Bearer otoapk_<api-key-id>_<hash>
```

Le token est un token dédié généré par Otoroshi, utilisé directement dans les requêtes. C'est l'extracteur `oto_bearer` du plugin `ApikeyCalls` qui le valide côté Otoroshi.

On peut configurer des quotas (requêtes par jour, par mois) et restreindre par IP ou domaine. Le site Aux Alentours par MAIF dispose de sa propre clé pour consommer l'API — tout comme les partenaires qui ont chacun leur clé dédiée avec des limites ajustées. Si une clé est compromise, on la révoque sans impacter les autres clients.

---

## Route 2 — Documentation publique

```json
"frontend": {
  "domains": ["api.auxalentours.innovation.maif/doc"],
  "strip_path": false
},
"backend": {
  "targets": [{ "hostname": "app-xxxxxxxx.innovation.maif" }],
  "root": "/"
}
```

La documentation est exposée sur le même domaine que l'API (`api.auxalentours.innovation.maif`), mais sur le path `/doc`. Avec `strip_path: false`, le path est conservé tel quel jusqu'au backend.

Aucun plugin d'authentification n'est ajouté : la documentation est publique. Otoroshi route les requêtes vers `/doc` sans condition, même si la route API principale exige une clé.

---

## Route 3 — API de tiles

```json
"frontend": {
  "domains": ["tiles.auxalentours.innovation.maif"],
  "strip_path": true
},
"backend": {
  "targets": [{ "hostname": "app-xxxxxxxx.innovation.maif" }],
  "root": "/tiles/"
}
```

Les tuiles cartographiques sont exposées sur un sous-domaine dédié (`tiles.auxalentours.innovation.maif`). Avec `strip_path: true` et `root: /tiles/`, Otoroshi relaie vers `/tiles/<z>/<x>/<y>.png` côté backend.

Les tiles sont publiques — elles sont consommées directement par le navigateur, et une seule carte peut déclencher plusieurs dizaines de requêtes en parallèle. Exposer les tiles sur un sous-domaine séparé apporte une clarté architecturale immédiate et facilite les politiques de trafic différenciées — notamment la mise en cache, qu'on abordera dans l'article 3.

---

## Vue d'ensemble

| Route | Frontend | Auth | Spécificités |
|-------|----------|------|--------------|
| API | `api.auxalentours.innovation.maif` | API key | `ApikeyCalls` mandatory |
| Documentation | `api.auxalentours.innovation.maif/doc` | Aucune | Accès public |
| Tiles | `tiles.auxalentours.innovation.maif` | Aucune | — |

---

## Ce qu'on retient

Otoroshi permet de **découpler la politique d'exposition d'une application de son implémentation**. Une même application peut être accessible de façons radicalement différentes selon le contexte, sans modification du code ni déploiement supplémentaire.

La granularité se joue à deux niveaux : le **domaine** (sous-domaine dédié pour les tiles) et le **path** (même domaine, path différent pour la doc). Dans les deux cas, chaque route a son propre pipeline de plugins, totalement indépendant.

Dans le prochain article, on reste sur Aux Alentours par MAIF pour deux cas HTTP concrets (CORS et robots.txt), avant d'aborder des use-cases indépendants (redirections HTTP, iframe).
