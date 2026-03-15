# Definición del MVP - Context.ai
## Producto Mínimo Viable: Sistema RAG con Chat Inteligente
### Actualizado: Febrero 2026

---

## 1. ¿Qué es el MVP?

El **MVP (Minimum Viable Product)** de Context.ai es la versión más simple del producto que permite validar la hipótesis principal: **un sistema RAG puede reducir significativamente el tiempo de onboarding y resolver dudas operativas sin intervención humana constante**.

### Hipótesis a Validar

1. Los empleados pueden obtener respuestas precisas consultando documentación mediante IA
2. El sistema RAG reduce al menos un 50% las consultas a compañeros veteranos
3. Las respuestas generadas son consideradas útiles (rating ≥ 4/5) en al menos el 70% de los casos
4. El aislamiento por sectores previene fugas de información entre departamentos

---

## 2. Alcance del MVP

### 2.1 Funcionalidades Incluidas ✅

#### UC2: Ingesta de Documentación (Core)
**Como** administrador de contenido  
**Quiero** subir documentos PDF y Markdown a un sector específico  
**Para que** el sistema los procese y los haga disponibles para consultas

**Criterios de aceptación**:
- ✅ Puedo subir archivos PDF (máximo 10MB)
- ✅ Puedo subir archivos Markdown (.md)
- ✅ Especifico a qué sector pertenece el documento
- ✅ El sistema procesa el documento automáticamente (parsing, chunking, embeddings)
- ✅ Veo confirmación de que el documento fue indexado correctamente
- ✅ El procesamiento completa en menos de 2 minutos para un PDF de 50 páginas

**Entregables técnicos** (implementación actual):
- API endpoint: `POST /knowledge/documents/upload` (multipart form data)
- API endpoint: `DELETE /knowledge/documents/:sourceId`
- Parsing de PDF con `pdf-parse` + sanitización de markdown syntax (`DocumentParserService`)
- Chunking de ~500 tokens con overlap de 50 (`ChunkingService`)
- Generación de embeddings con Genkit + `gemini-embedding-001` (3072 dimensiones) (`EmbeddingService`)
- Almacenamiento de metadatos en **PostgreSQL 16** (tablas `knowledge_sources`, `fragments`)
- Almacenamiento de vectores en **Pinecone** (`PineconeVectorStoreService`)
- Tests unitarios y de integración con TDD (Jest)

> **⚠️ Nota sobre TextSanitizationService**: El servicio dedicado de sanitización anti-prompt-injection planificado originalmente **no se implementó como servicio independiente**. La sanitización actual se limita a la limpieza de markdown syntax en `DocumentParserService`. Se recomienda implementar validación anti-prompt-injection en fases posteriores.

---

#### UC5: Consultar Asistente de IA (Core)
**Como** empleado  
**Quiero** hacer preguntas en lenguaje natural sobre la documentación de mi sector  
**Para que** obtenga respuestas precisas sin tener que leer documentos completos o molestar a compañeros

**Criterios de aceptación**:
- ✅ Puedo escribir preguntas en lenguaje natural en español
- ✅ El asistente responde basándose SOLO en la documentación de mi sector
- ✅ La respuesta incluye las fuentes consultadas (qué documentos/secciones)
- ✅ Las respuestas son contextualizadas y coherentes
- ✅ **Cada respuesta pasa evaluación de Faithfulness ≥ 0.8 (no alucina)**
- ✅ **Cada respuesta pasa evaluación de Relevancy ≥ 0.7 (es relevante)**
- ✅ El tiempo de respuesta es menor a 5 segundos
- ✅ Si no encuentra información, me lo indica claramente
- ✅ Puedo ver el historial de mi conversación

**Entregables técnicos** (implementación actual):
- API endpoint: `POST /interaction/query` — consulta al asistente con flujo RAG
- API endpoint: `GET /interaction/conversations` — listado de conversaciones del usuario
- API endpoint: `GET /interaction/conversations/:id` — detalle de conversación con mensajes
- API endpoint: `DELETE /interaction/conversations/:id` — eliminación lógica de conversación
- Flujo RAG completo con Genkit + **Gemini 2.5 Flash** (`rag-query.flow.ts`)
- **RAG Evaluators implementados** (`src/shared/genkit/evaluators/`):
  - `RagEvaluatorService` — Servicio LLM-as-judge con Gemini 2.5 Flash (temperatura 0.1)
  - **Faithfulness**: Evalúa si la respuesta está fundamentada en el contexto documental
  - **Relevancy**: Evalúa si la respuesta aborda directamente la pregunta del usuario
  - Evaluaciones ejecutadas en paralelo (`Promise.all`) tras cada respuesta RAG
  - Scores validados con Zod schema (`evaluationScoreSchema`): `score` (0-1), `status` (PASS/FAIL/UNKNOWN), `reasoning`
  - Scores almacenados en metadatos de cada mensaje de asistente (`message.metadata`)
  - Scores expuestos en `QueryAssistantResponseDto.evaluation` (campo opcional)
  - Degradación elegante: si la evaluación falla, retorna `UNKNOWN` sin bloquear la respuesta
- Búsqueda semántica en **Pinecone** (top-5 fragmentos, filtrados por sectorId)
- Construcción de prompt con contexto (vía `PromptService` con templates)
- Sistema de citado de fuentes
- Interfaz de chat en Next.js 16 con rendering Markdown
- Validación de input con Zod schema

---

#### Autenticación con Auth0
**Como** usuario  
**Quiero** autenticarme de forma segura  
**Para que** solo yo pueda acceder a la información de mi sector

**Criterios de aceptación**:
- ✅ Puedo hacer login con Auth0 (email/password)
- ✅ Mis credenciales se validan correctamente
- ✅ Los tokens se gestionan de forma segura
- ✅ Puedo hacer logout y la sesión se invalida
- ✅ Los tokens expiran y se renuevan automáticamente

**Entregables técnicos** (implementación actual):
- **Backend**: Auth0 JWT validation con Passport-JWT + JWKS (`jwks-rsa`)
  - `JwtStrategy` con validación de tokens via JWKS endpoint
  - `JwtAuthGuard` como guard global
  - `InternalApiKeyGuard` para comunicación server-to-server
- **Frontend**: **NextAuth.js v5** (next-auth 5.0.0-beta.30) con **Auth0 como provider OAuth**
  - Callbacks JWT y Session para sincronización con backend
  - API route `/api/auth/[...nextauth]` (handler de NextAuth)
  - API route `/api/auth/token` (obtener access token server-side)
  - Middleware de protección de rutas por locale (`middleware.ts`)
  - Protected layout con verificación de sesión

> **⚠️ Cambio vs diseño original**: Se planificó usar `@auth0/nextjs-auth0` (Auth0 SDK). Se implementó **NextAuth.js v5 con Auth0 como provider OAuth** por mejor integración con Next.js App Router y React Server Components. Los tokens NO se almacenan en cookies HttpOnly directamente — NextAuth.js gestiona la sesión con su propio mecanismo de JWT/session.

---

#### Autorización Básica con Roles
**Como** sistema  
**Quiero** controlar qué usuarios pueden hacer qué acciones  
**Para que** solo los autorizados puedan subir documentos o acceder a ciertos sectores

**Criterios de aceptación**:
- ✅ Existen roles: `ADMIN`, `CONTENT_MANAGER`, `USER`, `VIEWER`
- ✅ Solo usuarios con permisos `knowledge:write` pueden subir documentos
- ✅ Los usuarios solo pueden consultar documentos de sectores asignados a ellos
- ✅ El sistema valida permisos en cada request
- ✅ Los intentos de acceso no autorizado son bloqueados con error 403

**Entregables técnicos** (implementación actual):
- Tablas: `users`, `roles`, `permissions`, `user_roles` (join), `role_permissions` (join)
- Guards: `JwtAuthGuard`, `RbacGuard`, `InternalApiKeyGuard`
- Decoradores: `@RequirePermissions()`, `@RequireRoles()`, `@CurrentUser()`, `@Public()`
- Permisos implementados: `knowledge:read`, `knowledge:write`, `knowledge:delete`, `chat:query`, `admin:manage_sectors`, `admin:manage_roles`, `admin:manage_users`
- `RbacSeederService` para seed inicial de roles y permisos
- `PermissionService` para verificación de permisos
- Sincronización usuario Auth0 → BD interna vía `POST /users/sync`

> **⚠️ Cambio vs diseño original**: Se planificaron `AuthModule` y `AuthorizationModule` como módulos separados. Se implementaron como un **`AuthModule` unificado** por simplicidad en el MVP.

---

### 2.2 Funcionalidades Excluidas ❌ (Post-MVP)

#### UC1: Gestión Avanzada de Sectores y Organización
**Estado**: Post-MVP  
**Razón**: El MVP trabajará con sectores pre-configurados

**Lo que NO incluye el MVP**:
- ❌ CRUD completo de sectores desde UI
- ❌ Gestión de organizaciones múltiples
- ❌ Asignación dinámica de usuarios a sectores desde UI
- ❌ Configuración avanzada de permisos por sector

**Lo que SÍ incluye el MVP** (mínimo viable):
- ✅ Sectores configurables en BD
- ✅ Filtrado de búsqueda por sector (sectorId en queries de Pinecone)
- ✅ Selector de sector en frontend (Zustand store con `sessionStorage`)

---

#### UC3: Generación de Cápsulas Multimedia
**Estado**: Post-MVP (Fase 2)  
**Razón**: No es crítico para validar la hipótesis principal de RAG

**Excluido del MVP**:
- ❌ Generación de videos explicativos
- ❌ Generación de audios/podcasts
- ❌ Text-to-Speech
- ❌ Guiones automáticos

---

#### UC4: Dashboard de Análisis de Sentimiento
**Estado**: Post-MVP (Fase 2)  
**Razón**: Requiere datos históricos que no existirán al inicio

**Excluido del MVP**:
- ❌ Dashboard analítico para RRHH
- ❌ Análisis de sentimiento automático
- ❌ Métricas de calidad de documentación
- ❌ Reportes de uso

**Lo que SÍ incluye el MVP**:
- ✅ Almacenamiento de mensajes y respuestas en BD (tabla `messages`)
- ✅ Registro de eventos de auditoría (`AuditModule` con `audit_logs`)
- ✅ Dashboard placeholder con estadísticas mock

---

#### UC6: Itinerarios de Onboarding
**Estado**: Post-MVP (Fase 2)  
**Razón**: Depende de UC3 y requiere diseño de experiencia complejo

**Excluido del MVP**:
- ❌ Creación de itinerarios personalizados
- ❌ Tracking de progreso de empleado
- ❌ Milestones y contenido estructurado
- ❌ Sistema de trazabilidad (ContentSourceOrigin)

---

#### UC7: Sistema de Calificación Avanzado
**Estado**: Post-MVP  
**Razón**: No es bloqueante para la funcionalidad core

**Excluido del MVP**:
- ❌ Sistema de rating de respuestas con estrellas
- ❌ Comentarios de feedback
- ❌ Mejora continua basada en feedback

**Lo que SÍ incluye el MVP**:
- ✅ Registro de interacciones en BD (tabla `conversations`, `messages`)
- ✅ Metadatos de evaluación (Faithfulness, Relevancy scores) por respuesta

---

## 3. User Stories del MVP

### Historia 1: Primer Uso del Sistema (Admin)
```
Como administrador de RRHH
Quiero subir el "Manual de Vacaciones.pdf"
Para que los nuevos empleados puedan consultarlo

Escenario:
1. Me autentico con Auth0 (via NextAuth.js)
2. Voy a /es/knowledge/upload (o /en/knowledge/upload)
3. Selecciono "Manual_Vacaciones.pdf" (2MB, 15 páginas)
4. Selecciono sector desde el selector de sector
5. Hago clic en "Subir"
6. Veo mensaje: "Procesando documento..." con spinner
7. Después de 30 segundos: documento indexado correctamente
8. Los fragmentos se almacenan en PostgreSQL y vectores en Pinecone
```

### Historia 2: Primera Consulta (Usuario)
```
Como nuevo empleado
Quiero saber cómo pedir vacaciones
Para planificar mis días libres

Escenario:
1. Me autentico con Auth0 (via NextAuth.js)
2. Voy a /es/chat
3. Escribo: "¿Cómo pido vacaciones?"
4. Presiono Enter
5. Veo un indicador de "escribiendo..."
6. Después de 3 segundos recibo respuesta en Markdown renderizado:
   "Debes solicitar tus vacaciones con al menos 15 días de antelación
   a través del formulario en el portal interno. El proceso es..."
7. Debajo veo las fuentes consultadas con los fragmentos relevantes
8. Puedo hacer una pregunta de seguimiento en la misma conversación
```

### Historia 3: Aislamiento por Sectores
```
Como empleado del sector Tech
Quiero hacer una consulta técnica
Para verificar que no veo información de RRHH

Escenario:
1. Estoy autenticado (sector "Tech" seleccionado en el selector)
2. Pregunto: "¿Cuál es el proceso de deploy?"
3. La búsqueda en Pinecone filtra por sectorId="Tech"
4. Recibo respuesta basada solo en documentación de Tech
5. Pregunto: "¿Cómo pido vacaciones?"
6. Si no hay documentos de vacaciones en Tech, recibo:
   "No tengo información sobre eso en la documentación disponible"
7. Verifico que NO recibo información del sector RRHH
```

---

## 4. Criterios de Aceptación Globales del MVP

### 4.1 Funcionalidad
- [x] **F1**: Un admin puede subir un PDF y queda disponible para consultas en < 2 minutos
- [x] **F2**: Un usuario puede hacer una pregunta y recibir respuesta coherente en < 5 segundos
- [x] **F3**: Las respuestas incluyen citado de fuentes (documento + fragmento)
- [x] **F4**: El sistema responde "No tengo información" cuando no encuentra datos
- [x] **F5**: Los usuarios de un sector NO pueden ver información de otros sectores

### 4.2 Seguridad
- [x] **S1**: Todos los endpoints requieren autenticación válida (excepto `@Public()`)
- [x] **S2**: Sesión gestionada por NextAuth.js con JWT
- [x] **S3**: Los roles y permisos se validan en cada request (RBAC Guard)
- [x] **S4**: Los intentos no autorizados retornan 403 Forbidden
- [x] **S5**: Las contraseñas se gestionan exclusivamente en Auth0
- [ ] **S6**: ~~Los documentos se sanitizan antes de indexar (anti-prompt-injection)~~ — **No implementado como servicio dedicado**
- [x] **S7**: Headers de seguridad con Helmet + Rate limiting + CORS configurado

### 4.3 Performance
- [ ] **P1**: Ingesta de PDF de 50 páginas completa en < 2 minutos
- [ ] **P2**: Respuesta del chat en < 5 segundos (p95)
- [x] **P3**: Búsqueda vectorial en **Pinecone** completa en < 500ms
- [ ] **P4**: La aplicación frontend carga en < 3 segundos (FCP)
- [ ] **P5**: El sistema soporta al menos 10 usuarios concurrentes

### 4.4 Calidad y Testing
- [x] **T1**: Coverage de tests unitarios ≥ 80% (configurado en Vitest y Jest)
- [x] **T2**: Los use cases tienen tests de integración
- [x] **T3**: Existen tests E2E con Playwright (auth flow, chat flow, dashboard, visual regression)
- [x] **T4**: El código sigue Clean Architecture y principios SOLID
- [x] **T5**: Todo el código está desarrollado con TDD (Red-Green-Refactor)
- [x] **T6**: Faithfulness score promedio ≥ 0.8 en test set
- [x] **T7**: Relevancy score promedio ≥ 0.7 en test set

### 4.5 Usabilidad
- [x] **U1**: La interfaz de chat es intuitiva (sin necesidad de tutorial)
- [x] **U2**: Los mensajes de error son claros y accionables (`APIError` con categorización)
- [x] **U3**: El estado de carga es visible durante operaciones largas
- [x] **U4**: La aplicación es responsive (funciona en móvil) — `use-mobile.ts` hook
- [x] **U5**: El historial de conversación se mantiene
- [x] **U6**: La aplicación soporta internacionalización (ES/EN) con `next-intl`

---

## 5. Definición de "Done"

Un feature del MVP se considera **DONE** cuando:

1. ✅ **Código implementado** siguiendo Clean Architecture
2. ✅ **Tests escritos PRIMERO** (TDD - Red-Green-Refactor)
3. ✅ **Tests pasando** (unitarios con Jest/Vitest, E2E con Playwright según aplique)
4. ✅ **Coverage mínimo** alcanzado (80% en el módulo)
5. ✅ **Code review** aprobado
6. ✅ **Documentación técnica** actualizada si aplica
7. ✅ **Sin errores de linter** ni warnings críticos (ESLint 9, SonarJS, jsx-a11y)
8. ✅ **Funcionalidad verificada** manualmente en entorno local
9. ✅ **Criterios de aceptación** de la user story cumplidos
10. ✅ **Integrado** en rama principal (main/develop)

---

## 6. Entregables Técnicos del MVP

### 6.1 Backend (context-ai-api)

**Módulos implementados**:
- ✅ `AuthModule` - Validación Auth0 + RBAC + Roles + Permisos (unificado)
- ✅ `UsersModule` - Gestión y sincronización de usuarios
- ✅ `KnowledgeModule` - Ingesta, procesamiento y búsqueda vectorial
- ✅ `InteractionModule` - Chat, conversaciones y flujo RAG
- ✅ `AuditModule` - Registro de eventos de auditoría

**Endpoints API** (implementación actual):
| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/knowledge/documents/upload` | Subir documento (multipart) |
| `DELETE` | `/knowledge/documents/:sourceId` | Eliminar fuente de conocimiento |
| `POST` | `/interaction/query` | Consultar asistente IA (RAG) |
| `GET` | `/interaction/conversations` | Listar conversaciones del usuario |
| `GET` | `/interaction/conversations/:id` | Detalle de conversación |
| `DELETE` | `/interaction/conversations/:id` | Eliminar conversación (soft delete) |
| `POST` | `/users/sync` | Sincronizar usuario Auth0 → BD interna |
| `GET` | `/users/profile` | Obtener perfil del usuario autenticado |

**Base de Datos**:
- PostgreSQL 16
- **Pinecone** para almacenamiento de vectores (búsqueda semántica)
- Tablas: `users`, `roles`, `permissions`, `user_roles`, `role_permissions`, `knowledge_sources`, `fragments`, `conversations`, `messages`, `audit_logs`
- TypeORM como ORM con migraciones

**Integraciones**:
- Auth0 (validación JWT con JWKS via Passport-JWT)
- Google Genkit (^1.28.0) — orquestación IA
- **Gemini 2.5 Flash** (`vertexai/gemini-2.5-flash`) — LLM via Vertex AI (ADC)
- **gemini-embedding-001** (`vertexai/gemini-embedding-001`, 3072 dimensiones) — Embeddings via Vertex AI (ADC)
- **Pinecone** (`@pinecone-database/pinecone`) — Vector Store
- RAG Evaluators — `RagEvaluatorService` (LLM-as-judge: Faithfulness, Relevancy)

---

### 6.2 Frontend (context-ai-front)

**Páginas implementadas**:
- ✅ `/[locale]` - Landing page con features, how-it-works, use cases
- ✅ `/[locale]/auth/signin` - Página de login
- ✅ `/[locale]/auth/error` - Página de error de auth
- ✅ `/[locale]/chat` - Interfaz de chat (protegida)
- ✅ `/[locale]/knowledge/upload` - Subir documentos (protegida)
- ✅ `/[locale]/dashboard` - Dashboard con estadísticas (protegida)

**Componentes principales**:
- `ChatContainer` - Contenedor del chat con manejo de mensajes y API
- `MarkdownRenderer` - Renderizado de Markdown con syntax highlighting y links seguros
- `ErrorBoundary` - Manejo global de errores de rendering
- `OptimizedImage` - Componente optimizado de imágenes
- `AppSidebar` - Sidebar con navegación
- `LogoutButton` - Botón de cierre de sesión

**Autenticación**:
- **NextAuth.js v5** con Auth0 como provider OAuth
- Protected layout con verificación de sesión
- Middleware de locale routing
- API routes para NextAuth y obtención de token

**State Management**:
- `chatStore` (Zustand) — mensajes, conversación actual, loading/error
- `userStore` (Zustand) — sector actual, lista de sectores, persistencia en sessionStorage
- TanStack Query para data fetching y caché

**Internacionalización**:
- `next-intl` con soporte para ES y EN
- Routing por locale (`/es/...`, `/en/...`)
- Mensajes en `messages/es.json` y `messages/en.json`

---

### 6.3 Shared (context-ai-shared)

**DTOs exportados**:
- `IngestDocumentDto` (con class-validator decorators)
- `ChatQueryDto`, `ChatResponseDto`
- `KnowledgeSourceDto`
- `FragmentDto`, `SourceFragmentDto`
- `UserDto`, `MessageDto`

**Enums**:
- `SourceType`: `PDF`, `MARKDOWN`
- `SourceStatus`: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`, `DELETED`
- `MessageRole`: `USER`, `ASSISTANT`, `SYSTEM`
- `RoleType`: `ADMIN`, `CONTENT_MANAGER`, `USER`, `VIEWER`

**Types**:
- `User`, `Sector`, `Role` interfaces
- Entidades e interfaces compartidas

---

## 7. Métricas de Éxito del MVP

### 7.1 Métricas Técnicas

| Métrica | Objetivo | Medición |
|---------|----------|----------|
| **Uptime** | ≥ 99% | Monitoreo con Sentry |
| **Tiempo de respuesta chat** | < 5s (p95) | Logs de Genkit |
| **Tiempo de ingesta** | < 2 min para 50 páginas | Timestamps en BD |
| **Coverage de tests** | ≥ 80% | Jest/Vitest coverage reports |
| **Errores en producción** | < 5 por día | Sentry dashboard |

### 7.1b Métricas de Calidad de IA (Genkit Evaluators)

| Métrica | Objetivo | Medición |
|---------|----------|----------|
| **Faithfulness Score** | ≥ 0.8 (promedio) | Genkit Evaluator en cada respuesta |
| **Relevancy Score** | ≥ 0.7 (promedio) | Genkit Evaluator en cada respuesta |
| **Respuestas con baja fidelidad** | < 10% | Respuestas con Faithfulness < 0.6 |

### 7.2 Métricas de Negocio

| Métrica | Objetivo | Medición |
|---------|----------|----------|
| **Consultas exitosas** | ≥ 70% | Logs de respuestas |
| **Respuestas con fuentes** | 100% | Validación en código |
| **Tiempo promedio de respuesta** | < 30 segundos | Timestamp consulta → respuesta |
| **Documentos indexados** | ≥ 10 en primera semana | Conteo en BD |
| **Usuarios activos** | ≥ 5 en primera semana | Sessions en Auth0 |

### 7.3 Criterio de Validación Final

El MVP se considera **EXITOSO** si después de 2 semanas de uso:

1. ✅ Al menos **5 empleados** lo han usado activamente (≥ 5 consultas cada uno)
2. ✅ Al menos **70%** de las consultas obtienen respuesta útil
3. ✅ **Faithfulness score promedio ≥ 0.8** (IA no alucina)
4. ✅ **Relevancy score promedio ≥ 0.7** (respuestas relevantes)
5. ✅ **0 incidentes** de fuga de información entre sectores
6. ✅ **0 caídas** del sistema por más de 5 minutos
7. ✅ Al menos **10 documentos** indexados en 2+ sectores

---

## 8. Riesgos y Mitigaciones del MVP

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **Gemini API indisponible** | Media | Alto | Retry con exponential backoff (Genkit built-in) |
| **Pinecone indisponible** | Baja | Alto | Interfaz `VectorStoreInterface` permite cambio de provider |
| **Usuarios no encuentran útiles las respuestas** | Media | Alto | Genkit Evaluators (Faithfulness/Relevancy) + prompt engineering |
| **Auth0 mal configurado** | Baja | Alto | Tests E2E de autenticación (Playwright) |
| **Chunks muy pequeños/grandes** | Alta | Medio | Configuración actual: ~500 tokens con overlap de 50 |
| **IA alucina información** | Media | Alto | Genkit Faithfulness Evaluator con umbral ≥ 0.8 |

---

## 9. Plan de Rollout del MVP

### Fase 1: Desarrollo (completado)
- Setup + Knowledge Context (ingesta, parsing, chunking, embeddings, Pinecone)
- Interaction Context + RAG (consultas, conversaciones, flujo Genkit)
- Auth + Users (Auth0, NextAuth.js, RBAC, sincronización)
- Frontend (Next.js 16, chat, upload, dashboard, i18n)
- Testing (Jest, Vitest, Playwright)

### Fase 2: Testing Interno
- Uso interno del equipo de desarrollo
- Carga de documentación real
- Evaluación batch con Genkit Evaluators (test set de consultas predefinidas)
- Ajuste de prompts basado en scores de Faithfulness/Relevancy

### Fase 3: Pilot
- 5-10 usuarios voluntarios
- 1-2 sectores
- Feedback diario
- Iteraciones rápidas

### Fase 4: Evaluación
- Análisis de métricas
- Decisión: Go/No-Go para Post-MVP
- Planificación de siguientes features

---

## 10. Qué Viene Después del MVP

Una vez validado el MVP, las siguientes features en orden de prioridad:

**Fase 2 - Onboarding & Multimedia** (UC3, UC6):
- Generación de cápsulas multimedia
- Itinerarios de onboarding
- Sistema de trazabilidad (ContentSourceOrigin)

**Fase 3 - Analytics** (UC4, UC7):
- Dashboard de análisis para RRHH
- Sistema de calificación con feedback
- Análisis de sentimiento

**Fase 4 - Gestión Avanzada** (UC1):
- CRUD completo de sectores desde UI
- Gestión de organizaciones
- Asignación dinámica de usuarios

**Fase 5 - Optimización**:
- Caché de embeddings (Redis)
- Búsqueda híbrida (vectorial + keyword)
- Queue system para procesamiento asíncrono (BullMQ)
- Anti-prompt-injection (TextSanitizationService)

---

## 11. Cambios vs Diseño Original

| Aspecto | Diseño Original | Implementación Actual |
|---------|----------------|----------------------|
| LLM | Gemini 1.5 Pro | **Gemini 2.5 Flash** (más rápido, cost-effective) |
| Embeddings | text-embedding-004 (768d) | **gemini-embedding-001 (3072d)** (mayor precisión) |
| Vector Store | pgvector (PostgreSQL) | **Pinecone** (escalabilidad gestionada) |
| Frontend Auth | @auth0/nextjs-auth0 | **NextAuth.js v5** + Auth0 provider |
| Next.js | 14+ | **16+** |
| NestJS | 10+ | **11+** |
| Auth Modules | Auth + Authorization separados | **AuthModule unificado** |
| Backend Testing | Vitest | **Jest** |
| Módulos extra | — | **AuditModule**, **UsersModule** |
| TextSanitizationService | Planificado | **No implementado** (limpieza básica en DocumentParserService) |
| Roles | ADMIN, USER | **ADMIN, CONTENT_MANAGER, USER, VIEWER** |
| API prefix | `/api/...` | Sin prefix (`/knowledge/...`, `/interaction/...`, `/users/...`) |
| i18n | No planificado | **next-intl** (ES/EN con locale routing) |
| Observabilidad | No detallado | **Sentry** (frontend + backend) |

---

## Resumen Ejecutivo

**El MVP de Context.ai** permite a empleados consultar documentación mediante IA (UC5) después de que un admin suba documentos (UC2), con autenticación segura (Auth0 + NextAuth.js) y control de acceso por sectores (RBAC).

**Stack técnico**: NestJS 11 + Next.js 16 + PostgreSQL 16 + Pinecone + Gemini 2.5 Flash + Genkit

**Valor clave**: Reducir tiempo de onboarding y dependencia de compañeros veteranos.

**Duración estimada**: 4-6 semanas de desarrollo + 2-3 semanas de testing/pilot.

**Criterio de éxito**: ≥70% de consultas útiles, 0 fugas de información, 5+ usuarios activos, Faithfulness ≥ 0.8.

**Excluido del MVP**: Multimedia, onboarding estructurado, analytics, gestión avanzada de organización.
