---
layout: post
title: "Créer un thème Keycloak avec l'IA — retour d'expérience"
date: 2026-02-27
categories: [keycloak, ia]
tags: [keycloak, claude-code, freemarker, clever-cloud, spring-boot]
---

> Cet article raconte comment nous avons créé un thème Keycloak personnalisé pour le backoffice de DICRIM numérique, en utilisant Claude Code comme assistant. De la première question aux derniers bugs en prod, on vous raconte tout — y compris les erreurs.

---

## Le contexte

[DICRIM numérique](https://www.clever.cloud/) est une application Spring Boot déployée sur **Clever Cloud** qui permet aux collectivités de créer et diffuser leur Document d'Information Communal sur les Risques Majeurs. L'authentification des utilisateurs du backoffice est gérée par l'addon **Managed Keycloak** de Clever Cloud (version 26.1.2).

Par défaut, Keycloak affiche ses propres pages de connexion — fonctionnelles, mais génériques. L'objectif : les remplacer par des pages aux couleurs de DICRIM numérique pour une expérience cohérente.

> 📸 **Capture d'écran** : page de login Keycloak par défaut (avant)

> 📸 **Capture d'écran** : page de login DICRIM après personnalisation (après)

---

## "Il te faut quoi comme infos pour commencer ?"

Tout a commencé par cette question posée à Claude Code :

> *"j'ai besoin de faire un theme keycloak pour ce projet (on utilise keycloak pour l'auth des utilisateurs). tu peux m'aider à faire ça ? il te faut quoi comme infos pour commencer ?"*

Plutôt que de se lancer directement, l'assistant a commencé par explorer le projet — lisant les fichiers SCSS, relevant la palette de couleurs Bootstrap personnalisée, localisant les polices — avant de poser les questions vraiment utiles :

- **Version de Keycloak ?** → 26.1.2
- **Approche ?** → Freemarker natif (pas de Keycloakify/React)
- **Pages à personnaliser ?** → Login, Account, Email
- **Déploiement ?** → JAR provider, dans un dossier `keycloak/` à la racine du projet

Ces choix ont orienté toute l'architecture de la solution.

### Pourquoi Freemarker natif plutôt que Keycloakify ?

[Keycloakify](https://www.keycloakify.dev/) est un outil moderne qui permet de créer des thèmes Keycloak en React/TypeScript. C'est puissant, mais ça implique un sous-projet JS séparé, un outillage front supplémentaire et une courbe d'apprentissage.

Pour ce projet, l'approche **Freemarker native** est plus adaptée :
- Pas de dépendance JS à gérer
- Templates `.ftl` simples à maintenir
- Compatible avec le stack Maven/Java existant

---

## La palette de couleurs : extraite du code, pas inventée

Plutôt que de demander les couleurs, Claude Code les a directement extraites des fichiers SCSS du projet :

```scss
/* src/main/scss/bootstrap/_variables.scss */
$primary:   #133478;  /* bleu marine */
$secondary: #C34A09;  /* orange */
$success:   #009686;  /* vert */
$info:      #E2EEFA;  /* bleu clair */
```

Même chose pour les polices : les fichiers `Poligon-*.otf` ont été localisés dans `src/main/fonts/` et intégrés directement dans le JAR du thème via le `maven-resources-plugin`.

---

## La structure du projet

Le thème est un **projet Maven standalone** dans le dossier `keycloak/` à la racine du dépôt, indépendant du projet Spring Boot principal.

```
keycloak/
├── mvnw / mvnw.cmd                     ← Maven Wrapper
├── pom.xml                             ← Build → JAR provider KC
└── src/main/resources/
    ├── META-INF/
    │   └── keycloak-themes.json        ← Déclaration du thème
    └── theme/dicrim/
        ├── login/                      ← Pages d'authentification (FTL)
        │   ├── template.ftl            ← Layout maître
        │   ├── login.ftl
        │   ├── logout-confirm.ftl
        │   ├── resources/css/login.css
        │   └── ...
        ├── account/                    ← Console compte utilisateur
        │   ├── theme.properties
        │   └── resources/css/account.css
        └── email/                      ← Emails transactionnels
            ├── messages/messages_fr.properties
            └── html/*.ftl
```

### "Je n'ai pas Maven sur ma machine"

Un détail pratique : le projet n'avait pas Maven installé localement. Solution immédiate — copier le Maven Wrapper (`mvnw`, `mvnw.cmd`, `.mvn/`) depuis le projet Spring Boot principal, qui en dispose déjà. Même version (Maven 3.9.9), même comportement : le wrapper le télécharge automatiquement au premier lancement.

```bash
cd keycloak && ./mvnw package
# → target/dicrim-keycloak-theme.jar
```

---

## Le design de la page de login

### Layout split-screen

La page de connexion reprend les codes visuels du backoffice avec un layout **split-screen** :

- **Panneau gauche** : fond bleu marine (`#133478`), nom de l'application, icône
- **Panneau droit** : formulaire sur fond clair, champs stylisés, bouton orange

> 📸 **Capture d'écran** : la page de login DICRIM en plein écran (desktop)

> 📸 **Capture d'écran** : la page de login DICRIM sur mobile (panneau gauche masqué)

### `template.ftl` : le layout maître

En Freemarker, `template.ftl` joue le rôle de layout maître. Il définit une macro `registrationLayout` que tous les templates de pages importent :

```
template.ftl          ← HTML commun, CSS, split-screen, alertes
    ↑ importé par
├── login.ftl                 connexion
├── login-reset-password.ftl  mot de passe oublié
├── logout-confirm.ftl        confirmation de déconnexion
├── login-otp.ftl             code OTP
├── info.ftl / error.ftl      pages génériques
└── ...
```

Chaque template appelle la macro en passant son titre et son contenu :

```freemarker
<#import "template.ftl" as layout>
<@layout.registrationLayout header=msg("loginAccountTitle")>
    <form action="${url.loginAction}" method="post">
        <!-- champs du formulaire -->
    </form>
</@layout.registrationLayout>
```

---

## Les bugs — parce qu'il y en a toujours

### Bug #1 — `; section` : une syntaxe Freemarker non supportée

Premier déploiement du JAR, premier 500. Le log est sans appel :

```
Caused by: freemarker.core.ParseException: Syntax error in template "template.ftl" in line 1, column 106:
Encountered ";", but was expecting one of these patterns: ".", "..", "?", ...
```

**Cause** : le thème parent Keycloak utilise un pattern Freemarker avancé — les *body parameters* de macro — pour différencier les sections `header` et `form` dans un même template :

```freemarker
{{!-- Thème parent keycloak --}}
<#macro registrationLayout ...; section>
    <#nested "header">   {{!-- appelle le bloc "header" du template appelant --}}
    <#nested "form">     {{!-- appelle le bloc "form" du template appelant --}}
</#macro>
```

La syntaxe `; section` dans la *définition* d'une macro n'est pas supportée par la version de Freemarker embarquée dans KC 26.

**Fix** : on change d'approche. Le titre (`header`) devient un simple paramètre string de la macro, et `<#nested>` gère le contenu unique :

```freemarker
{{!-- Notre template.ftl --}}
<#macro registrationLayout header="" ...>
    <h2>${header}</h2>
    <#nested>
</#macro>

{{!-- Nos pages --}}
<@layout.registrationLayout header=msg("loginAccountTitle")>
    <form>...</form>
</@layout.registrationLayout>
```

**Piège associé** : les templates du thème *parent* non surchargés (ex: `logout-confirm.ftl`) continuent d'utiliser l'ancien pattern `; section`. Ils chargent notre `template.ftl`, mais comme notre macro ne fournit pas de valeur à `section`, la variable est `null` et le template plante. **Chaque template parent qui utilise ce pattern doit être surchargé dans notre thème.**

---

### Bug #2 — `locale` : variable absente selon la config du realm

Deuxième déploiement, deuxième 500 — différent cette fois :

```
Caused by: freemarker.core.InvalidReferenceException:
==> locale  [in template "template.ftl" at line 3, column 15]
- Failed at: ${locale.currentLanguageTag}
```

**Cause** : quand l'internationalisation est désactivée sur le realm Keycloak, la variable `locale` n'est pas injectée dans le contexte Freemarker. L'accès direct à `${locale.currentLanguageTag}` provoque une `InvalidReferenceException`.

**Fix** : l'opérateur `!` de Freemarker permet de définir une valeur par défaut en couvrant toute l'expression avec des parenthèses :

```freemarker
{{!-- ❌ Plante si locale est absent --}}
<html lang="${locale.currentLanguageTag}">

{{!-- ✅ Valeur par défaut 'fr' si locale est absent --}}
<html lang="${(locale.currentLanguageTag)!'fr'}">
```

> La subtilité : `${locale.currentLanguageTag!'fr'}` ne couvre que la *dernière étape* de l'expression. Si `locale` lui-même est absent, ça plante quand même. Il faut entourer l'expression entière de parenthèses : `${(locale.currentLanguageTag)!'fr'}`.

---

### Bug #3 — Le thème account invisible dans l'admin KC

Le thème `dicrim` n'apparaissait pas dans la liste des thèmes disponibles pour le **Account theme** dans `Realm settings → Themes`.

> 📸 **Capture d'écran** : page Realm settings → Themes avec le thème dicrim visible uniquement pour Login et Email, absent pour Account

**Cause** : la mauvaise valeur de `parent` dans `account/theme.properties`. En KC 26, le thème account natif s'appelle `keycloak.v3`. Si le parent déclaré n'existe pas, KC ignore silencieusement le thème dans la liste.

| Version KC | Thème account par défaut |
|---|---|
| ≤ 18 | `keycloak` |
| 19 – 25 | `keycloak.v2` |
| **26+** | **`keycloak.v3`** |

**Règle pratique** : utiliser comme `parent` exactement le nom qui apparaît dans la liste de l'admin KC pour le type de thème concerné — pas de suppositions.

```properties
# account/theme.properties
parent=keycloak.v3
styles=css/account.css
```

---

### Bug #4 — Le logout plante

Login fonctionnel, thème account activé — mais le logout renvoie un 500 :

```
Failed to process template logout-confirm.ftl
Caused by: freemarker.core.InvalidReferenceException:
==> section  [in template "logout-confirm.ftl" at line 3, column 10]
~ Reached through: @layout.registrationLayout; section  [in template "logout-confirm.ftl"]
```

C'est la conséquence du Bug #1 : `logout-confirm.ftl` vient du thème parent et utilise l'ancien pattern `; section`. Il appelle notre `template.ftl`, `section` est `null`, ça plante.

**Fix** : surcharger `logout-confirm.ftl` dans notre thème. Et par la même occasion, surcharger tous les templates susceptibles d'être déclenchés en usage courant :

```
login/
├── logout-confirm.ftl      ← déconnexion
├── login-otp.ftl           ← double authentification
└── login-page-expired.ftl  ← session expirée
```

> 📸 **Capture d'écran** : page de confirmation de logout avec le style DICRIM

---

## Le thème account : juste du CSS

La console de gestion de compte (`/account`) en KC 26 est une **SPA React** basée sur PatternFly 5. On ne peut pas la modifier avec des templates Freemarker — elle s'auto-génère côté client.

En revanche, KC permet d'injecter une feuille de style supplémentaire. Il suffit de surcharger les **variables CSS PatternFly** pour appliquer les couleurs DICRIM :

```css
:root {
    --pf-v5-global--primary-color--100: #133478;
    --pf-v5-c-button--m-primary--BackgroundColor: #133478;
    --pf-v5-c-masthead--BackgroundColor: #133478;
    --pf-v5-global--link--Color--hover: #C34A09;
    --pf-v5-global--success-color--100: #009686;
    /* ... */
}
```

> 📸 **Capture d'écran** : console account avec les couleurs DICRIM (header bleu, boutons bleus)

---

## Le thème email

Trois templates HTML aux couleurs DICRIM pour les emails transactionnels :

- **`email-verification.ftl`** — vérification d'adresse à la création de compte
- **`password-reset.ftl`** — réinitialisation de mot de passe (bouton orange)
- **`executeActions.ftl`** — actions requises demandées par un admin

Un fichier `messages_fr.properties` traduit les sujets et corps des emails en français.

> 📸 **Capture d'écran** : aperçu de l'email de réinitialisation de mot de passe

---

## Déploiement sur Clever Cloud

L'addon Managed Keycloak de Clever Cloud stocke les thèmes et plugins dans un **FSBucket** associé à l'instance KC. Deux dossiers y sont disponibles :

- `themes/` — thèmes sous forme de dossiers dézippés
- `providers/` — JARs (plugins, thèmes packagés)

Notre thème est un JAR provider : il va dans `providers/`.

**Procédure complète :**

```bash
# 1. Builder le JAR
cd keycloak && ./mvnw package
```

```
# 2. Uploader target/dicrim-keycloak-theme.jar
#    dans providers/ du FSBucket via la console Clever Cloud
```

```
# 3. Redémarrer l'addon Keycloak depuis la console Clever Cloud
```

```
# 4. Dans l'admin KC : Realm settings → Themes
#    Login theme   → dicrim
#    Account theme → dicrim
#    Email theme   → dicrim
```

> 📸 **Capture d'écran** : FSBucket avec le JAR dans providers/

> 📸 **Capture d'écran** : Realm settings → Themes avec les 3 thèmes dicrim sélectionnés

> **Avantage Clever Cloud** : pas besoin de lancer `kc.sh build` manuellement. Le rebuild Quarkus est géré par la plateforme au redémarrage de l'addon.

---

## Ce qu'on retient

**Sur Keycloak :**
- Le nom du thème account parent change à chaque version majeure (`v2` → `v3`) — toujours vérifier dans l'admin avant de coder
- En KC 26, la syntaxe Freemarker `; section` dans la *définition* d'une macro ne parse pas — il faut adapter le pattern
- Toute variable KC optionnelle (`locale`, etc.) doit être protégée avec l'opérateur `!` de Freemarker
- Surcharger `template.ftl` implique de surcharger **tous** les templates du thème parent qui l'utilisent — sinon ils plantent au premier usage

**Sur l'approche IA-assistée :**
- L'IA explore le code existant avant de générer (couleurs extraites des SCSS, polices trouvées automatiquement) plutôt que de demander des informations déjà disponibles
- Les bugs sont inévitables sur des sujets avec peu de documentation à jour — l'itération rapide est clé
- Les messages d'erreur Keycloak sont précis et suffisants pour diagnostiquer : ligne, colonne, variable manquante

---

*Thème développé avec [Claude Code](https://claude.ai/code) — Anthropic.*
