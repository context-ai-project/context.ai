---
name: "Release v1.3 — Issue Tracker Summary"
description: "Resumen detallado para seguimiento del release v1.3 en Jira/GitHub Issues"
version: "1.3.0"
date: "2026-02-18"
related_docs:
  - "019-release-v1.3.md"
  - "020-future-improvements.md"
---

# Release v1.3.0 — CI/CD, Invitaciones Auth0, Notificaciones y Respuestas Estructuradas

**Versión:** 1.3.0 · **Fecha:** 17 feb 2026 · **Repos:** `context-ai-api` + `context-ai-front` · **Estimación total:** ~35h
**Documentación técnica:** `019-release-v1.3.md` · `020-future-improvements.md`

---

## Feature 1: CI/CD — Deploy Automático de API a Cloud Run

**Estimación:** 4h · **Repo:** `context-ai-api` · **Prioridad:** Alta

### Descripción

Integración de GitHub Actions con Google Cloud Run para automatizar el despliegue de la API. Cada push a `main` ejecuta el pipeline completo.

### Pipeline

1. **Job Test**: Checkout → Setup Node 22 + pnpm → Lint → Build → Tests (con PostgreSQL 16 como servicio)
2. **Job Deploy** (solo si test pasa): Auth Google Cloud (Workload Identity Federation/OIDC) → Docker build multi-stage → Push a Artifact Registry → Deploy Cloud Run → Smoke test (health endpoint con retries)

### Detalles técnicos

- **Configuración Cloud Run:** Port 3001 · 512Mi RAM · 1 vCPU · min 0 / max 3 instancias · timeout 300s
- **Seguridad:** Sin service account keys; usa Workload Identity Federation. Secrets inyectados vía Google Secret Manager
- **Archivo:** `context-ai-api/.github/workflows/deploy-production.yml`

### Acceptance Criteria

- [ ] Push a main dispara el workflow automáticamente
- [ ] Tests y lint pasan antes del deploy
- [ ] Imagen Docker se construye y pushea a Artifact Registry
- [ ] Cloud Run despliega la nueva versión con secrets de Secret Manager
- [ ] Smoke test valida que `/health` responde 200
- [ ] Setup completo de Workload Identity Federation en GCP

---

## Feature 2: Sistema de Invitaciones + Notificaciones In-App

**Estimación:** 18h · **Repos:** `context-ai-api` + `context-ai-front` · **Prioridad:** Alta

### Descripción

Sistema completo de invitación de usuarios desde el panel admin. El admin selecciona email, nombre, rol y sectores → la API crea la cuenta en Auth0 vía Management API → el usuario recibe un email de password-reset de Auth0 → al loguearse por primera vez se le asignan automáticamente el rol y sectores. Signup público deshabilitado en Auth0. Incluye sistema de notificaciones in-app basado en eventos reutilizable para v2.

### Subsistemas

| Componente | Stack | Descripción |
|-----------|-------|-------------|
| **Invitaciones API** | NestJS + TypeORM | Módulo `invitations/` con service, controller, DTOs, repository |
| **Auth0 Management** | `auth0` SDK (M2M) | Creación de usuarios + tickets de password-change |
| **Notificaciones in-app** | `@nestjs/event-emitter` | Event bus + tabla `notifications` + listeners para `invitation.created`, `invitation.accepted`, `user.activated` |
| **Frontend — Invitación** | React + shadcn/ui | `InviteUserDialog.tsx` con multi-select de sectores. Si rol es admin, sectores se auto-seleccionan |
| **Frontend — Campana** | React + polling | `NotificationBell.tsx` + `NotificationDropdown.tsx` en el header. Polling cada 30s |

### Flujo completo

```
Admin (Invite User) → POST /admin/invitations {email, name, role, sectorIds[]}
  → Valida email no duplicado + sectores existentes
  → Crea usuario en Auth0 (Management API)
  → Genera password-change ticket (Auth0 envía email)
  → Persiste invitación (status: pending) + sectores en invitation_sectors
  → Emite evento invitation.created → Notificación in-app para admin

Usuario recibe email → Configura contraseña → Login
  → POST /users/sync detecta invitación pendiente
  → Marca invitación como accepted
  → Asigna rol + sectores de la invitación al user (user_sectors)
  → Emite evento user.activated → Notificación para todos los admins
```

### Impacto en base de datos

- **Nuevas tablas:** `invitations`, `invitation_sectors`, `notifications` (6 índices)
- **Migración:** `1740000000000-CreateInvitationsAndNotificationsTables.ts`

### Nuevas variables de entorno

| Variable | Tipo |
|----------|------|
| `AUTH0_MGMT_CLIENT_ID` | Secret Manager |
| `AUTH0_MGMT_CLIENT_SECRET` | Secret Manager |
| `AUTH0_MGMT_DOMAIN` | env var |
| `AUTH0_DB_CONNECTION` | env var |

### Dependencias nuevas

```bash
pnpm add auth0 @nestjs/event-emitter
pnpm add -D @types/auth0
```

### Acceptance Criteria

**Invitaciones:**

- [ ] Admin invita usuario con email, nombre, rol y sectores (multi-select)
- [ ] Si rol es admin, sectores se auto-seleccionan (acceso completo)
- [ ] Cuenta creada en Auth0 vía Management API
- [ ] Usuario recibe email de password-reset de Auth0
- [ ] Invitación registrada como `pending` con sectores en `invitation_sectors`
- [ ] Al primer login, invitación pasa a `accepted` → rol y sectores asignados automáticamente
- [ ] No se permiten invitaciones duplicadas para el mismo email
- [ ] Invitaciones expiran a los 7 días
- [ ] Admin puede revocar invitaciones pendientes
- [ ] Signup público deshabilitado en Auth0

**Notificaciones:**

- [ ] Notificación in-app al enviar invitación y cuando usuario acepta
- [ ] Campana con badge de no leídas en el header
- [ ] Dropdown con notificaciones recientes
- [ ] Marcar como leída (individual y todas)
- [ ] Eventos se procesan async (no bloquean flujo principal)
- [ ] Tests unitarios para InvitationService, NotificationService y listeners

---

## Feature 3: Mejora de Respuesta Fallback (Sin Información)

**Estimación:** 3h · **Repo:** `context-ai-api` + `context-ai-front` · **Prioridad:** Media

### Descripción

Actualmente cuando la IA no encuentra información relevante retorna un string estático hardcodeado en inglés. Se reemplaza por un **fallback inteligente** que pasa por el LLM con un prompt especializado para generar una respuesta empática, contextual y en el mismo idioma de la pregunta. Se agrega campo `responseType` para que el frontend distinga respuestas con/sin contexto.

### Problemas actuales

1. **Estático** — No se adapta al idioma del usuario ni al contexto de la pregunta
2. **Genérico** — No indica qué documentos se consultaron ni sugiere alternativas
3. **No aprovecha la IA** — El fallback es un string, no pasa por el LLM
4. **Sin metadata** — El frontend no puede distinguir "sin información" de respuesta normal

### Solución

- Nuevo prompt `buildFallbackPrompt(query, sectorName)` que genera respuesta contextual vía LLM
- Nuevo enum `RagResponseType`: `answer` | `no_context` | `error`
- Campo `responseType` en el output schema para renderizado diferenciado en frontend
- Graceful degradation: si el LLM falla en el fallback, cae al string estático actual

### Archivos a modificar

| Archivo | Cambio |
|---------|--------|
| `rag-query.flow.ts` | Nuevo prompt de fallback + responseType + llamada al LLM |
| `query-assistant.dto.ts` | Agregar `responseType` al DTO |
| `interaction-dto.mapper.ts` | Mapear responseType |
| `query-assistant.use-case.ts` | Propagar responseType |
| Frontend: MessageList | Renderizado visual diferenciado |

### Acceptance Criteria

- [ ] Sin documentos relevantes → LLM genera respuesta contextual
- [ ] Respuesta de fallback en el mismo idioma de la pregunta
- [ ] Campo `responseType` indica `answer` | `no_context` | `error`
- [ ] Frontend muestra indicador visual cuando es `no_context`
- [ ] Si el LLM falla en el fallback, degrada al string estático
- [ ] Tests unitarios para el nuevo flujo

---

## Feature 4: Respuestas RAG Estructuradas (Genkit Structured Output)

**Estimación:** 8h · **Repos:** `context-ai-api` + `context-ai-front` · **Prioridad:** Media

### Descripción

Usar **Structured Output** de Genkit (`ai.generate` con `output.schema` + Zod) para que el LLM retorne JSON tipado en vez de texto libre. Las respuestas se organizan en secciones con tipos (`info`, `steps`, `warning`, `tip`), key points y temas relacionados. El frontend renderiza con componentes diferenciados por tipo de sección.

### Schema Zod

```typescript
structuredResponseSchema = z.object({
  summary: z.string(),            // Resumen 1-2 oraciones
  sections: z.array(z.object({
    title: z.string(),            // Título de sección
    content: z.string(),          // Contenido markdown
    type: z.enum(['info', 'steps', 'warning', 'tip']),
  })),
  keyPoints: z.array(z.string()).optional(),     // Puntos clave
  relatedTopics: z.array(z.string()).optional(), // Temas relacionados
});
```

### Compatibilidad

- **Backward compatible:** Campo `response` (texto plano) se mantiene. `structured` es opcional
- **Fallback:** Frontend usa `MarkdownRenderer` si `structured` no está presente
- Si Genkit structured output falla, se usa el texto plano

### Archivos a crear/modificar

**API:**

| Archivo | Acción |
|---------|--------|
| `structured-response.schema.ts` | Crear — Schema Zod |
| `rag-query.flow.ts` | Modificar — Usar `output.schema` en `ai.generate` |
| `query-assistant.dto.ts` | Modificar — Agregar DTOs de secciones |
| `interaction-dto.mapper.ts` | Modificar — Mapear structured response |
| `query-assistant.use-case.ts` | Modificar — Propagar structured data |

**Frontend:**

| Archivo | Acción |
|---------|--------|
| `StructuredResponse.tsx` | Crear — Container principal |
| `ResponseSection.tsx` | Crear — Sección individual con icono por tipo |
| `KeyPoints.tsx` | Crear — Lista de puntos clave |
| `RelatedTopics.tsx` | Crear — Chips/badges de temas relacionados |
| `MessageList.tsx` | Modificar — Condicional structured/markdown |
| `message.types.ts` | Modificar — Agregar tipos structured |

### Acceptance Criteria

- [ ] RAG flow usa `output.schema` de Genkit con schema Zod
- [ ] Respuestas del LLM se validan automáticamente contra el schema
- [ ] Campo `response` sigue presente (backward compatible)
- [ ] Campo `structured` contiene respuesta con secciones tipadas
- [ ] Frontend renderiza secciones con iconos diferenciados por tipo
- [ ] Fallback graceful: si structured output falla, usa texto plano
- [ ] Tests unitarios para schema y mapeo
- [ ] Swagger actualizado con nuevos DTOs

---

## Feature 5: Mejora del Workflow de Release Automático

**Estimación:** 2h · **Repo:** `context-ai-api` · **Prioridad:** Media · **Estado:** ✅ Completado

### Descripción

Workflow `release.yml` de GitHub Actions que automatiza la creación de GitHub Releases al pushear tags `v*`. Valida calidad (build + tests) antes de crear el release y genera changelog automático basado en commits desde el tag anterior.

### Flujo

```
Push tag v1.3.0
  → Checkout (full history)
  → Setup Node 22 + pnpm
  → Install → Build → Tests
  → Extraer versión del tag
  → Generar changelog (git log previous_tag..HEAD)
  → Crear GitHub Release (softprops/action-gh-release)
```

**Archivo:** `context-ai-api/.github/workflows/release.yml`

### Acceptance Criteria

- [x] Workflow se dispara al pushear tags `v*`
- [x] Ejecuta build y tests antes de crear release
- [x] Genera changelog automático desde tag anterior
- [x] Crea GitHub Release con versión, changelog e instrucciones

---

## Resumen de impacto total

| Área | Nuevos | Modificados |
|------|--------|-------------|
| Módulos API | `invitations/`, `notifications/` | `users/`, `app.module.ts`, `auth.config.ts` |
| Tablas BD | `invitations`, `invitation_sectors`, `notifications` | — |
| Workflows | `deploy-production.yml` | `release.yml` |
| Componentes Frontend | 8 nuevos | 3 modificados |
| Dependencias | `auth0`, `@nestjs/event-emitter` | — |
| Secrets | `AUTH0_MGMT_*` | — |

## Estimación por feature

| Feature | Estimación | Estado |
|---------|-----------|--------|
| CI/CD Deploy | 4h | Workflow YML creado ✓, falta setup GCP |
| Invitaciones + Notificaciones | 18h | Diseño completo ✓, pendiente implementar |
| Fallback mejorado | 3h | Pendiente |
| Structured Output | 8h | Análisis listo ✓, pendiente implementar |
| Release Workflow | 2h | ✅ Completado |
| **Total** | **35h** | **~4-5 días de trabajo** |

## Ramas de feature

| Feature | Rama |
|---------|------|
| CI/CD Deploy | `feature/v1.3-cicd-cloud-run` |
| Invitaciones + Notificaciones | `feature/v1.3-invitations-notifications` |
| Fallback mejorado | `feature/v1.3-improved-fallback` |
| Structured Output | `feature/v1.3-structured-responses` |
| Release Workflow | `feature/v1.3-release-workflow` |

