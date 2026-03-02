---
layout: single
title: "Building a Keycloak Theme with AI, deployed on Clever Cloud"
date: 2026-02-27
lang: en
categories: [keycloak, ai]
tags: [keycloak, claude-code, freemarker, clever-cloud, spring-boot]
---

{% include lang-switcher.html %}

> In this article, I explain how I built a custom Keycloak theme for the DICRIM numérique backoffice using Claude Code as an assistant. From the first question to the final production bugs, I share everything — including the mistakes.

---

## Context

[DICRIM numérique](https://dicrim.territoires-prevention.fr/) is a Spring Boot application that I deploy on **Clever Cloud**. It allows local authorities to create and publish their Municipal Major Risk Information Document.  
Backoffice user authentication is handled through Clever Cloud’s **Managed Keycloak** addon (version 26.1.2).

By default, Keycloak provides its own login pages — functional, but very generic. My goal was straightforward: replace them with pages matching the DICRIM numérique design to provide a consistent user experience.

![Default Keycloak login page]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-login-page.png" | relative_url }})
*Default Keycloak login page before customization.*

---

## “What information do you need to get started?”

Everything started with this question I asked Claude Code:

> *"I need to build a Keycloak theme for this project (we use Keycloak for user authentication). Can you help me? What information do you need to get started?"*

Instead of immediately generating code, the assistant began by exploring the project:

- reading SCSS files  
- analyzing the customized Bootstrap color palette  
- locating fonts  

Only then did it ask the relevant questions:

- **Keycloak version?** → 26.1.2  
- **Approach?** → Native Freemarker (no Keycloakify/React)  
- **Pages to customize?** → Login, Account, Email  
- **Deployment?** → JAR provider inside a `keycloak/` folder at the repository root  

These decisions shaped the entire implementation.

### Why native Freemarker instead of Keycloakify?

[Keycloakify](https://www.keycloakify.dev/) is a modern tool that enables building Keycloak themes using React/TypeScript. It’s powerful, but it introduces a separate frontend project, additional tooling, and extra complexity.

For this project, the **native Freemarker** approach was a better fit:

- no additional JS dependencies
- simple `.ftl` templates
- seamless integration with the existing Maven/Java stack

---

## Color palette: extracted from code, not invented

Instead of asking for colors, Claude Code extracted them directly from the project SCSS files:

```scss
/* src/main/scss/bootstrap/_variables.scss */
$primary:   #133478;  /* navy blue */
$secondary: #C34A09;  /* orange */
$success:   #009686;  /* green */
$info:      #E2EEFA;  /* light blue */
```

The same happened for fonts: `Poligon-*.otf` files were automatically located in `src/main/fonts/` and embedded into the theme JAR using the `maven-resources-plugin`.

---

## Project structure

The theme is a **standalone Maven project** placed inside the `keycloak/` directory at the repository root, independent from the main Spring Boot application.

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

### “I don’t have Maven installed locally”

I didn’t have Maven installed on my machine.  
The simple solution was to copy the Maven Wrapper (`mvnw`, `.mvn/`) from the main Spring Boot project.

```bash
cd keycloak && ./mvnw package
# → target/dicrim-keycloak-theme.jar
```

---

## Login page design

### Split-screen layout

I reused the visual identity of the backoffice with a **split-screen** layout:

- **Left panel**: navy background (`#133478`), application identity
- **Right panel**: clean and minimal login form

![DICRIM login page after customization]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-login-page-with-theme.png" | relative_url }})
*Login page after applying the DICRIM theme.*

---

## Bugs — because there are always bugs

I encountered several issues during the first deployments, mostly related to Freemarker subtleties and internal changes introduced in Keycloak 26.

![Logout confirmation page with DICRIM styling]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-logout-with-theme.png" | relative_url }})
*Logout confirmation page after theme customization.*

---

## The account theme: CSS only

The `/account` console in Keycloak 26 is a React SPA based on PatternFly 5.  
The HTML cannot be modified, but I was able to inject a stylesheet and override PatternFly CSS variables.

![Account console with DICRIM colors]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-account-with-theme.png" | relative_url }})
*Keycloak user account console recolored using the DICRIM design system.*

---

## Email theme

I created three transactional email templates:

- `email-verification.ftl`
- `password-reset.ftl`
- `executeActions.ftl`

A `messages_fr.properties` file handles French translations for subjects and content.

---

## Deployment on Clever Cloud

The Clever Cloud Managed Keycloak addon stores themes and plugins inside an associated **FSBucket**.

![FSBucket with JAR in providers]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-fsbucket.png" | relative_url }})
*Uploading the packaged theme (JAR provider) into the Clever Cloud FSBucket.*

![Dicrim themes selected in Keycloak admin]({{ "/assets/images/2026-02-27-theme-keycloak-ia/dicrim-keycloak-config.png" | relative_url }})
*Activating Login, Account and Email themes in the Keycloak administration panel.*

---

## Key takeaways

**About Keycloak:**

- the account theme parent name changes between major versions
- Freemarker remains powerful but strict
- optional variables must always be protected using `!`
- overriding `template.ftl` requires overriding dependent templates as well

**About AI-assisted development:**

- the AI starts by understanding the existing project
- mistakes are part of the workflow
- Keycloak logs are precise enough to diagnose issues quickly

---
