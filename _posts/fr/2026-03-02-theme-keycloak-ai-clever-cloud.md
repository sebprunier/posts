---
layout: single
title: "Créer un thème Keycloak avec l'IA, déployé sur Clever Cloud"
date: 2026-02-27
lang: fr
categories: [keycloak, ia]
tags: [keycloak, claude-code, freemarker, clever-cloud, spring-boot]
---

{% include lang-switcher.html %}

> Dans cet article, je raconte comment j’ai créé un thème Keycloak personnalisé pour le backoffice de DICRIM numérique, en utilisant Claude Code comme assistant. De la première question aux derniers bugs en production, je partage tout — y compris les erreurs.

---

## Le contexte

Le [DICRIM numérique](https://dicrim.territoires-prevention.fr/) est une application Spring Boot que je déploie sur **Clever Cloud**. Elle permet aux collectivités de créer et diffuser leur Document d'Information Communal sur les Risques Majeurs.  
L'authentification des utilisateurs du backoffice est gérée par l'addon **Managed Keycloak** de Clever Cloud (version 26.1.2).

Par défaut, Keycloak affiche ses propres pages de connexion — fonctionnelles, mais très génériques. Mon objectif était simple : les remplacer par des pages aux couleurs de DICRIM numérique afin d’obtenir une expérience cohérente.

![Page de login Keycloak par défaut]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-login-page.png" | relative_url }})
*Page de connexion Keycloak par défaut avant personnalisation.*

---

## "Il te faut quoi comme infos pour commencer ?"

Tout a commencé par cette question que j’ai posée à Claude Code :

> *"j'ai besoin de faire un theme keycloak pour ce projet (on utilise keycloak pour l'auth des utilisateurs). tu peux m'aider à faire ça ? il te faut quoi comme infos pour commencer ?"*

Plutôt que de générer directement du code, l’assistant a commencé par explorer le projet : lecture des fichiers SCSS, analyse de la palette Bootstrap personnalisée, localisation des polices… puis seulement ensuite les bonnes questions :

- **Version de Keycloak ?** → 26.1.2  
- **Approche ?** → Freemarker natif (pas de Keycloakify/React)  
- **Pages à personnaliser ?** → Login, Account, Email  
- **Déploiement ?** → JAR provider dans un dossier `keycloak/` à la racine du projet  

Ces choix ont structuré toute la suite.

### Pourquoi Freemarker natif plutôt que Keycloakify ?

[Keycloakify](https://www.keycloakify.dev/) est un outil moderne permettant de créer des thèmes Keycloak en React/TypeScript. C’est puissant, mais cela introduit un sous-projet JS supplémentaire et un outillage front plus lourd.

Dans mon cas, l’approche **Freemarker native** était plus adaptée :

- aucune dépendance JS supplémentaire
- templates `.ftl` simples à maintenir
- intégration naturelle dans le stack Maven/Java existant

---

## La palette de couleurs : extraite du code, pas inventée

Plutôt que de me demander les couleurs, Claude Code les a directement extraites des fichiers SCSS du projet :

```scss
/* src/main/scss/bootstrap/_variables.scss */
$primary:   #133478;  /* bleu marine */
$secondary: #C34A09;  /* orange */
$success:   #009686;  /* vert */
$info:      #E2EEFA;  /* bleu clair */
```

Même logique pour les polices : les fichiers `Poligon-*.otf` ont été trouvés automatiquement dans `src/main/fonts/` puis intégrés dans le JAR du thème via le `maven-resources-plugin`.

---

## La structure du projet

Le thème est un **projet Maven standalone** que j’ai placé dans le dossier `keycloak/` à la racine du dépôt, indépendant du projet Spring Boot principal.

```
keycloak/
├── mvnw / mvnw.cmd
├── pom.xml
└── src/main/resources/
    ├── META-INF/
    │   └── keycloak-themes.json
    └── theme/dicrim/
        ├── login/
        ├── account/
        └── email/
```

### "Je n'ai pas Maven sur ma machine"

Je n’avais pas Maven installé localement.  
Solution simple : copier le Maven Wrapper (`mvnw`, `.mvn/`) depuis le projet Spring Boot existant.

```bash
cd keycloak && ./mvnw package
# → target/dicrim-keycloak-theme.jar
```

---

## Le design de la page de login

### Layout split-screen

J’ai repris les codes visuels du backoffice avec un layout **split-screen** :

- **Panneau gauche** : fond bleu marine (`#133478`), identité applicative
- **Panneau droit** : formulaire clair et minimaliste

![Page de login DICRIM après personnalisation]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-login-page-with-theme.png" | relative_url }})
*Page de connexion après application du thème DICRIM.*

---

## Les bugs — parce qu'il y en a toujours

J’ai rencontré plusieurs erreurs dès les premiers déploiements, principalement liées aux subtilités Freemarker et aux changements internes de Keycloak 26.

![Page de confirmation de logout avec le style DICRIM]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-logout-with-theme.png" | relative_url }})
*Page de confirmation de déconnexion après personnalisation du thème.*

---

## Le thème account : juste du CSS

La console `/account` de Keycloak 26 est une SPA React basée sur PatternFly 5.  
Impossible de modifier le HTML, mais j’ai pu injecter une feuille CSS et surcharger les variables PatternFly.

![Account avec les couleurs DICRIM]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-account-with-theme.png" | relative_url }})
*Console utilisateur Keycloak recolorisée avec la charte graphique DICRIM.*

---

## Le thème email

J’ai créé trois templates HTML transactionnels :

- `email-verification.ftl`
- `password-reset.ftl`
- `executeActions.ftl`

Un fichier `messages_fr.properties` gère la traduction française.

---

## Déploiement sur Clever Cloud

L’addon Managed Keycloak de Clever Cloud stocke les thèmes et plugins dans un **FSBucket** associé à l’instance.

![FSBucket avec le JAR dans providers]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-fsbucket.png" | relative_url }})
*Upload du thème packagé (JAR provider) dans le FSBucket Clever Cloud.*

![Les 3 thèmes dicrim sélectionnés]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-keycloak-config.png" | relative_url }})
*Activation des thèmes Login, Account et Email dans l’administration Keycloak.*

---

## Ce que je retiens

**Sur Keycloak :**
- le nom du thème parent change selon les versions majeures
- Freemarker reste puissant mais exigeant
- toute variable optionnelle doit être protégée (`!`)
- surcharger `template.ftl` implique de surcharger les templates dépendants

**Sur l’approche IA-assistée :**
- l’IA commence par comprendre le projet existant
- les erreurs font partie du processus
- les logs Keycloak suffisent souvent à diagnostiquer rapidement

---

