# Context.ai 🧠✨
### El Orquestador de Cultura y Conocimiento Dinámico para Startups

---

## 📝 Descripción General
**Context.ai** es una solución de ingeniería diseñada para mitigar los problemas de fragmentación de información y la alta rotación en entornos de startups de alto crecimiento. A diferencia de las wikis tradicionales, Context.ai utiliza **IA Generativa y arquitecturas RAG (Retrieval-Augmented Generation)** para actuar como un "cerebro central" que facilita el onboarding autónomo y la retención del conocimiento táctico.

Este **proyecto** nace como el Trabajo de Fin de Máster del **Máster en Desarrollo con IA**, aplicando principios avanzados de **Ingeniería de Software**, **Arquitecturas Distribuidas** y **Desarrollo Potenciado por IA**.

### El Problema
* **Fragmentación**: Información dispersa en Slack, Confluence y documentos locales.
* **Onboarding costoso**: Los veteranos pierden tiempo valioso guiando a los nuevos.
* **Fuga de Conocimiento**: Cuando un empleado se va, su "know-how" desaparece con él.

### Dónde está esta documentación

Este README y la carpeta **`documentation/`** son el punto de entrada para el setup y la documentación del proyecto. Los repositorios **context-ai-api**, **context-ai-front** y **context-ai-shared** se clonan en la misma raíz (por ejemplo `Context.ai/`). Las guías técnicas del backend están en **`context-ai-api/docs/`** (Auth0, base de datos, variables de entorno, etc.).

---

## 🛠️ Stack Tecnológico
Para garantizar la escalabilidad, mantenibilidad y robustez, se ha seleccionado el siguiente stack:

* **Runtime & Lenguaje**: Node.js 22+ con TypeScript (tipado estricto).
* **Backend**: NestJS siguiendo patrones de **Arquitectura Limpia (Clean Architecture)** y **DDD**.
* **Frontend**: Next.js (App Router) optimizado para **Core Web Vitals**.
* **Orquestación de IA**: **Google Genkit** para flujos agénticos y Tool Calling.
* **Modelos (LLM)**: **Gemini 2.5 Flash** por su amplia ventana de contexto y eficiencia.
* **Vector Store**: **Pinecone** para almacenamiento y búsqueda de embeddings vectoriales.
* **Embeddings**: **gemini-embedding-001** (3072 dimensiones).
* **Base de Datos**: PostgreSQL (Neon) para datos relacionales.
* **Observabilidad**: **Sentry** para monitorización de errores.

---

## 📦 Repositorios

El proyecto está organizado bajo la organización [context-ai-project](https://github.com/context-ai-project) en GitHub:

| Repositorio | Descripción |
|-------------|-------------|
| [context-ai-api](https://github.com/context-ai-project/context-ai-api) | Backend API (NestJS, Clean Architecture) |
| [context-ai-front](https://github.com/context-ai-project/context-ai-front) | Frontend (Next.js 16, React 19) |
| [context-ai-shared](https://github.com/context-ai-project/context-ai-shared) | Librería compartida de tipos, DTOs y validadores |

### Librería compartida (`@context-ai-project/shared`)

Los tipos, DTOs y enums compartidos entre el backend y el frontend se publican como paquete npm en **GitHub Packages**: [`@context-ai-project/shared`](https://github.com/orgs/context-ai-project/packages).

---

## 🏗️ Estructura del proyecto

A alto nivel, el proyecto se organiza así:

| Elemento | Contenido |
|----------|------------|
| **Carpeta con este README** (p. ej. `Context.ia/`) | Documentación general: roadmap, casos de uso, plan de implementación, arquitectura y este README. Carpeta `documentation/`. |
| **context-ai-api** | Backend (NestJS): API REST, Clean Architecture, módulos de conocimiento (RAG), interacción (chat), auth (Auth0 + RBAC), usuarios, sectores, estadísticas, invitaciones, notificaciones y cápsulas multimedia. |
| **context-ai-front** | Frontend (Next.js 16, App Router): chat, dashboard, gestión de conocimiento, autenticación con NextAuth.js, internacionalización (i18n). |
| **context-ai-shared** | Librería de tipos, DTOs y validadores compartidos (publicada en GitHub Packages). |

La **estructura detallada del backend** (árbol de `src/`, módulos y capas) está en [context-ai-api/README.md](../context-ai-api/README.md#estructura-del-proyecto). La del frontend, en [context-ai-front/README.md](../context-ai-front/README.md).

---

## 🚀 Setup para Desarrolladores

### Requisitos previos

**Herramientas locales:**
- **Node.js** 22.x (recomendado LTS 22). El proyecto usa características que requieren Node 22; versiones anteriores pueden fallar. Los `package.json` de api y front incluyen `"engines": { "node": ">=22" }` para que pnpm avise si la versión no es la esperada.
- **pnpm** >= 10.x — instalar con `corepack enable` y `corepack prepare pnpm@latest`, o con `npm install -g pnpm`. Si `corepack` no está disponible en tu instalación de Node, usa el método global. Comprueba con `pnpm -v` antes de seguir.
- **Docker** >= 24.x y **Docker Compose v2** (comando `docker compose`, con espacio). Los scripts del backend usan `docker compose`; si en tu sistema solo existe el binario antiguo `docker-compose` (con guion), instala el plugin o crea un alias. En Linux es habitual tener solo `docker compose`.
- Cuenta de **GitHub** (para acceder a GitHub Packages)
- **Google Cloud SDK** ([instalar](https://cloud.google.com/sdk/docs/install)) para ejecutar `gcloud auth application-default login` en local

**Cuentas en servicios externos (deben configurarse antes de tocar código):**
- Cuenta en **Auth0** → [auth0.com](https://auth0.com/) — autenticación de usuarios
- Cuenta en **Pinecone** → [pinecone.io](https://www.pinecone.io/) — vector store para embeddings
- **Google Cloud Platform** → [console.cloud.google.com](https://console.cloud.google.com/) — Vertex AI para LLM y embeddings (Gemini). Autenticación via ADC (`gcloud auth application-default login`)

> **Orden importante**: Configura los servicios externos (Auth0, Pinecone, Google Cloud) y autentica con `gcloud auth application-default login` **antes** de editar los archivos `.env`. Los pasos 3 y 4 del setup requieren esos valores.

**Checklist de prerrequisitos (orden recomendado):**

1. **Auth0**: Tenant creado; API con RBAC y permisos; aplicación **Machine-to-Machine (M2M)** para invitaciones; aplicación **Regular Web Application** para el frontend. Ver [`context-ai-api/docs/AUTH0_SETUP.md`](../context-ai-api/docs/AUTH0_SETUP.md).
2. **Pinecone**: Cuenta e índice creado (nombre `context-ai`, dimensiones 3072, métrica cosine). Ver [`context-ai-api/docs/DATABASE_SETUP.md`](../context-ai-api/docs/DATABASE_SETUP.md).
3. **GCP**: Proyecto con Vertex AI habilitado; en local ejecutar `gcloud auth application-default login`; la cuenta o service account con rol **Vertex AI User**.
4. **INTERNAL_API_KEY**: Generada una sola vez (`openssl rand -hex 32`) y el **mismo valor** en `context-ai-api/.env` y `context-ai-front/.env.local`.
5. **Variables para cápsulas (backend)**: La API arranca con el módulo de cápsulas (audio/vídeo) cargado. Para que el servidor inicie sin error debes rellenar en `context-ai-api/.env`: **ELEVENLABS_API_KEY** ([ElevenLabs](https://elevenlabs.io/app/api-key)), **GCS_BUCKET_CAPSULES** y **GCS_PROJECT_ID** (bucket y proyecto de Google Cloud Storage), y **SHOTSTACK_API_KEY** ([Shotstack](https://dashboard.shotstack.io/)). Ver `context-ai-api/docs/ENVIRONMENT_VARIABLES.md` para detalles.
6. Tras ejecutar las migraciones en el backend, ejecutar **`pnpm seed:rbac`**; sin este paso los endpoints protegidos por RBAC pueden devolver 403 aunque el JWT sea válido.

---

### 0. Generar la clave interna compartida

El backend y el frontend se comunican mediante una clave secreta compartida (`INTERNAL_API_KEY`). Debes generarla **una sola vez** y usarla en ambos servicios:

```bash
openssl rand -hex 32
# Copia el resultado — lo necesitarás en context-ai-api/.env y en context-ai-front/.env.local
```

> **Windows**: Si `openssl` no está en PATH, usa Git Bash o: `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"`

---

### 1. Configurar acceso a GitHub Packages

GitHub Packages requiere autenticación incluso para paquetes públicos. Esto se configura **una sola vez** por máquina:

1. Crea un **Personal Access Token (Classic)** en GitHub:
   - Ve a https://github.com/settings/tokens/new
   - Marca ✅ `read:packages`
   - Genera y copia el token

2. Configura tu `~/.npmrc` global:

```bash
echo "//npm.pkg.github.com/:_authToken=ghp_TU_TOKEN_AQUI" >> ~/.npmrc
echo "@context-ai-project:registry=https://npm.pkg.github.com/" >> ~/.npmrc
```

**PowerShell (Windows)** — Añade el token al npmrc del usuario (ruta típica `$env:USERPROFILE\.npmrc`):
```powershell
"//npm.pkg.github.com/:_authToken=ghp_TU_TOKEN_AQUI" | Out-File -FilePath $env:USERPROFILE\.npmrc -Append
"@context-ai-project:registry=https://npm.pkg.github.com/" | Out-File -FilePath $env:USERPROFILE\.npmrc -Append
```

### 2. Clonar los repositorios

```bash
# Crear directorio del proyecto
mkdir Context.ai && cd Context.ai

# Si tu equipo tiene un repo con este README y la carpeta documentation/, clónalo primero, por ejemplo:
# git clone <URL_DEL_REPO_CONTEXT_IA> Context.ia
# cd Context.ia   # opcional, para seguir la guía desde aquí

# Clonar repositorios de código (desde la raíz del proyecto, p. ej. Context.ai)
git clone https://github.com/context-ai-project/context-ai-api.git
git clone https://github.com/context-ai-project/context-ai-front.git
git clone https://github.com/context-ai-project/context-ai-shared.git  # opcional, solo si modificarás la librería
```

### 3. Configurar y ejecutar el Backend

> Antes de este paso necesitas: credenciales de Auth0 (ver [`context-ai-api/docs/AUTH0_SETUP.md`](../context-ai-api/docs/AUTH0_SETUP.md)), índice de Pinecone creado (ver [`context-ai-api/docs/DATABASE_SETUP.md`](../context-ai-api/docs/DATABASE_SETUP.md)), proyecto GCP con Vertex AI habilitado y autenticación ADC (`gcloud auth application-default login`), y las variables de cápsulas (ElevenLabs, GCS, Shotstack) para que la API arranque.

```bash
cd context-ai-api
pnpm install
cp .env.example .env    # Editar con tus credenciales (Auth0, Pinecone, GCP, INTERNAL_API_KEY, ELEVENLABS, GCS_*, SHOTSTACK_*)
docker compose up -d    # Iniciar PostgreSQL (usa "docker compose" v2; si falla, prueba "docker-compose up -d")
pnpm migration:run      # Ejecutar migraciones (si falla por conexión, espera unos segundos a que Postgres esté "healthy" y repite)
pnpm seed:rbac          # Sembrar roles y permisos (requerido para que RBAC funcione)
./scripts/verify-setup.sh  # Verificar que todo arrancó correctamente (solo Unix/macOS; en Windows ver nota abajo)
pnpm start:dev          # http://localhost:3001 | Swagger: http://localhost:3001/api/docs
```

**Windows:** El script `verify-setup.sh` no se ejecuta. Comprueba a mano: (1) `docker ps` y que el contenedor `context-ai-postgres` esté en ejecución, (2) que el puerto 5433 esté accesible, (3) que la API responda en `http://localhost:3001/api/v1` tras `pnpm start:dev`.

**Husky:** Al hacer `pnpm install` se ejecuta el hook `prepare` y se configuran los Git hooks (Husky). Es normal; si el directorio no es un repo git aún, Husky puede mostrar un aviso y no bloquear la instalación.

### 4. Configurar y ejecutar el Frontend

> La `INTERNAL_API_KEY` del paso 4 debe ser **exactamente la misma** que configuraste en `context-ai-api/.env`. Ambos servicios la comparten para comunicación interna.

```bash
cd context-ai-front
pnpm install
cp env.local.example .env.local  # Editar con tus credenciales (Auth0, INTERNAL_API_KEY)
pnpm dev                     # http://localhost:3000
```

### 5. Desarrollo local de la librería compartida (opcional)

Si necesitas modificar `@context-ai-project/shared` y probar cambios sin publicar:

```bash
# Desde el directorio del proyecto que consume el paquete
cd context-ai-api
pnpm link ../context-ai-shared

# Cuando termines, restaurar la versión publicada
pnpm unlink @context-ai-project/shared
pnpm install
```

---

## 🚀 Funcionalidades Principales

| # | Funcionalidad | Descripción |
|---|----------------|-------------|
| 1 | **Aislamiento por Sectores** | Gestión de espacios de conocimiento por departamento (RRHH, Tech, Ventas) con control de acceso (RBAC). |
| 2 | **Motor RAG Multimodal** | Ingesta y consulta de documentación (PDF, MD) mediante búsqueda semántica avanzada y embeddings en Pinecone. |
| 3 | **Asistente Conversacional** | Chat IA con respuestas contextualizadas (Gemini vía Genkit) y citas a fuentes originales. |
| 4 | **Historial de Conversaciones** | Persistencia de conversaciones y mensajes con contexto multi-turno. |
| 5 | **Dashboard de Métricas** | Visualización de estadísticas de uso y calidad para administradores. |
| 6 | **Autenticación y Autorización** | Login con **Auth0** (OAuth2/JWT), **RBAC** interno (roles admin, manager, user), revocación de tokens y auditoría de eventos de seguridad. |
| 7 | **Invitaciones de usuarios** | Flujo de invitación por correo (Auth0 M2M + user tickets); sin registro público. |
| 8 | **Notificaciones** | Notificaciones in-app (event-driven) para avisos y actualizaciones. |
| 9 | **Cápsulas multimedia (v2)** | Generación de **cápsulas de audio** (TTS con ElevenLabs) y **cápsulas de vídeo** (guion + audio + montaje con Shotstack); almacenamiento en Google Cloud Storage; pipeline asíncrono con Cloud Tasks. |

---

## 🧪 Calidad, Seguridad y CI/CD

* **GitHub Actions**: Automatización de lint, tests, build y security scanning.
* **Testing**: Jest (backend) + Vitest + Playwright (frontend) con cobertura > 80%.
* **Security by Design**: ESLint security plugins, Snyk, CodeQL y OWASP guidelines.
* **Git Hooks**: Husky + lint-staged para validación automática pre-commit y pre-push.
* **Docker**: Contenerización de servicios para desarrollo y producción.

---

## 📚 Documentación

La documentación completa del proyecto se encuentra en el directorio [`documentation/`](./documentation/):

| Documento | Descripción |
|-----------|-------------|
| [000-story.md](./documentation/000-story.md) | Historia del proyecto |
| [001-requirements.md](./documentation/001-requirements.md) | Requisitos funcionales y no funcionales |
| [002-use-cases.md](./documentation/002-use-cases.md) | Casos de uso |
| [005-technical-architecture.md](./documentation/005-technical-architecture.md) | Arquitectura técnica |
| [005b-mvp-definition.md](./documentation/005b-mvp-definition.md) | Definición del MVP |
| [006-data-model.md](./documentation/006-data-model.md) | Modelo de datos |
| [007-api-contract.md](./documentation/007-api-contract.md) | Contrato de API |
| [008-roadmap.md](./documentation/008-roadmap.md) | Roadmap del proyecto |
| [009-implementation-plan.md](./documentation/009-implementation-plan.md) | Plan de implementación |
| [015-deployment-cloud-architecture.md](./documentation/015-deployment-cloud-architecture.md) | Arquitectura de deployment en cloud |
| [016-rbac-permissions-matrix.md](./documentation/016-rbac-permissions-matrix.md) | Matriz de permisos RBAC |
| [README-audit-cumplimiento.md](./documentation/README-audit-cumplimiento.md) | Auditoría de cumplimiento del README (qué faltaba y qué se añadió) |

### Documentación técnica por repositorio

| Documento | Descripción |
|-----------|-------------|
| [context-ai-api/docs/AUTH0_SETUP.md](../context-ai-api/docs/AUTH0_SETUP.md) | Guía completa de configuración de Auth0 (backend + frontend) |
| [context-ai-api/docs/DATABASE_SETUP.md](../context-ai-api/docs/DATABASE_SETUP.md) | Setup de PostgreSQL y Pinecone (creación de índice) |
| [context-ai-api/docs/ARCHITECTURE.md](../context-ai-api/docs/ARCHITECTURE.md) | Arquitectura interna del backend (Clean Architecture) |
| [context-ai-api/docs/ENVIRONMENT_VARIABLES.md](../context-ai-api/docs/ENVIRONMENT_VARIABLES.md) | Referencia completa de variables de entorno del backend |
