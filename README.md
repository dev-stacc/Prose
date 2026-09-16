# Prose

An internship management platform for higher-education programs. Students
upload CVs, employers post and review applications, professors approve
workplace evaluations, and program managers oversee the full agreement
lifecycle, from CV approval through signed internship agreements to
end-of-term evaluations.

**Live demo:** https://prose-jet.vercel.app/

Built as a team class project across six one-week sprints by five
contributors. UI is primarily in French with i18next wiring for English.

## Tech stack

- **Backend**: Java 21, Spring Boot 3.5.5, Spring Security + JWT, Spring
  Data JPA / Hibernate, PostgreSQL, Maven
- **Frontend**: React 19, Vite 7, Tailwind CSS 4, React Router 7, axios,
  i18next, React PDF
- **Tests**: JUnit + Spring Security Test (backend), Vitest + Testing
  Library + MSW (frontend)
- **Infra**: Docker Compose for PostgreSQL

## Domain model

Four user roles drive the workflow:

| Role | Capabilities |
|---|---|
| **Student** (`Etudiant`) | Upload CV, browse internship offers, apply to offers, sign agreements |
| **Employer** (`Employeur`) | Post internship offers, review applicants, schedule interviews, sign agreements, evaluate students |
| **Professor** (`Professeur`) | Approve workplace evaluations for their students |
| **Program manager** (`Gestionnaire`) | Approve CVs, oversee agreements, manage the global state of the program |

The lifecycle a single internship travels through:
`CV → Internship offer (Stage) → Application (Candidature) → Interview (Convocation) → Agreement (Entente) → Evaluation`.

## Quick start

### Prerequisites

- JDK 21
- Node.js 20+ and npm
- Docker (for the PostgreSQL container)

### 1. Database

```bash
cp .env.example .env       # then edit values as needed
docker compose up -d       # starts PostgreSQL on :5432
```

### 2. Backend

```bash
./mvnw spring-boot:run     # http://localhost:8080
```

On first run, four demo users are seeded (see below). Schema is created
automatically via `spring.jpa.hibernate.ddl-auto=update`.

### 3. Frontend

```bash
cd prose-fe
cp .env.example .env
npm install
npm run dev                # http://localhost:5173
```

## Demo accounts

The backend seeds these users on startup for local development. Password for
all of them is `password123`.

| Role | Email |
|---|---|
| Student | `etudiant@etudiant.com` |
| Employer | `employeur@employeur.com` |
| Program manager | `gestionnaire@gestionnaire.com` |
| Professor | `professeur@professeur.com` |

> The seeding runner is gated on `@Profile({"dev", "local", "test"})` and
> `spring.profiles.active` defaults to `dev`. For LAN or production, set
> `SPRING_PROFILES_ACTIVE=prod` (or any other value) to skip seeding.

## Deployment

The production setup is split across a self-hosted Kubernetes cluster and
two managed free tiers:

| Layer | Host | Notes |
|---|---|---|
| Frontend | **Vercel** (Hobby) | React build, auto-deployed from `main` via the GitHub integration |
| Backend | **Self-hosted k3s cluster** (Flux GitOps) | Multi-arch container, GHCR-hosted, image bumps auto-committed by Flux |
| Database | **Neon** (free tier) | Managed Postgres over TLS, ready for branch-per-PR |
| Public ingress | **Tailscale Operator + Funnel** | Public HTTPS at `prose-api.<tailnet>.ts.net`, no domain, no port forwarding |
| Secrets | **HashiCorp Vault** via **External Secrets Operator** | `secret/prose/*` injected as env vars |
| CI | **GitHub Actions** + **Buildx** | `mvn verify`, multi-arch image (amd64 + arm64), pushed to GHCR |

### Backend on k3s via Flux

The backend ships as an OCI image. CI builds `linux/amd64` and `linux/arm64`
and pushes to `ghcr.io/<owner>/prose:<UTC-timestamp>-<sha>`. Flux's image
automation (`ImageRepository` + `ImagePolicy` + `ImageUpdateAutomation`)
watches the tag pattern and auto-commits the new tag back to the cluster
manifests repo, which Flux then reconciles, rolling the Deployment.

Manifests under `apps/prose/` declare a `Namespace`, `Deployment`,
internal `Service`, public Tailscale Funnel `Service`, `ExternalSecret`,
and image automation. The Pod is stateless: CV uploads land in Postgres
as `bytea`, no PVC required.

Resource requests are `384Mi` / `200m`, limits `1Gi` / `1000m`, with
`JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75 -XX:InitialRAMPercentage=50`
so the JVM respects the cgroup limit. Readiness and liveness probes hit
Spring Boot Actuator at `/actuator/health/readiness` and
`/actuator/health/liveness`.

### Database on Neon

1. Create a Neon project, copy the JDBC connection string with
   `?sslmode=require` and `?currentSchema=public` if needed.
2. On first Pod start, `spring.jpa.hibernate.ddl-auto=update` creates the
   schema. Apply patches from [`db/patches/`](db/patches/README.md) only
   if you hit the edge cases listed there.

### Public HTTPS via Tailscale Funnel

A second `Service` in the `prose` namespace is annotated for the
Tailscale Operator:

```yaml
annotations:
  tailscale.com/expose: "true"
  tailscale.com/funnel: "true"
  tailscale.com/hostname: "prose-api"
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
  ports:
    - port: 443
      targetPort: 8080
```

The operator spins up a `tsnet` proxy Pod that terminates HTTPS at
`prose-api.<your-tailnet>.ts.net:443` and forwards plaintext to the
backend Service. No custom domain, no Cloudflare account, no port
forwarding. The Funnel attribute must be granted to the operator's
proxy tag (typically `tag:k8s`) in the tailnet ACL.

### Secrets via Vault + External Secrets Operator

`PROSE_DB_URL`, `PROSE_DB_USERNAME`, `PROSE_DB_PASSWORD`, and
`PROSE_JWT_SECRET` live at `secret/prose` in Vault. An
`ExternalSecret` syncs them into a Kubernetes `Secret`, which the
Deployment loads via `envFrom: { secretRef }`. Nothing sensitive ever
sits in the manifests repo.

### Frontend on Vercel

1. Import the repo on Vercel with `prose-fe/` as the project root.
2. Set `VITE_API_BASE_URL=https://prose-api.<your-tailnet>.ts.net` in
   Production and Preview environments.
3. Vercel rebuilds on every push to `main`. Preview URLs
   (`prose-fe-<hash>.vercel.app`) are covered by the CORS wildcard.

### Local / LAN-only deploy

The same jar runs on any host without Kubernetes: `./mvnw clean package`,
set the env vars above pointing at a local Postgres, run the jar, and
point the frontend build at `http://<lan-host>:8080`.

## Tests

```bash
./mvnw test                # backend
cd prose-fe && npm test    # frontend
```

## Project layout

```
.
├── src/main/java/com/AL565/prose/   # Spring Boot app
│   ├── controller/                  # REST controllers per role
│   ├── model/                       # JPA entities (Stage, CV, Candidature, Entente, …)
│   ├── repository/                  # Spring Data repositories
│   ├── security/                    # JWT filter, exceptions
│   └── service/                     # business logic + DTOs
├── prose-fe/src/
│   ├── components/                  # grouped by role: etudiant-, employeur-, gestionnaire-, professeur-
│   ├── services/                    # axios clients per role
│   └── ...                          # context providers, routes, i18n
├── db/patches/                      # manual SQL patches for ddl-auto edge cases
└── compose.yaml                     # PostgreSQL dev container
```

## Internationalization

`prose-fe/public/locales/{en,fr}/translations.json` holds bilingual strings
loaded via `i18next`. Coverage is heavier in French (the original language
of the project); contributions to the English translation are welcome.

## Contributors

In alphabetical order:

- [@LuisFonmarty](https://github.com/LuisFonmarty)
- [@MokhttaAr](https://github.com/MokhttaAr)
- [@RobyCeo](https://github.com/RobyCeo)
- [@dev-stacc](https://github.com/dev-stacc)
- [@ZacharieBouchard16](https://github.com/ZacharieBouchard16)

## License

[MIT](LICENSE). See file for copyright line.

---

# Prose (français)

Plateforme de gestion de stages pour programmes d'enseignement supérieur.
Les étudiants déposent leur CV, les employeurs publient des offres et
évaluent les candidatures, les professeurs approuvent les évaluations en
milieu de travail, et les gestionnaires de programme supervisent tout le
cycle de vie des ententes, de l'approbation du CV jusqu'aux évaluations
de fin de session, en passant par la signature des ententes de stage.

**Démo en ligne :** https://prose-jet.vercel.app/

Réalisé en équipe dans le cadre d'un projet de classe sur six sprints
hebdomadaires par cinq personnes. L'interface est principalement en
français, avec une infrastructure i18next pour l'anglais.

## Pile technique

- **Backend** : Java 21, Spring Boot 3.5.5, Spring Security + JWT, Spring
  Data JPA / Hibernate, PostgreSQL, Maven
- **Frontend** : React 19, Vite 7, Tailwind CSS 4, React Router 7, axios,
  i18next, React PDF
- **Tests** : JUnit + Spring Security Test (backend), Vitest + Testing
  Library + MSW (frontend)
- **Infra** : Docker Compose pour PostgreSQL

## Modèle du domaine

Quatre rôles utilisateurs animent le flux :

| Rôle | Capacités |
|---|---|
| **Étudiant** | Déposer un CV, parcourir les stages, postuler aux offres, signer des ententes |
| **Employeur** | Publier des stages, évaluer les candidatures, planifier des entrevues, signer des ententes, évaluer les étudiants |
| **Professeur** | Approuver les évaluations en milieu de travail de ses étudiants |
| **Gestionnaire** | Approuver les CV, superviser les ententes, gérer l'état global du programme |

Le cycle de vie d'un stage :
`CV → Offre de stage → Candidature → Convocation (entrevue) → Entente → Évaluation`.

## Démarrage rapide

### Prérequis

- JDK 21
- Node.js 20+ et npm
- Docker (pour le conteneur PostgreSQL)

### 1. Base de données

```bash
cp .env.example .env       # ajustez les valeurs au besoin
docker compose up -d       # démarre PostgreSQL sur :5432
```

### 2. Backend

```bash
./mvnw spring-boot:run     # http://localhost:8080
```

Au premier démarrage, quatre comptes de démonstration sont créés (voir
ci-dessous). Le schéma est généré automatiquement via
`spring.jpa.hibernate.ddl-auto=update`.

### 3. Frontend

```bash
cd prose-fe
cp .env.example .env
npm install
npm run dev                # http://localhost:5173
```

## Comptes de démonstration

Le backend crée ces utilisateurs au démarrage pour le développement local.
Le mot de passe est `password123` pour tous.

| Rôle | Courriel |
|---|---|
| Étudiant | `etudiant@etudiant.com` |
| Employeur | `employeur@employeur.com` |
| Gestionnaire | `gestionnaire@gestionnaire.com` |
| Professeur | `professeur@professeur.com` |

> Le seeder est encadré par `@Profile({"dev", "local", "test"})` et
> `spring.profiles.active` vaut `dev` par défaut. Pour un déploiement LAN
> ou en production, définissez `SPRING_PROFILES_ACTIVE=prod` (ou toute
> autre valeur) afin de désactiver le seeding.

## Déploiement

Le déploiement de production est réparti entre un cluster Kubernetes
auto-hébergé et deux services infogérés gratuits :

| Couche | Hébergement | Notes |
|---|---|---|
| Frontend | **Vercel** (Hobby) | Build React, redéployé automatiquement depuis `main` via l'intégration GitHub |
| Backend | **Cluster k3s auto-hébergé** (GitOps Flux) | Conteneur multi-arch, hébergé sur GHCR, mises à jour d'image auto-commitées par Flux |
| Base de données | **Neon** (palier gratuit) | Postgres infogéré sur TLS, prêt pour le branchement par PR |
| Ingress public | **Opérateur Tailscale + Funnel** | HTTPS public à `prose-api.<tailnet>.ts.net`, sans domaine ni redirection de ports |
| Secrets | **HashiCorp Vault** via **External Secrets Operator** | `secret/prose/*` injectés comme variables d'environnement |
| CI | **GitHub Actions** + **Buildx** | `mvn verify`, image multi-arch (amd64 + arm64), poussée sur GHCR |

### Backend sur k3s via Flux

Le backend est livré sous forme d'image OCI. La CI construit
`linux/amd64` et `linux/arm64`, puis pousse vers
`ghcr.io/<owner>/prose:<horodatage-UTC>-<sha>`. L'automatisation d'images
de Flux (`ImageRepository` + `ImagePolicy` + `ImageUpdateAutomation`)
surveille le motif de tag et auto-commit le nouveau tag dans le dépôt de
manifestes du cluster, que Flux réconcilie ensuite, faisant rouler le
Deployment.

Les manifestes sous `apps/prose/` déclarent un `Namespace`, un
`Deployment`, un `Service` interne, un `Service` public via Tailscale
Funnel, un `ExternalSecret`, et l'automatisation d'image. Le Pod est
sans état : les téléversements de CV vont dans Postgres sous forme de
`bytea`, aucun PVC requis.

Ressources : requests `384Mi` / `200m`, limits `1Gi` / `1000m`, avec
`JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75 -XX:InitialRAMPercentage=50`
pour que la JVM respecte la limite cgroup. Les sondes readiness et
liveness frappent Spring Boot Actuator à `/actuator/health/readiness` et
`/actuator/health/liveness`.

### Base de données sur Neon

1. Créez un projet Neon, copiez la chaîne de connexion JDBC avec
   `?sslmode=require`.
2. Au premier démarrage du Pod, `spring.jpa.hibernate.ddl-auto=update`
   crée le schéma. Appliquez les correctifs de
   [`db/patches/`](db/patches/README.md) uniquement si vous rencontrez
   les cas limites qui y sont listés.

### HTTPS public via Tailscale Funnel

Un second `Service` dans le namespace `prose` est annoté pour l'opérateur
Tailscale :

```yaml
annotations:
  tailscale.com/expose: "true"
  tailscale.com/funnel: "true"
  tailscale.com/hostname: "prose-api"
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
  ports:
    - port: 443
      targetPort: 8080
```

L'opérateur crée un Pod proxy `tsnet` qui termine HTTPS à
`prose-api.<votre-tailnet>.ts.net:443` et le transfère en clair vers le
Service backend. Pas de domaine personnalisé, pas de compte Cloudflare,
aucune redirection de ports. L'attribut Funnel doit être accordé au tag
de proxy de l'opérateur (typiquement `tag:k8s`) dans l'ACL du tailnet.

### Secrets via Vault + External Secrets Operator

`PROSE_DB_URL`, `PROSE_DB_USERNAME`, `PROSE_DB_PASSWORD`, et
`PROSE_JWT_SECRET` vivent dans Vault à `secret/prose`. Un
`ExternalSecret` les synchronise dans un `Secret` Kubernetes, que le
Deployment charge via `envFrom: { secretRef }`. Rien de sensible ne se
trouve dans le dépôt de manifestes.

### Frontend sur Vercel

1. Importez le dépôt dans Vercel avec `prose-fe/` comme racine du projet.
2. Définissez `VITE_API_BASE_URL=https://prose-api.<votre-tailnet>.ts.net`
   dans les environnements Production et Preview.
3. Vercel reconstruit à chaque push sur `main`. Les URL de preview
   (`prose-fe-<hash>.vercel.app`) sont couvertes par le wildcard CORS.

### Déploiement local / réseau local

Le même jar fonctionne sur n'importe quel hôte sans Kubernetes :
`./mvnw clean package`, définissez les variables d'environnement
ci-dessus pointant vers un Postgres local, exécutez le jar, et compilez
le frontend en pointant sur `http://<hôte-lan>:8080`.

## Tests

```bash
./mvnw test                # backend
cd prose-fe && npm test    # frontend
```

## Arborescence du projet

```
.
├── src/main/java/com/AL565/prose/   # application Spring Boot
│   ├── controller/                  # contrôleurs REST par rôle
│   ├── model/                       # entités JPA (Stage, CV, Candidature, Entente, …)
│   ├── repository/                  # dépôts Spring Data
│   ├── security/                    # filtre JWT, exceptions
│   └── service/                     # logique métier + DTO
├── prose-fe/src/
│   ├── components/                  # regroupés par rôle : etudiant-, employeur-, gestionnaire-, professeur-
│   ├── services/                    # clients axios par rôle
│   └── ...                          # providers de contexte, routes, i18n
├── db/patches/                      # correctifs SQL manuels pour les cas limites de ddl-auto
└── compose.yaml                     # conteneur PostgreSQL de développement
```

## Internationalisation

`prose-fe/public/locales/{en,fr}/translations.json` contient les chaînes
bilingues chargées via `i18next`. La couverture est plus complète en
français (langue d'origine du projet) ; les contributions à la
traduction anglaise sont bienvenues.

## Contributeurs

Par ordre alphabétique :

- [@LuisFonmarty](https://github.com/LuisFonmarty)
- [@MokhttaAr](https://github.com/MokhttaAr)
- [@RobyCeo](https://github.com/RobyCeo)
- [@dev-stacc](https://github.com/dev-stacc)
- [@ZacharieBouchard16](https://github.com/ZacharieBouchard16)

## Licence

[MIT](LICENSE). Voir le fichier pour la mention de copyright.
