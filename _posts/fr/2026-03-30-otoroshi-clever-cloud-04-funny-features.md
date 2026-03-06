---
layout: single
title: "Last Night Otoroshi Saved My Life — #4 : Funny Features"
excerpt: "Servir des contenus statiques sans application dédiée (ZIP, S3, assets HTTP), et exposer une UI Swagger complète depuis un simple fichier openapi.json — le tout sans déployer de serveur supplémentaire."
date: 2026-03-30
lang: fr
categories: [otoroshi, clever-cloud]
tags: [otoroshi, clever-cloud, backendless, swagger, static, s3]
---

{% include lang-switcher.html %}

Quatrième et dernier article de la [série Otoroshi + Clever Cloud]({{ "/2026/03/06/otoroshi-clever-cloud-00-intro/" | relative_url }}). On sort du fil rouge Aux Alentours par MAIF pour explorer deux *funny features* d'Otoroshi qui permettent de servir des contenus sans déployer d'application backend dédiée.

---

## Backends "sans backend"

Un backend Otoroshi n'est pas forcément une application qui tourne quelque part. Otoroshi peut lui-même jouer le rôle de serveur de contenu pour plusieurs types de sources. Ces modes sont particulièrement utiles pour des besoins ponctuels : page de maintenance, documentation statique, assets partagés, prototypes — sans mobiliser une application Clever Cloud complète.

### ZIP backend — servir une archive statique

Le backend de type ZIP permet de servir le contenu d'une archive ZIP hébergée à une URL accessible. Otoroshi télécharge l'archive, l'extrait en mémoire, et sert les fichiers qu'elle contient comme un serveur de fichiers statiques.

```json
{
  "targets": [{
    "hostname": "zip-backend.otoroshi.io",
    "port": 443,
    "tls": true
  }],
  "root": "/local/zip/https://storage.example.com/mon-site.zip"
}
```

Cas d'usage typique : déployer rapidement une page de maintenance ou une landing page statique. On génère le site avec un outil comme Hugo ou Vite, on upload le ZIP sur un bucket S3 (ou un FSBucket Clever Cloud), et Otoroshi le sert directement. Zéro runtime à gérer.

### S3 backend — servir depuis un bucket

Le backend S3 connecte Otoroshi directement à un bucket compatible S3. Les fichiers du bucket sont servis comme des assets statiques.

```json
{
  "targets": [{
    "hostname": "s3-backend.otoroshi.io",
    "port": 443,
    "tls": true
  }],
  "root": "/s3/mon-bucket/mon-prefix"
}
```

La configuration inclut les credentials S3 (access key, secret key, région, endpoint). Clever Cloud propose des **Cellar** add-ons, des buckets S3-compatibles provisionnables en quelques clics. Un bucket Cellar + une route Otoroshi = un hébergement statique opérationnel sans aucun serveur.

<pre class="mermaid">
graph LR
    Browser["🌐 Navigateur"]
    Oto["⚙️ Otoroshi"]
    Cellar["🗄️ Cellar S3<br/>Clever Cloud add-on"]

    Browser -->|"GET /assets/..."| Oto
    Oto -->|"lecture"| Cellar
    Cellar -->|"fichier"| Oto
    Oto -->|"200 OK"| Browser
</pre>

---

## Swagger UI — exposer une documentation API sans serveur

### Le besoin

Une API dispose d'un fichier `openapi.json` décrivant ses endpoints, modèles et paramètres. On veut exposer une interface Swagger UI navigable pour les développeurs qui consomment l'API — sans déployer un serveur Node.js, sans intégrer Swagger UI dans l'application, et sans maintenir une infrastructure dédiée.

### La solution avec Otoroshi

Otoroshi inclut un plugin **`SwaggerUi`** qui sert une interface Swagger UI complète à partir d'une URL vers un fichier OpenAPI. Il n'y a pas de serveur à provisionner : Otoroshi embarque le rendu Swagger directement.

```json
{
  "plugin": "cp:otoroshi.next.plugins.SwaggerUi",
  "config": {
    "openapi_url": "https://api.mon-projet.example.com/doc/openapi.json"
  }
}
```

On crée une route dédiée :

```json
"frontend": {
  "domains": ["api.mon-projet.example.com/swagger"]
}
```

En accédant à `https://api.mon-projet.example.com/swagger`, le développeur obtient une interface Swagger UI complète, interactive, pointant vers le fichier OpenAPI de l'API.

<pre class="mermaid">
graph LR
    Dev["👩‍💻 Développeur"]
    Oto["⚙️ Otoroshi<br/>SwaggerUi plugin"]
    Json["📄 openapi.json"]

    Dev -->|"GET /swagger"| Oto
    Oto -->|"Swagger UI + fetch"| Dev
    Dev -->|"fetch openapi.json"| Json
</pre>

### Ce que ça remplace

Sans ce plugin, les options classiques seraient :
- Intégrer Swagger UI dans l'application backend (dépendance, configuration, maintenance)
- Déployer un conteneur Swagger UI dédié (une app Clever Cloud supplémentaire, un déploiement à maintenir)
- Utiliser Swagger Editor en ligne (pas de customisation, pas de domaine propre)

Avec le plugin `SwaggerUi`, la documentation est disponible en quelques secondes après la configuration de la route — et le fichier `openapi.json` reste la seule source de vérité à maintenir.

---

## Ce qu'on retient

Ces deux fonctionnalités illustrent un aspect souvent sous-estimé d'Otoroshi : sa capacité à **remplacer des applications entières** pour des besoins de contenu statique ou de présentation.

Sur Clever Cloud, chaque application déployée a un coût en ressources et en maintenance. Pour des besoins simples — documentation, assets, pages statiques — les backends Otoroshi permettent d'éviter un déploiement complet tout en bénéficiant du même routing, des mêmes plugins (auth, cache, CORS) et des mêmes domaines que les routes applicatives.

---

## Bilan de la série

Neuf use cases, une instance Otoroshi, des dizaines d'applications servies. Ce qui ressort de cette série :

**Otoroshi comme couche de contrôle** — routing, sécurité, transformation HTTP, cache, authentification : tout se configure en un seul endroit, sans modifier le code des applications.

**Clever Cloud comme socle** — les applications et les add-ons (PostgreSQL/PostGIS, Cellar S3, Otoroshi lui-même) sont provisionnés et opérés par la plateforme. L'équipe se concentre sur la valeur métier, pas sur l'infrastructure.

**Le duo en production** — les use cases de cette série ne sont pas des exemples théoriques. Ils sont issus de projets réels, en production, avec de vraies contraintes de disponibilité et de performance.
