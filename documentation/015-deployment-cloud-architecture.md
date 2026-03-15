---
name: Arquitectura Cloud y Deployment a Producción
overview: "Guía completa para desplegar Context.ai en producción usando Vercel (frontend Next.js), Google Cloud Run (API NestJS), Neon PostgreSQL (base de datos relacional), y Pinecone (base de datos vectorial). Incluye configuración paso a paso, variables de entorno, CI/CD, costos y troubleshooting."
phase: 8
parent_phase: "009-plan-implementacion-detallado.md"
related_docs:
  - "005-technical-architecture.md"
  - "013-fase-8-deployment-monitoring-issues.md"
  - "014-migracion-pgvector-pinecone.md"
decision: "Opción 2 - Vercel + Google Cloud Run"
---

# Arquitectura Cloud y Deployment a Producción

## 1. Resumen Ejecutivo

### Decisión Arquitectónica

Se selecciona la **Opción 2: Vercel + Google Cloud Run** como arquitectura de producción por las siguientes razones:

1. **Vercel** es el creador de Next.js → soporte nativo para Next.js 16 (App Router, SSR, ISR)
2. **Google Cloud Run** → pay-per-request, escala a cero, menor latencia a Gemini API (misma red Google)
3. **Neon PostgreSQL** → serverless, free tier generoso, sin necesidad de pgvector
4. **Pinecone** → base de datos vectorial gestionada, free tier disponible
5. **Auth0** → autenticación gestionada, free tier hasta 7,000 usuarios activos

### Stack de Producción

| Componente | Servicio Cloud | Tier/Plan |
|-----------|---------------|-----------|
| **Frontend** (Next.js 16) | Vercel | Hobby (free) / Pro ($20/mes) |
| **API** (NestJS 11) | Google Cloud Run | Pay-per-use (~$0-15/mes) |
| **PostgreSQL** (relacional) | Neon | Free tier (0.5 GB) |
| **Vectorial** (embeddings) | Pinecone | Free tier (1 index) |
| **Auth** | Auth0 | Free tier (7,500 MAU) |
| **AI/LLM** | Google Gemini API | Free tier generoso |
| **Monitoring** | Sentry | Free tier (5K events/mes) |
| **DNS/CDN** | Vercel Edge Network | Incluido |

### Costo Estimado Mensual

| Servicio | Costo |
|----------|-------|
| Vercel (Hobby) | $0 |
| Cloud Run (bajo tráfico) | $0–5 |
| Neon (Free) | $0 |
| Pinecone (Free) | $0 |
| Auth0 (Free) | $0 |
| Google Gemini API | $0–5 |
| Sentry (Free) | $0 |
| **TOTAL** | **$0–10/mes** |

---

## 2. Arquitectura de Producción

### 2.1 Diagrama General

```
                          ┌─────────────────────────────────────────────────────────────┐
                          │                        Internet                              │
                          │                           │                                  │
                          │              ┌────────────┴────────────┐                     │
                          │              │                         │                     │
                          │    ┌─────────▼──────────┐   ┌─────────▼──────────┐          │
                          │    │     Vercel          │   │  Google Cloud Run  │          │
                          │    │  ┌───────────────┐  │   │  ┌──────────────┐ │          │
                          │    │  │  Next.js 16   │  │   │  │  NestJS 11   │ │          │
                          │    │  │  (SSR/SSG)    │  │──▶│  │  REST API    │ │          │
                          │    │  │  React 19     │  │   │  │  Port 3001   │ │          │
                          │    │  │  TailwindCSS  │  │   │  │  Swagger     │ │          │
                          │    │  └───────────────┘  │   │  └──────┬───────┘ │          │
                          │    │  Edge Network (CDN) │   │         │         │          │
                          │    │  SSL automático     │   │  Docker Container │          │
                          │    │  Preview Deploys    │   │  Auto-scaling     │          │
                          │    └────────────────────┘   │  Scale-to-zero    │          │
                          │                              └─────────┬─────────┘          │
                          │                                        │                     │
                          │              ┌─────────────────────────┼──────────────┐      │
                          │              │                         │              │      │
                          │    ┌─────────▼──────────┐   ┌─────────▼────┐  ┌──────▼───┐ │
                          │    │   Neon PostgreSQL   │   │   Pinecone   │  │  Gemini  │ │
                          │    │  ┌───────────────┐  │   │  ┌─────────┐│  │  API     │ │
                          │    │  │  Relational   │  │   │  │ Vectors ││  │  ┌─────┐ │ │
                          │    │  │  Data         │  │   │  │ 3072D   ││  │  │ LLM │ │ │
                          │    │  │  - users      │  │   │  │ cosine  ││  │  │ Emb │ │ │
                          │    │  │  - sectors    │  │   │  └─────────┘│  │  └─────┘ │ │
                          │    │  │  - sources    │  │   │  Namespaces │  │  gemini-  │ │
                          │    │  │  - fragments  │  │   │  por sector │  │  2.5-flash│ │
                          │    │  │  - messages   │  │   └─────────────┘  └──────────┘ │
                          │    │  │  - convos     │  │                                  │
                          │    │  │  - roles/perms│  │                                  │
                          │    │  └───────────────┘  │                                  │
                          │    │  Serverless PG      │                                  │
                          │    │  Auto-suspend       │                                  │
                          │    └────────────────────┘                                   │
                          │                                                              │
                          │    ┌─────────────────┐         ┌────────────────┐            │
                          │    │     Auth0        │         │    Sentry      │            │
                          │    │  Authentication  │         │  Error Track   │            │
                          │    │  JWT tokens      │         │  Performance   │            │
                          │    └─────────────────┘         └────────────────┘            │
                          └─────────────────────────────────────────────────────────────┘
```

### 2.2 Flujo de Request

```
Usuario → Vercel CDN → Next.js (SSR) → Auth0 (JWT) → Cloud Run (API) → Neon PG + Pinecone + Gemini
```

1. El usuario accede a `https://app.contextai.com`
2. Vercel sirve la app Next.js con SSR
3. Next.js valida la sesión con Auth0
4. El frontend hace requests a `https://api.contextai.com/api/v1/...`
5. Cloud Run recibe el request, valida JWT con Auth0
6. La API consulta Neon (datos) y/o Pinecone (vectores) y/o Gemini (AI)
7. Respuesta regresa al usuario

---

## 3. Prerequisitos

### 3.1 Cuentas Necesarias

| Servicio | URL | Cuenta |
|----------|-----|--------|
| Vercel | [vercel.com](https://vercel.com) | GitHub SSO |
| Google Cloud | [cloud.google.com](https://cloud.google.com) | Google Account |
| Neon | [neon.tech](https://neon.tech) | GitHub SSO |
| Pinecone | [pinecone.io](https://www.pinecone.io) | Email |
| Auth0 | [auth0.com](https://auth0.com) | Existente |
| Sentry | [sentry.io](https://sentry.io) | Existente |

### 3.2 CLIs Necesarios

```bash
# Vercel CLI
npm i -g vercel

# Google Cloud CLI
brew install google-cloud-sdk   # macOS
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# Docker (para Cloud Run)
docker --version  # Debe ser >= 20.x
```

### 3.3 Resolver el Shared Package

El API tiene una dependencia local:

```json
"@context-ai-project/shared": "^0.1.0"
```

Esto **no funciona en producción** (no hay monorepo en el servidor). Opciones:

#### Publicar en GitHub Packages

```bash
cd context-ai-shared
pnpm build

# Publicar a GitHub Packages
npm publish --registry=https://npm.pkg.github.com
```

Luego en `context-ai-api/package.json`:

```json
"@context-ai-project/shared": "^0.1.0"
```

Y crear `.npmrc`:

```
@context-ai-project:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```


Modificar el build del API para incluir shared inline:

```json
// tsconfig.json del API
{
  "compilerOptions": {
    "paths": {
      "@context-ai-project/shared": ["../context-ai-shared/src"]
    }
  }
}
```
---

## 4. Configuración de Neon PostgreSQL

### 4.1 Crear Proyecto en Neon

1. Ir a [console.neon.tech](https://console.neon.tech)
2. **Create a project:**
   - Name: `context-ai`
   - Region: `us-east-1` (cercano a Cloud Run)
   - PostgreSQL version: `16`
3. Obtener connection string:
   ```
   postgresql://username:password@ep-xxxx.us-east-1.aws.neon.tech/contextai?sslmode=require
   ```

### 4.2 Crear la Base de Datos

```sql
-- Conectarse via psql o Neon SQL Editor

-- Extensiones necesarias (ya NO se necesita pgvector)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Ejecutar migraciones
-- Opción 1: Desde el CLI local
DATABASE_URL="postgresql://..." pnpm migration:run

-- Opción 2: Ejecutar SQL directamente desde migrations/init/
```

### 4.3 Variables de Entorno de Neon

```bash
# Para Cloud Run (API)
DB_HOST=ep-xxxx.eu-central-1.aws.neon.tech
DB_PORT=5432
DB_USERNAME=your_neon_user
DB_PASSWORD=your_neon_password
DB_DATABASE=contextai
DB_SSL_REJECT_UNAUTHORIZED=false   # Neon requiere SSL
```

### 4.4 Configuración SSL

La API ya tiene soporte SSL en `database.config.ts`:

```typescript
ssl: isProduction
  ? {
      rejectUnauthorized: process.env.DB_SSL_REJECT_UNAUTHORIZED !== 'false',
    }
  : false,
```

Para Neon, configurar `DB_SSL_REJECT_UNAUTHORIZED=false` en producción.

### 4.5 Neon Free Tier Limits

| Recurso | Límite |
|---------|--------|
| Storage | 0.5 GB |
| Compute | 0.25 vCPU |
| Branches | 10 |
| Active compute time | 191.9 horas/mes |
| Auto-suspend | Tras 5 min de inactividad |

> **Nota:** Auto-suspend puede causar cold starts de ~1-3 segundos. Aceptable para TFM.

---

## 5. Configuración de Pinecone

### 5.1 Crear Index

1. Ir a [app.pinecone.io](https://app.pinecone.io)
2. **Create Index:**
   - Name: `context-ai`
   - Dimensions: `3072`
   - Metric: `cosine`
   - Cloud: `AWS`
   - Region: `us-east-1`
   - Plan: `Starter (Free)`

### 5.2 Obtener API Key

1. Dashboard → API Keys
2. Copiar la default API key
3. Configurar como variable de entorno

### 5.3 Variables de Entorno

```bash
PINECONE_API_KEY=pcsk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
PINECONE_INDEX=context-ai
```

### 5.4 Pinecone Free Tier Limits

| Recurso | Límite |
|---------|--------|
| Indexes | 1 |
| Dimensions | Hasta 20,000 |
| Vectors | ~100K (varía) |
| Storage | 2 GB |
| Writes | 2 writes/sec |
| Reads | 10 reads/sec |

---

## 6. Despliegue del Frontend en Vercel

### 6.1 Setup Inicial

```bash
cd context-ai-front

# Login en Vercel
vercel login

# Vincular proyecto
vercel link
# ? Set up "context-ai-front"? → Yes
# ? Which scope? → tu-equipo
# ? Link to existing project? → No (primera vez)
# ? What's your project's name? → context-ai-front
# ? In which directory is your code located? → ./
```

### 6.2 Configurar Variables de Entorno

En el **Vercel Dashboard** → Settings → Environment Variables:

| Variable | Valor | Entornos |
|----------|-------|----------|
| `AUTH0_SECRET` | `[openssl rand -hex 32]` | Production, Preview |
| `AUTH0_BASE_URL` | `https://app.contextai.com` | Production |
| `AUTH0_BASE_URL` | `https://preview-xxx.vercel.app` | Preview |
| `AUTH0_ISSUER_BASE_URL` | `https://your-tenant.auth0.com` | All |
| `AUTH0_CLIENT_ID` | `your_client_id` | All |
| `AUTH0_CLIENT_SECRET` | `your_client_secret` | All |
| `AUTH0_AUDIENCE` | `https://api.contextai.com` | All |
| `NEXT_PUBLIC_API_URL` | `https://api.contextai.com` | Production |
| `NEXT_PUBLIC_API_URL` | `https://api-staging.contextai.com` | Preview |
| `NEXT_PUBLIC_SENTRY_DSN` | `https://xxx@sentry.io/xxx` | All |
| `NODE_ENV` | `production` | Production |

### 6.3 Configurar next.config.ts para Producción

Verificar que la configuración actual es compatible con Vercel. El archivo actual ya es compatible:

```typescript
// next.config.ts (actual - ya es compatible con Vercel)
import type { NextConfig } from 'next';
import createNextIntlPlugin from 'next-intl/plugin';

const withNextIntl = createNextIntlPlugin('./src/i18n.ts');

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 's.gravatar.com' },
      { protocol: 'https', hostname: 'lh3.googleusercontent.com' },
      { protocol: 'https', hostname: 'avatars.githubusercontent.com' },
      { protocol: 'https', hostname: '*.auth0.com' },
    ],
  },
};

export default withNextIntl(nextConfig);
```

> **Nota:** No se necesita `output: 'standalone'` para Vercel (solo para Docker/Cloud Run).

### 6.4 Deploy

```bash
# Deploy a producción
vercel --prod

# O configurar auto-deploy desde GitHub:
# Vercel Dashboard → Settings → Git → Connected Git Repository
# Branch: main → Production
# Branch: develop → Preview
```

### 6.5 Custom Domain

1. Vercel Dashboard → Settings → Domains
2. Agregar `app.contextai.com`
3. Configurar DNS:
   ```
   CNAME  app  cname.vercel-dns.com
   ```
4. SSL se configura automáticamente

### 6.6 Auto-Deploy desde GitHub

1. Vercel Dashboard → Settings → Git
2. Conectar repositorio `context-ai-front`
3. Production Branch: `main`
4. Preview: todas las ramas y PRs

Cada push a `main` despliega automáticamente. Cada PR genera un Preview Deploy.

---

## 7. Despliegue del API en Google Cloud Run

### 7.1 Crear Proyecto en Google Cloud

```bash
# Crear proyecto (si no existe)
gcloud projects create context-ai-prod --name="Context AI Production"

# Seleccionar proyecto
gcloud config set project context-ai-prod

# Habilitar APIs necesarias
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com
```

### 7.2 Crear Dockerfile para el API

**Archivo:** `context-ai-api/Dockerfile`

```dockerfile
# ============================================
# Stage 1: Build
# ============================================
FROM node:22-alpine AS builder

WORKDIR /app

# Install pnpm
RUN corepack enable && corepack prepare pnpm@10 --activate

# Copy package files
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./

# Copy shared package (si se usa Opción B)
# COPY ../context-ai-shared ./context-ai-shared

# Install all dependencies (including devDependencies for build)
RUN pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build application
RUN pnpm build

# ============================================
# Stage 2: Production
# ============================================
FROM node:22-alpine AS production

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nestjs -u 1001

# Install pnpm
RUN corepack enable && corepack prepare pnpm@10 --activate

# Copy package files
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./

# Install production dependencies only
RUN pnpm install --prod --frozen-lockfile

# Copy built application from builder
COPY --from=builder /app/dist ./dist

# Change ownership
RUN chown -R nestjs:nodejs /app

# Switch to non-root user
USER nestjs

# Cloud Run uses PORT env var
ENV PORT=3001
EXPOSE 3001

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "const http = require('http'); http.get('http://localhost:3001/api/v1/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

# Start application
CMD ["node", "dist/main.js"]
```

**Archivo:** `context-ai-api/.dockerignore`

```
node_modules
.git
.github
.husky
coverage
test
test-data
docs
*.md
.env*
.genkit
dist
.pnpm-store
```

### 7.3 Crear Artifact Registry (Container Registry)

```bash
# Crear repositorio de imágenes Docker
gcloud artifacts repositories create context-ai \
  --repository-format=docker \
  --location=europe-west1 \
  --description="Context AI Docker images"

# Configurar Docker para usar Artifact Registry
gcloud auth configure-docker europe-west1-docker.pkg.dev
```

### 7.4 Build y Push de la Imagen

```bash
cd context-ai-api

# Build imagen
docker build -t europe-west1-docker.pkg.dev/context-ai-prod/context-ai/api:latest .

# Push a Artifact Registry
docker push europe-west1-docker.pkg.dev/context-ai-prod/context-ai/api:latest
```

**O usar Cloud Build (recomendado):**

```bash
# Build directamente en Google Cloud
gcloud builds submit --tag europe-west1-docker.pkg.dev/context-ai-prod/context-ai/api:latest
```

### 7.5 Configurar Secrets en Secret Manager

```bash
# Crear secrets
echo -n "your-db-password" | gcloud secrets create DB_PASSWORD --data-file=-
# Vertex AI usa ADC del service account de Cloud Run — no requiere secret para GOOGLE_API_KEY
echo -n "your-pinecone-api-key" | gcloud secrets create PINECONE_API_KEY --data-file=-
echo -n "your-auth0-domain" | gcloud secrets create AUTH0_DOMAIN --data-file=-
echo -n "your-auth0-audience" | gcloud secrets create AUTH0_AUDIENCE --data-file=-
echo -n "your-sentry-dsn" | gcloud secrets create SENTRY_DSN --data-file=-
```

### 7.6 Deploy a Cloud Run

```bash
gcloud run deploy context-ai-api \
  --image=europe-west1-docker.pkg.dev/context-ai-prod/context-ai/api:latest \
  --region=europe-west1 \
  --platform=managed \
  --allow-unauthenticated \
  --port=3001 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=3 \
  --concurrency=80 \
  --timeout=300 \
  --set-env-vars="NODE_ENV=production" \
  --set-env-vars="PORT=3001" \
  --set-env-vars="API_PREFIX=api/v1" \
  --set-env-vars="DB_HOST=ep-xxxx.us-east-1.aws.neon.tech" \
  --set-env-vars="DB_PORT=5432" \
  --set-env-vars="DB_USERNAME=your_neon_user" \
  --set-env-vars="DB_DATABASE=contextai" \
  --set-env-vars="DB_SSL_REJECT_UNAUTHORIZED=false" \
  --set-env-vars="PINECONE_INDEX=context-ai" \
  --set-env-vars="FRONTEND_URL=https://app.contextai.com" \
  --set-env-vars="ALLOWED_ORIGINS=https://app.contextai.com" \
  --set-env-vars="RATE_LIMIT_WINDOW_MS=60000" \
  --set-env-vars="RATE_LIMIT_MAX_REQUESTS=100" \
  --set-secrets="DB_PASSWORD=DB_PASSWORD:latest" \
  --set-env-vars="GCP_PROJECT_ID=context-ai-production" \
  --set-secrets="PINECONE_API_KEY=PINECONE_API_KEY:latest" \
  --set-secrets="AUTH0_DOMAIN=AUTH0_DOMAIN:latest" \
  --set-secrets="AUTH0_AUDIENCE=AUTH0_AUDIENCE:latest" \
  --set-secrets="SENTRY_DSN=SENTRY_DSN:latest"
```

### 7.7 Custom Domain para Cloud Run

```bash
# Mapear dominio personalizado
gcloud beta run domain-mappings create \
  --service=context-ai-api \
  --domain=api.contextai.com \
  --region=europe-west1
```

Configurar DNS:
```
CNAME  api  ghs.googlehosted.com
```

### 7.8 Consideraciones de Cloud Run

| Configuración | Valor | Razón |
|--------------|-------|-------|
| `min-instances=0` | Scale to zero | Ahorro de costos (TFM) |
| `max-instances=3` | Límite | Prevenir costos excesivos |
| `memory=512Mi` | Suficiente para NestJS | Puede ajustarse si es necesario |
| `cpu=1` | 1 vCPU | Suficiente para tráfico bajo |
| `timeout=300` | 5 minutos | Para procesamiento de documentos/RAG |
| `concurrency=80` | Por instancia | Default de Cloud Run |

> **Cold starts:** Con `min-instances=0`, la primera request después de inactividad puede tardar 3-10 segundos. Aceptable para TFM. Para eliminar cold starts, cambiar a `min-instances=1` (~$10/mes).

---

## 8. Configuración de Auth0 para Producción

### 8.1 Crear Aplicación de Producción en Auth0

1. Auth0 Dashboard → Applications → Create Application
2. **API (Backend):**
   - Name: `Context.AI API - Production`
   - Type: Regular Web Application
   - Allowed Callback URLs: `https://app.contextai.com/api/auth/callback`
   - Allowed Logout URLs: `https://app.contextai.com`
   - Allowed Web Origins: `https://app.contextai.com`

3. **API Identifier:**
   - Auth0 Dashboard → APIs
   - Crear API: `https://api.contextai.com`
   - Signing Algorithm: RS256

### 8.2 Variables de Auth0 por Entorno

**Frontend (Vercel):**

```bash
AUTH0_SECRET=<openssl rand -hex 32>
AUTH0_BASE_URL=https://app.contextai.com
AUTH0_ISSUER_BASE_URL=https://your-tenant.auth0.com
AUTH0_CLIENT_ID=your_prod_client_id
AUTH0_CLIENT_SECRET=your_prod_client_secret
AUTH0_AUDIENCE=https://api.contextai.com
```

**API (Cloud Run):**

```bash
AUTH0_DOMAIN=your-tenant.auth0.com
AUTH0_AUDIENCE=https://api.contextai.com
AUTH0_ISSUER=https://your-tenant.auth0.com/
```

---

## 9. CI/CD con GitHub Actions

### 9.1 Deploy API a Cloud Run

**Archivo:** `context-ai-api/.github/workflows/deploy-production.yml`

```yaml
name: Deploy API to Cloud Run

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test:
    name: Test & Build
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine  # Ya no necesita pgvector
        env:
          POSTGRES_DB: contextai_test
          POSTGRES_USER: contextai_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - uses: pnpm/action-setup@v4
        with:
          version: 10

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm lint

      - name: Build
        run: pnpm build

      - name: Test
        run: pnpm test
        env:
          DATABASE_HOST: localhost
          DATABASE_PORT: 5432
          DATABASE_USER: contextai_user
          DATABASE_PASSWORD: test_password
          DATABASE_NAME: contextai_test
          NODE_ENV: test

  deploy:
    name: Deploy to Cloud Run
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'

    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker europe-west1-docker.pkg.dev

      - name: Build and Push Docker image
        run: |
          docker build -t europe-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/context-ai/api:${{ github.sha }} .
          docker build -t europe-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/context-ai/api:latest .
          docker push europe-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/context-ai/api:${{ github.sha }}
          docker push europe-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/context-ai/api:latest

      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: context-ai-api
          region: europe-west1
          image: europe-west1-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/context-ai/api:${{ github.sha }}
          env_vars: |
            NODE_ENV=production
            PORT=3001
            API_PREFIX=api/v1
            FRONTEND_URL=https://app.contextai.com
            ALLOWED_ORIGINS=https://app.contextai.com
            PINECONE_INDEX=context-ai

      - name: Smoke Test
        run: |
          URL=$(gcloud run services describe context-ai-api --region=europe-west1 --format='value(status.url)')
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL/api/v1/health")
          if [ "$STATUS" != "200" ]; then
            echo "❌ Health check failed with status $STATUS"
            exit 1
          fi
          echo "✅ Health check passed"
```

### 9.2 Deploy Frontend a Vercel

Vercel se configura automáticamente con GitHub. No se necesita un workflow de GitHub Actions.

**Configuración en Vercel Dashboard:**

1. **Git Integration:** Conectar repositorio `context-ai-front`
2. **Production Branch:** `main`
3. **Build Command:** `pnpm build`
4. **Output Directory:** `.next`
5. **Install Command:** `pnpm install --frozen-lockfile`
6. **Node.js Version:** `22.x`

Cada push a `main` genera un deploy automático a producción.
Cada PR genera un Preview Deploy con URL única.

### 9.3 GitHub Secrets Necesarios

**Para el API (Cloud Run):**

| Secret | Descripción |
|--------|-------------|
| `GCP_PROJECT_ID` | ID del proyecto en Google Cloud |
| `WIF_PROVIDER` | Workload Identity Federation provider |
| `WIF_SERVICE_ACCOUNT` | Service account para Cloud Run |

**Para variables de entorno (ya configuradas en Cloud Run/Vercel):**

Los secrets de aplicación (DB_PASSWORD, API keys, etc.) se gestionan directamente en Cloud Run Secret Manager y Vercel Environment Variables, **no** en GitHub Secrets.

---

## 10. Variables de Entorno — Resumen Completo

### 10.1 API (Cloud Run)

| Variable | Valor | Tipo |
|----------|-------|------|
| `NODE_ENV` | `production` | Env var |
| `PORT` | `3001` | Env var |
| `API_PREFIX` | `api/v1` | Env var |
| `DB_HOST` | `ep-xxxx.neon.tech` | Env var |
| `DB_PORT` | `5432` | Env var |
| `DB_USERNAME` | `contextai_user` | Env var |
| `DB_PASSWORD` | `xxx` | **Secret** |
| `DB_DATABASE` | `contextai` | Env var |
| `DB_SSL_REJECT_UNAUTHORIZED` | `false` | Env var |
| `GCP_PROJECT_ID` | `context-ai-production` | Env var |
| `PINECONE_API_KEY` | `xxx` | **Secret** |
| `PINECONE_INDEX` | `context-ai` | Env var |
| `AUTH0_DOMAIN` | `your-tenant.auth0.com` | Env var |
| `AUTH0_AUDIENCE` | `https://api.contextai.com` | Env var |
| `AUTH0_ISSUER` | `https://your-tenant.auth0.com/` | Env var |
| `FRONTEND_URL` | `https://app.contextai.com` | Env var |
| `ALLOWED_ORIGINS` | `https://app.contextai.com` | Env var |
| `RATE_LIMIT_WINDOW_MS` | `60000` | Env var |
| `RATE_LIMIT_MAX_REQUESTS` | `100` | Env var |
| `SENTRY_DSN` | `https://xxx@sentry.io/xxx` | **Secret** |
| `LOG_LEVEL` | `info` | Env var |

### 10.2 Frontend (Vercel)

| Variable | Valor | Tipo |
|----------|-------|------|
| `AUTH0_SECRET` | `xxx` | Secret (Vercel) |
| `AUTH0_BASE_URL` | `https://app.contextai.com` | Env var |
| `AUTH0_ISSUER_BASE_URL` | `https://your-tenant.auth0.com` | Env var |
| `AUTH0_CLIENT_ID` | `xxx` | Secret (Vercel) |
| `AUTH0_CLIENT_SECRET` | `xxx` | Secret (Vercel) |
| `AUTH0_AUDIENCE` | `https://api.contextai.com` | Env var |
| `NEXT_PUBLIC_API_URL` | `https://api.contextai.com` | Env var |
| `NEXT_PUBLIC_SENTRY_DSN` | `https://xxx@sentry.io/xxx` | Env var |
| `NODE_ENV` | `production` | Env var |

---

## 11. CORS y Seguridad

### 11.1 Configuración CORS en la API

La API debe permitir requests desde el dominio de Vercel:

```typescript
// main.ts
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') ?? ['https://app.contextai.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
});
```

### 11.2 Helmet (Security Headers)

Ya configurado en la API con `helmet`. Se aplica automáticamente en producción.

### 11.3 Rate Limiting

Ya configurado con `express-rate-limit`. Los límites se configuran via variables de entorno.

---

## 12. Monitoreo y Observabilidad

### 12.1 Sentry (Error Tracking)

**Backend (Cloud Run):**
- Ya tiene `SENTRY_DSN` configurado
- Captura errores automáticamente con `@sentry/node`

**Frontend (Vercel):**
- Ya tiene `@sentry/nextjs` instalado
- Configurar en `sentry.client.config.ts` y `sentry.server.config.ts`

### 12.2 Google Cloud Monitoring

Cloud Run incluye automáticamente:
- Request count y latency
- Instance count
- Memory y CPU usage
- Error rate
- Logs (Cloud Logging)

Acceder en: Google Cloud Console → Cloud Run → context-ai-api → Metrics

### 12.3 Vercel Analytics

Vercel incluye automáticamente:
- Web Vitals (LCP, FID, CLS)
- Page views
- Build times
- Edge function metrics

---

## 13. Base de Datos — Migraciones en Producción

### 13.1 Ejecutar Migraciones

```bash
# Desde tu máquina local, apuntando a la BD de producción:
DATABASE_URL="postgresql://user:pass@ep-xxxx.neon.tech/contextai?sslmode=require" \
  pnpm migration:run
```

### 13.2 Migración Inicial

Los scripts SQL de `migrations/init/` deben ejecutarse primero:

```bash
# Conectar a Neon via psql
psql "postgresql://user:pass@ep-xxxx.neon.tech/contextai?sslmode=require"

# Ejecutar scripts en orden
\i migrations/init/001_extensions.sql
\i migrations/init/003_rbac_tables.sql

# Luego ejecutar migraciones TypeORM
pnpm migration:run
```

### 13.3 Backups

Neon incluye:
- **Point-in-time recovery** (hasta 7 días en free tier)
- **Branching** (crear copias de la BD para testing)

---

## 14. Checklist de Deployment

### Pre-Deployment

- [ ] Todas las cuentas creadas (Vercel, GCP, Neon, Pinecone)
- [ ] Auth0 configurado para producción (URLs, callbacks)
- [ ] DNS configurado (app.contextai.com, api.contextai.com)
- [ ] Shared package resuelto (GitHub Packages o build inline)
- [ ] Neon PostgreSQL creado y migraciones ejecutadas
- [ ] Pinecone index creado (3072D, cosine)
- [ ] Google Cloud project creado y APIs habilitadas
- [ ] Secrets configurados en Secret Manager
- [ ] Variables de entorno configuradas en Vercel

### Deployment

- [ ] Frontend desplegado en Vercel
- [ ] API Dockerfile creado y probado localmente
- [ ] API imagen pushed a Artifact Registry
- [ ] API desplegada en Cloud Run
- [ ] Custom domains configurados
- [ ] SSL verificado en ambos servicios

### Post-Deployment

- [ ] Health check del API: `curl https://api.contextai.com/api/v1/health`
- [ ] Frontend accesible: `https://app.contextai.com`
- [ ] Login con Auth0 funciona
- [ ] Ingesta de documento funciona
- [ ] Chat/RAG query funciona
- [ ] Sentry recibe eventos
- [ ] Logs visibles en Cloud Logging
- [ ] CORS funciona correctamente
- [ ] Rate limiting funciona
- [ ] No hay errores en consola del navegador

### CI/CD

- [ ] GitHub Actions workflow configurado para API
- [ ] Vercel auto-deploy configurado para Frontend
- [ ] Preview deploys funcionan en PRs
- [ ] Smoke tests pasan después de deploy

---

## 15. Troubleshooting

### Problema: Cold starts lentos en Cloud Run

**Síntoma:** Primera request tarda 5-10 segundos.

**Solución:**
```bash
# Mantener 1 instancia siempre activa (~$10/mes)
gcloud run services update context-ai-api \
  --min-instances=1 \
  --region=europe-west1
```

### Problema: Cold starts en Neon PostgreSQL

**Síntoma:** Query lenta después de inactividad.

**Solución:** Neon auto-suspende tras 5 min. Opciones:
1. Aceptar 1-3s cold start (suficiente para TFM)
2. Configurar keep-alive con un cron job

### Problema: CORS errors

**Síntoma:** `Access-Control-Allow-Origin` error en el browser.

**Solución:** Verificar que `ALLOWED_ORIGINS` en Cloud Run incluye el dominio exacto de Vercel (con `https://` y sin trailing slash).

### Problema: Auth0 callback error

**Síntoma:** Error después de login "Callback URL mismatch".

**Solución:** Verificar en Auth0 Dashboard que:
- `Allowed Callback URLs` incluye `https://app.contextai.com/api/auth/callback`
- `Allowed Logout URLs` incluye `https://app.contextai.com`
- `Allowed Web Origins` incluye `https://app.contextai.com`

### Problema: Build falla por @context-ai-project/shared

**Síntoma:** `Cannot find module '@context-ai-project/shared'` durante docker build.

**Solución:** Publicar shared package en GitHub Packages o incluir en Dockerfile:
```dockerfile
COPY context-ai-shared/dist ./node_modules/@context-ai-project/shared/dist
COPY context-ai-shared/package.json ./node_modules/@context-ai-project/shared/package.json
```

### Problema: Neon SSL connection error

**Síntoma:** `Error: self signed certificate in certificate chain`

**Solución:** Configurar `DB_SSL_REJECT_UNAUTHORIZED=false` en Cloud Run.

---

## 16. Escalabilidad Futura

Si el proyecto crece más allá del TFM:

| Cambio | Cuándo | Costo Adicional |
|--------|--------|-----------------|
| Vercel Pro | >100K pages/mes | $20/mes |
| Cloud Run `min-instances=1` | Eliminar cold starts | ~$10/mes |
| Neon Pro | >0.5 GB storage | $19/mes |
| Pinecone Standard | >100K vectores | $70/mes |
| Auth0 Essential | >7,500 MAU | $23/mes |
| Redis (caching) | Alta latencia | ~$10/mes |
| Cloud Armor (WAF) | Producción real | ~$5/mes |

---

## 17. Comparativa con Alternativas Descartadas

### ¿Por qué no Railway para el API?

| Criterio | Railway | Cloud Run | Ganador |
|----------|---------|-----------|---------|
| Latencia a Gemini | Red pública | Red interna Google | **Cloud Run** |
| Scale to zero | No (siempre activo) | Sí | **Cloud Run** |
| Costo mínimo | ~$5/mes siempre | $0 sin tráfico | **Cloud Run** |
| Facilidad | Más simple | Requiere Docker | Railway |
| Secret management | Env vars | Secret Manager | **Cloud Run** |

### ¿Por qué no Cloud SQL para PostgreSQL?

| Criterio | Cloud SQL | Neon | Ganador |
|----------|-----------|------|---------|
| Costo | ~$10-15/mes mínimo | $0 free tier | **Neon** |
| Serverless | No | Sí (auto-suspend) | **Neon** |
| Branching | No | Sí | **Neon** |
| Cold starts | No (siempre activo) | 1-3s | Cloud SQL |
| Para TFM | Overkill | Perfecto | **Neon** |

### ¿Por qué Vercel y no Amplify o Netlify?

| Criterio | Vercel | Amplify | Netlify | Ganador |
|----------|--------|---------|---------|---------|
| Next.js 16 support | Nativo (creators) | Parcial | Parcial | **Vercel** |
| App Router | Completo | Limitado | Limitado | **Vercel** |
| Preview deploys | Automático | Manual | Automático | Vercel/Netlify |
| Edge Functions | Sí | No | Sí | Vercel/Netlify |
| SSR performance | Optimizado | Básico | Básico | **Vercel** |

---

## 18. Referencias

- [Vercel Documentation](https://vercel.com/docs)
- [Google Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Neon PostgreSQL Documentation](https://neon.tech/docs)
- [Pinecone Documentation](https://docs.pinecone.io/)
- [Auth0 Documentation](https://auth0.com/docs)
- [Sentry Next.js Documentation](https://docs.sentry.io/platforms/javascript/guides/nextjs/)
- [Google Artifact Registry](https://cloud.google.com/artifact-registry/docs)
- [Google Secret Manager](https://cloud.google.com/secret-manager/docs)
- [GitHub Actions for Cloud Run](https://github.com/google-github-actions/deploy-cloudrun)

