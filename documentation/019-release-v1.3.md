---
name: "Release v1.3 — CI/CD, Invitaciones Auth0, Notificaciones In-App y Respuestas Estructuradas"
overview: "Release v1.3 incluye la integración de GitHub Actions con Google Cloud Run para deploy automático de la API, sistema de invitaciones con Auth0 Management API para control de acceso (con asignación de rol y sectores), sistema de notificaciones in-app basado en eventos (@nestjs/event-emitter) reutilizable para v2, mejora del fallback cuando la IA no tiene información, estructuración de respuestas RAG con Genkit Structured Output, y mejora del workflow de release automático."
version: "1.3.0"
date: "2026-02-17"
related_docs:
  - "015-deployment-cloud-architecture.md"
  - "017-improvements-monitoring-observability.md"
  - "005-technical-architecture.md"
  - "020-future-improvements.md"
total_features: 5
---

# Release v1.3 — CI/CD, Invitaciones Auth0, Notificaciones In-App y Respuestas Estructuradas

## 1. Resumen del Release

| Campo | Valor |
|-------|-------|
| **Versión** | 1.3.0 |
| **Fecha** | 17 de febrero de 2026 |
| **Tipo** | Feature Release |
| **Repositorios** | `context-ai-api`, `context-ai-front` |
| **Ramas** | `feature/v1.3-*` → `develop` → `main` |

### Features incluidos

| # | Feature | Repositorio | Prioridad |
|---|---------|-------------|-----------|
| 1 | CI/CD: Deploy automático de API a Cloud Run | `context-ai-api` | Alta |
| 2 | Sistema de Invitaciones + Notificaciones In-App (Auth0) | `context-ai-api` + `context-ai-front` | Alta |
| 3 | Mejora de respuesta fallback cuando no hay información | `context-ai-api` | Media |
| 4 | Respuestas RAG estructuradas (Genkit Structured Output) | `context-ai-api` + `context-ai-front` | Media |
| 5 | Mejora del workflow de Release automático (`release.yml`) | `context-ai-api` | Media |

---

## 2. Feature 1: CI/CD — Deploy Automático de API a Cloud Run

### 2.1 Descripción

Integración de GitHub Actions con Google Cloud Run para automatizar el despliegue de la API a producción. Cada push a `main` ejecuta el pipeline completo: lint → build → test → deploy → smoke test.

### 2.2 Archivo creado

**`context-ai-api/.github/workflows/deploy-production.yml`**

### 2.3 Arquitectura del Pipeline

```
Push to main
    │
    ▼
┌──────────────────────────┐
│  Job 1: Test & Build     │
│  ├─ Checkout             │
│  ├─ Setup Node 22 + pnpm │
│  ├─ Cache dependencies   │
│  ├─ Install (frozen)     │
│  ├─ Lint                 │
│  ├─ Build                │
│  └─ Test (con PG 16)     │
└───────────┬──────────────┘
            │ (success)
            ▼
┌──────────────────────────┐
│  Job 2: Deploy           │
│  ├─ Auth Google Cloud    │
│  │   (Workload Identity) │
│  ├─ Setup gcloud CLI     │
│  ├─ Docker build + push  │
│  │   → Artifact Registry │
│  ├─ Deploy Cloud Run     │
│  ├─ Smoke Test           │
│  │   (health endpoint)   │
│  └─ Notify success       │
└──────────────────────────┘
```

### 2.4 Jobs del Workflow

#### Job 1: `test` — Validación de calidad

- **PostgreSQL 16 Alpine** como servicio para tests
- **pnpm cache** para installs rápidos
- **Lint** → `pnpm lint`
- **Build** → `pnpm build` (verificación de tipos TypeScript)
- **Test** → `pnpm test` (unit tests con cobertura)

#### Job 2: `deploy` — Despliegue a producción

- **Condición**: Solo ejecuta si `test` pasa y la rama es `main`
- **Autenticación**: Workload Identity Federation (OIDC, sin service account keys)
- **Docker**: Build multi-stage con `NODE_AUTH_TOKEN` para GitHub Packages
- **Tags**: `${{ github.sha }}` + `latest`
- **Cloud Run**: Deploy con env vars + Google Secret Manager
- **Smoke Test**: Health check con retries (5 intentos, 10s entre cada uno)

### 2.5 GitHub Secrets requeridos

| Secret | Descripción | Dónde obtenerlo |
|--------|-------------|-----------------|
| `GCP_PROJECT_ID` | ID del proyecto Google Cloud | Google Cloud Console |
| `WIF_PROVIDER` | Workload Identity Federation provider | `gcloud iam workload-identity-pools providers describe` |
| `WIF_SERVICE_ACCOUNT` | Service account para Cloud Run | `gcloud iam service-accounts list` |
| `GH_PACKAGES_TOKEN` | PAT para GitHub Packages (shared pkg) | GitHub Settings → Developer settings → PAT |
| `DB_HOST` | Host de Neon PostgreSQL | Neon Dashboard |
| `DB_USERNAME` | Usuario de Neon PostgreSQL | Neon Dashboard |
| `FRONTEND_URL` | URL del frontend en producción | `https://app.contextai.com` |
| `ALLOWED_ORIGINS` | Orígenes CORS permitidos | `https://app.contextai.com` |

### 2.6 Google Secret Manager (secrets de aplicación)

Estos se inyectan en Cloud Run como secrets, no como env vars:

| Secret | Descripción |
|--------|-------------|
| `DB_PASSWORD` | Password de Neon PostgreSQL |
| `GCP_PROJECT_ID` | Google Cloud project ID (Vertex AI — usa ADC, no API key) |
| `PINECONE_API_KEY` | API Key para Pinecone |
| `AUTH0_DOMAIN` | Dominio Auth0 |
| `AUTH0_AUDIENCE` | Audience Auth0 |
| `SENTRY_DSN` | DSN de Sentry |
| `INTERNAL_API_KEY` | API Key interna para sync de usuarios |

### 2.7 Setup de Workload Identity Federation

```bash
# 1. Crear Workload Identity Pool
gcloud iam workload-identity-pools create "github-actions" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# 2. Crear Provider con OIDC de GitHub
gcloud iam workload-identity-pools providers create-oidc "github" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github-actions" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# 3. Crear Service Account
gcloud iam service-accounts create "github-actions-deployer" \
  --project="${PROJECT_ID}" \
  --display-name="GitHub Actions Deployer"

# 4. Asignar roles necesarios
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:github-actions-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:github-actions-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:github-actions-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:github-actions-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# 5. Permitir que GitHub Actions use el Service Account
gcloud iam service-accounts add-iam-policy-binding \
  "github-actions-deployer@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-actions/attribute.repository/${GITHUB_ORG}/${GITHUB_REPO}"
```

### 2.8 Cloud Run Configuration

| Parámetro | Valor | Razón |
|-----------|-------|-------|
| `port` | 3001 | Puerto del API NestJS |
| `memory` | 512Mi | Suficiente para NestJS + Genkit |
| `cpu` | 1 | 1 vCPU |
| `min-instances` | 0 | Scale to zero (ahorro para TFM) |
| `max-instances` | 3 | Límite de costos |
| `concurrency` | 80 | Requests por instancia |
| `timeout` | 300s | 5 min para RAG queries largas |

---

## 3. Feature 2: Sistema de Invitaciones + Notificaciones In-App

### 3.1 Descripción

Implementar un **sistema de invitaciones** desde el panel de administración para controlar quién puede crear una cuenta en la plataforma. Solo los usuarios que reciban una invitación del admin podrán registrarse. Adicionalmente, se implementa un **sistema de notificaciones in-app** basado en eventos (`@nestjs/event-emitter`) que servirá como infraestructura reutilizable para la v2.

### 3.2 Alcance de la Feature

| Subsistema | Descripción |
|------------|-------------|
| **Invitaciones** | Admin invita usuario → API crea cuenta en Auth0 → usuario recibe email para poner contraseña |
| **Restricción de signup** | Deshabilitar signup público en Auth0; solo admin puede crear cuentas |
| **Notificaciones in-app** | Sistema de eventos persistentes con tabla `notifications` en BD |
| **Event Bus** | `@nestjs/event-emitter` como infraestructura de eventos reutilizable para v2 |

### 3.3 Contexto actual

El flujo actual de usuarios funciona así:

```
Usuario login (Auth0) → Frontend (NextAuth callback) → POST /api/v1/users/sync → API crea/actualiza usuario
```

**Problemas:**
1. Cualquiera que tenga acceso a Auth0 puede crear una cuenta (signup público habilitado)
2. No hay mecanismo de invitación — el admin debe crear cuentas manualmente en Auth0 Dashboard
3. No hay sistema de notificaciones in-app para informar al admin de eventos importantes
4. El botón "Invite User" en `UsersTab.tsx` está deshabilitado (placeholder)

### 3.4 Arquitectura del Sistema de Invitaciones

#### 3.4.1 Enfoque: Auth0 Management API (Recomendado)

El admin invita al usuario directamente desde la app. La API crea la cuenta en Auth0 vía Management API y envía un email de bienvenida con link para configurar contraseña.

**Ventajas:**
- Control total desde nuestra API — no depende de que el usuario vaya a Auth0
- No requiere desplegar Auth0 Actions
- El admin tiene visibilidad de invitaciones pendientes/aceptadas
- Se integra con el botón `UserPlus` ya existente en `UsersTab.tsx`

#### 3.4.2 Flujo completo

```
                        ┌──────────────────────────────────────────────────┐
                        │            PANEL DE ADMINISTRACIÓN               │
                        │  Admin click "Invite User"                       │
                        │  → Ingresa email, nombre, rol                    │
                        │  → Selecciona sectores (multi-select)            │
                        └──────────────┬───────────────────────────────────┘
                                       │
                                       ▼
                        ┌──────────────────────────────┐
                        │  POST /admin/invitations      │
                        │  → Valida email no duplicado  │
                        │  → Crea registro invitation   │
                        │    (status: pending)          │
                        │  → Asocia sectores            │
                        │    seleccionados              │
                        └──────────────┬───────────────┘
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                          ▼                         ▼
               ┌────────────────────┐   ┌─────────────────────────┐
               │  Auth0 Management  │   │  Event: invitation.     │
               │  API               │   │  created                │
               │  → POST /api/v2/   │   │  → NotificationListener │
               │    users           │   │  → Crea notificación    │
               │  → POST /api/v2/   │   │    in-app para admin    │
               │    tickets/        │   └─────────────────────────┘
               │    password-change │
               └────────┬──────────┘
                        │
                        ▼
               ┌────────────────────┐
               │  Usuario recibe    │
               │  email con link    │
               │  → Configura su    │
               │    contraseña      │
               └────────┬──────────┘
                        │
                        ▼
               ┌────────────────────┐
               │  Usuario hace      │
               │  login por         │
               │  primera vez       │
               │  → /users/sync     │
               │  → Invitation      │
               │    status:accepted │
               │  → Asigna rol +    │
               │    sectores de la  │
               │    invitación      │
               │  → Event: user.    │
               │    activated       │
               └────────────────────┘
```

#### 3.4.3 Restricción de Signup en Auth0

**Configuración en Auth0 Dashboard:**

1. Ir a **Authentication → Database → Username-Password-Authentication**
2. En la pestaña **Settings**: **Deshabilitar "Disable Sign Ups"** ✅
3. Esto hace que el formulario de signup no aparezca en Universal Login
4. Solo la API (via Management API) puede crear nuevos usuarios

> **Nota**: Al deshabilitar signup público, el usuario que intente registrarse verá un mensaje de error. El único camino para crear cuentas es vía invitación del admin.

#### 3.4.4 Auth0 Management API — Configuración

Para que la API pueda crear usuarios en Auth0, necesita un **Machine-to-Machine (M2M) Application**:

**Paso 1: Crear M2M Application en Auth0**
1. Auth0 Dashboard → Applications → Create Application
2. Tipo: **Machine to Machine**
3. Nombre: `Context.ai API - User Management`
4. Autorizar la API de **Auth0 Management API**
5. Scopes necesarios: `create:users`, `create:user_tickets`, `read:users`

**Paso 2: Variables de entorno**

```env
# Auth0 Management API (M2M)
AUTH0_MGMT_CLIENT_ID=<m2m-client-id>
AUTH0_MGMT_CLIENT_SECRET=<m2m-client-secret>
AUTH0_MGMT_DOMAIN=<tenant>.auth0.com
AUTH0_DB_CONNECTION=Username-Password-Authentication
```

**Paso 3: Google Secret Manager** (producción)

| Secret | Descripción |
|--------|-------------|
| `AUTH0_MGMT_CLIENT_ID` | M2M Client ID |
| `AUTH0_MGMT_CLIENT_SECRET` | M2M Client Secret |

### 3.5 Modelo de Datos — Invitaciones

#### 3.5.1 Tabla `invitations`

```sql
CREATE TABLE invitations (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email         VARCHAR(255) NOT NULL,
  name          VARCHAR(255) NOT NULL,
  role          VARCHAR(50) NOT NULL DEFAULT 'user',
  status        VARCHAR(20) NOT NULL DEFAULT 'pending',
  token         VARCHAR(255) NOT NULL UNIQUE,
  invited_by    UUID NOT NULL REFERENCES users(id),
  auth0_user_id VARCHAR(255),           -- Se llena cuando Auth0 crea el usuario
  expires_at    TIMESTAMP WITH TIME ZONE NOT NULL,
  accepted_at   TIMESTAMP WITH TIME ZONE,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Tabla intermedia: sectores asignados por invitación (Many-to-Many)
-- Misma estructura que user_sectors para mantener consistencia
CREATE TABLE invitation_sectors (
  invitation_id UUID NOT NULL REFERENCES invitations(id) ON DELETE CASCADE,
  sector_id     UUID NOT NULL REFERENCES sectors(id) ON DELETE CASCADE,
  PRIMARY KEY (invitation_id, sector_id)
);

CREATE INDEX idx_invitations_email ON invitations(email);
CREATE INDEX idx_invitations_status ON invitations(status);
CREATE INDEX idx_invitations_token ON invitations(token);
CREATE INDEX idx_invitation_sectors_sector ON invitation_sectors(sector_id);
```

> **Nota**: La tabla `invitation_sectors` almacena los sectores que el admin seleccionó al crear la invitación. Al aceptar la invitación, estos sectores se copian a la tabla `user_sectors` del nuevo usuario.

#### 3.5.2 Estados de invitación

```typescript
export enum InvitationStatus {
  PENDING = 'pending',       // Invitación creada, email enviado
  ACCEPTED = 'accepted',     // Usuario configuró contraseña y se logueó
  EXPIRED = 'expired',       // Pasó el tiempo de expiración (7 días)
  REVOKED = 'revoked',       // Admin revocó la invitación
}
```

#### 3.5.3 Entity (TypeORM)

```typescript
// src/modules/invitations/infrastructure/persistence/models/invitation.model.ts
@Entity('invitations')
@Index(['email'])
@Index(['status'])
@Index(['token'], { unique: true })
export class InvitationModel {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar', length: 255 })
  email!: string;

  @Column({ type: 'varchar', length: 255 })
  name!: string;

  @Column({ type: 'varchar', length: 50, default: 'user' })
  role!: string;

  @Column({ type: 'varchar', length: 20, default: InvitationStatus.PENDING })
  status!: InvitationStatus;

  @Column({ type: 'varchar', length: 255, unique: true })
  token!: string;

  @Column({ name: 'invited_by', type: 'uuid' })
  invitedBy!: string;

  @ManyToOne(() => UserModel)
  @JoinColumn({ name: 'invited_by' })
  invitedByUser!: UserModel;

  @Column({ name: 'auth0_user_id', type: 'varchar', length: 255, nullable: true })
  auth0UserId!: string | null;

  // Sectores asignados al usuario vía invitación (Many-to-Many)
  // Al aceptar la invitación, se copian a user_sectors
  @ManyToMany(() => SectorModel, { eager: true })
  @JoinTable({
    name: 'invitation_sectors',
    joinColumn: { name: 'invitation_id', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'sector_id', referencedColumnName: 'id' },
  })
  sectors!: SectorModel[];

  @Column({ name: 'expires_at', type: 'timestamp with time zone' })
  expiresAt!: Date;

  @Column({ name: 'accepted_at', type: 'timestamp with time zone', nullable: true })
  acceptedAt!: Date | null;

  @CreateDateColumn({ name: 'created_at', type: 'timestamp with time zone' })
  createdAt!: Date;

  @UpdateDateColumn({ name: 'updated_at', type: 'timestamp with time zone' })
  updatedAt!: Date;
}
```

### 3.6 Modelo de Datos — Notificaciones In-App

#### 3.6.1 Tabla `notifications`

```sql
CREATE TABLE notifications (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id       UUID NOT NULL REFERENCES users(id),
  type          VARCHAR(50) NOT NULL,
  title         VARCHAR(255) NOT NULL,
  message       TEXT NOT NULL,
  is_read       BOOLEAN NOT NULL DEFAULT false,
  metadata      JSONB,
  created_at    TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user_read ON notifications(user_id, is_read);
CREATE INDEX idx_notifications_user_created ON notifications(user_id, created_at DESC);
```

#### 3.6.2 Tipos de notificación

```typescript
export enum NotificationType {
  // v1.3 — Invitaciones
  INVITATION_CREATED = 'invitation.created',     // Admin: "Se envió invitación a X"
  INVITATION_ACCEPTED = 'invitation.accepted',     // Admin: "X aceptó la invitación"
  INVITATION_EXPIRED = 'invitation.expired',       // Admin: "La invitación de X expiró"

  // v1.3 — Usuarios
  USER_ACTIVATED = 'user.activated',               // Admin: "Nuevo usuario X se registró"

  // v2 — Documentos (preparado para extensión)
  DOCUMENT_PROCESSED = 'document.processed',       // Manager: "Documento X procesado"
  DOCUMENT_FAILED = 'document.failed',             // Manager: "Error procesando documento X"
}
```

#### 3.6.3 Entity (TypeORM)

```typescript
// src/modules/notifications/infrastructure/persistence/models/notification.model.ts
@Entity('notifications')
@Index(['userId', 'isRead'])
@Index(['userId', 'createdAt'])
export class NotificationModel {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ name: 'user_id', type: 'uuid' })
  userId!: string;

  @ManyToOne(() => UserModel)
  @JoinColumn({ name: 'user_id' })
  user!: UserModel;

  @Column({ type: 'varchar', length: 50 })
  type!: NotificationType;

  @Column({ type: 'varchar', length: 255 })
  title!: string;

  @Column({ type: 'text' })
  message!: string;

  @Column({ name: 'is_read', type: 'boolean', default: false })
  isRead!: boolean;

  @Column({ type: 'jsonb', nullable: true })
  metadata!: Record<string, unknown> | null;

  @CreateDateColumn({ name: 'created_at', type: 'timestamp with time zone' })
  createdAt!: Date;
}
```

### 3.7 Event Bus — Arquitectura de Eventos

#### 3.7.1 Infraestructura

```typescript
// src/app.module.ts
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot({
      wildcard: false,
      delimiter: '.',
      maxListeners: 10,
      verboseMemoryLeak: true,
    }),
    // ... otros módulos
  ],
})
export class AppModule {}
```

#### 3.7.2 Eventos de dominio

```typescript
// src/modules/invitations/domain/events/invitation.events.ts

export class InvitationCreatedEvent {
  constructor(
    public readonly invitationId: string,
    public readonly email: string,
    public readonly name: string,
    public readonly role: string,
    public readonly sectorIds: string[],  // sectores asignados
    public readonly invitedBy: string,    // userId del admin
    public readonly createdAt: Date,
  ) {}
}

export class InvitationAcceptedEvent {
  constructor(
    public readonly invitationId: string,
    public readonly email: string,
    public readonly name: string,
    public readonly userId: string,       // userId del nuevo usuario
    public readonly acceptedAt: Date,
  ) {}
}
```

```typescript
// src/modules/users/domain/events/user.events.ts

export class UserActivatedEvent {
  constructor(
    public readonly userId: string,
    public readonly email: string,
    public readonly name: string,
    public readonly auth0UserId: string,
    public readonly activatedAt: Date,
  ) {}
}
```

#### 3.7.3 Listeners

```typescript
// src/modules/notifications/application/listeners/invitation-notification.listener.ts

@Injectable()
export class InvitationNotificationListener {
  private readonly logger = new Logger(InvitationNotificationListener.name);

  constructor(private readonly notificationService: NotificationService) {}

  @OnEvent('invitation.created')
  async handleInvitationCreated(event: InvitationCreatedEvent): Promise<void> {
    this.logger.log(`Invitation created for ${event.email} by admin ${event.invitedBy}`);

    // Crear notificación in-app para el admin que invitó
    await this.notificationService.create({
      userId: event.invitedBy,
      type: NotificationType.INVITATION_CREATED,
      title: 'Invitation Sent',
      message: `Invitation sent to ${event.name} (${event.email}) with role: ${event.role}`,
      metadata: {
        invitationId: event.invitationId,
        email: event.email,
        sectorIds: event.sectorIds,
      },
    });
  }

  @OnEvent('invitation.accepted')
  async handleInvitationAccepted(event: InvitationAcceptedEvent): Promise<void> {
    this.logger.log(`Invitation accepted by ${event.email}`);

    // Notificar a TODOS los admins
    await this.notificationService.notifyAdmins({
      type: NotificationType.INVITATION_ACCEPTED,
      title: 'New User Joined',
      message: `${event.name} (${event.email}) has accepted the invitation and joined the platform`,
      metadata: { userId: event.userId, email: event.email },
    });
  }
}
```

### 3.8 API Endpoints

#### 3.8.1 Invitaciones (Admin)

| Método | Endpoint | Descripción | Permiso |
|--------|----------|-------------|---------|
| `POST` | `/admin/invitations` | Crear invitación y cuenta en Auth0 | `users:manage` |
| `GET` | `/admin/invitations` | Listar invitaciones (con filtro por status) | `users:manage` |
| `DELETE` | `/admin/invitations/:id` | Revocar invitación pendiente | `users:manage` |

#### 3.8.2 Notificaciones (Authenticated Users)

| Método | Endpoint | Descripción | Permiso |
|--------|----------|-------------|---------|
| `GET` | `/notifications` | Listar notificaciones del usuario autenticado | JWT auth |
| `GET` | `/notifications/unread-count` | Contar notificaciones no leídas | JWT auth |
| `PATCH` | `/notifications/:id/read` | Marcar notificación como leída | JWT auth |
| `PATCH` | `/notifications/read-all` | Marcar todas como leídas | JWT auth |

#### 3.8.3 DTO de creación de invitación

```typescript
// src/modules/invitations/application/dtos/create-invitation.dto.ts
export class CreateInvitationDto {
  @ApiProperty({ description: 'Email of the user to invite' })
  @IsEmail()
  email!: string;

  @ApiProperty({ description: 'Full name of the user' })
  @IsString()
  @MinLength(2)
  @MaxLength(255)
  name!: string;

  @ApiProperty({ description: 'Role to assign', enum: ['admin', 'manager', 'user'], default: 'user' })
  @IsOptional()
  @IsEnum(['admin', 'manager', 'user'])
  role?: string;

  @ApiProperty({
    description: 'Array of sector UUIDs to grant initial access. Uses same structure as user_sectors.',
    example: ['550e8400-e29b-41d4-a716-446655440000'],
    type: [String],
    required: false,
  })
  @IsOptional()
  @IsArray()
  @IsUUID('4', { each: true })
  sectorIds?: string[];
}
```

#### 3.8.4 Invitation Service — Flujo de creación

```typescript
// src/modules/invitations/application/services/invitation.service.ts

@Injectable()
export class InvitationService {
  constructor(
    private readonly invitationRepository: InvitationRepository,
    private readonly userRepository: UserRepository,
    @Inject('ISectorRepository')
    private readonly sectorRepository: ISectorRepository,
    private readonly auth0ManagementService: Auth0ManagementService,
    private readonly emailService: EmailService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async createInvitation(
    dto: CreateInvitationDto,
    invitedByUserId: string,
  ): Promise<InvitationResponseDto> {
    // 1. Validar que no exista invitación pendiente para este email
    const existing = await this.invitationRepository.findPendingByEmail(dto.email);
    if (existing) {
      throw new ConflictException('An invitation for this email is already pending');
    }

    // 2. Validar que no exista usuario con este email
    const existingUser = await this.userRepository.findByEmail(dto.email);
    if (existingUser) {
      throw new ConflictException('A user with this email already exists');
    }

    // 3. Validar que los sectores existan (si se proporcionaron)
    let sectors: SectorModel[] = [];
    if (dto.sectorIds?.length) {
      for (const sectorId of dto.sectorIds) {
        const sector = await this.sectorRepository.findById(sectorId);
        if (!sector) {
          throw new BadRequestException(`Sector not found: ${sectorId}`);
        }
        sectors.push({ id: sectorId } as SectorModel);
      }
    }

    // 4. Generar token único
    const token = randomUUID();

    // 5. Crear usuario en Auth0 via Management API
    const auth0User = await this.auth0ManagementService.createUser({
      email: dto.email,
      name: dto.name,
      connection: 'Username-Password-Authentication',
    });

    // 6. Generar ticket de cambio de contraseña (envía email al usuario)
    await this.auth0ManagementService.createPasswordChangeTicket({
      user_id: auth0User.user_id,
      result_url: `${this.frontendUrl}/auth/login`,
    });

    // 7. Persistir invitación con sectores asociados
    const invitation = await this.invitationRepository.save({
      email: dto.email,
      name: dto.name,
      role: dto.role ?? 'user',
      status: InvitationStatus.PENDING,
      token,
      invitedBy: invitedByUserId,
      auth0UserId: auth0User.user_id,
      sectors,                                                    // ManyToMany — se persiste en invitation_sectors
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 días
    });

    // 8. Emitir evento
    this.eventEmitter.emit(
      'invitation.created',
      new InvitationCreatedEvent(
        invitation.id,
        dto.email,
        dto.name,
        dto.role ?? 'user',
        dto.sectorIds ?? [],
        invitedByUserId,
        new Date(),
      ),
    );

    return this.mapToDto(invitation);
  }
}
```

#### 3.8.5 Auth0 Management Service

```typescript
// src/modules/invitations/infrastructure/auth0/auth0-management.service.ts
import { ManagementClient } from 'auth0';

@Injectable()
export class Auth0ManagementService {
  private readonly client: ManagementClient;

  constructor(private readonly configService: ConfigService) {
    this.client = new ManagementClient({
      domain: this.configService.getOrThrow('AUTH0_MGMT_DOMAIN'),
      clientId: this.configService.getOrThrow('AUTH0_MGMT_CLIENT_ID'),
      clientSecret: this.configService.getOrThrow('AUTH0_MGMT_CLIENT_SECRET'),
    });
  }

  async createUser(params: {
    email: string;
    name: string;
    connection: string;
  }): Promise<{ user_id: string }> {
    const user = await this.client.users.create({
      email: params.email,
      name: params.name,
      connection: params.connection,
      password: randomUUID() + 'Aa1!', // Temp password (user will reset via ticket)
      email_verified: false,
    });
    return { user_id: user.data.user_id };
  }

  async createPasswordChangeTicket(params: {
    user_id: string;
    result_url: string;
  }): Promise<string> {
    const ticket = await this.client.tickets.changePassword({
      user_id: params.user_id,
      result_url: params.result_url,
      mark_email_as_verified: true,
    });
    return ticket.data.ticket;
  }
}
```

#### 3.8.6 Email de invitación

El email de password-reset que envía Auth0 es el único email que recibe el usuario invitado. Se decidió no integrar un servicio de email transaccional adicional (Resend, SendGrid, etc.) ya que Auth0 cubre la necesidad principal. La integración de emails personalizados queda como mejora futura (ver `020-future-improvements.md` → FUT-2).

### 3.9 Integración con User Sync (Aceptación de invitación)

Cuando un usuario invitado se loguea por primera vez, el flujo existente `POST /users/sync` se ejecuta. Debemos detectar si este usuario tiene una invitación pendiente y marcarla como aceptada:

```typescript
// Modificar: src/modules/users/application/services/user.service.ts

async syncUser(dto: SyncUserDto): Promise<UserResponseDto> {
  let user = await this.userRepository.findByAuth0UserId(dto.auth0UserId);

  if (user) {
    // Update last login (flujo existente)
    user = await this.userRepository.save({ ...user, lastLoginAt: new Date() });
  } else {
    // Create new user (flujo existente)
    user = await this.userRepository.save({
      auth0UserId: dto.auth0UserId,
      email: dto.email,
      name: dto.name,
      lastLoginAt: new Date(),
      isActive: true,
    });

    // NUEVO: Verificar y aceptar invitación pendiente
    const invitation = await this.invitationService.findPendingByEmail(dto.email);
    if (invitation) {
      await this.invitationService.acceptInvitation(invitation.id, user.id);
      // Asignar rol de la invitación
      await this.adminUserService.updateRole(user.id, invitation.role);
      // Asignar sectores de la invitación (si hay)
      // Reutiliza el mismo servicio que usa PermissionsTab
      const sectorIds = invitation.sectors?.map(s => s.id) ?? [];
      if (sectorIds.length > 0) {
        await this.adminUserService.updateUserSectors(user.id, sectorIds);
      }
    }

    // Emitir evento de usuario activado
    this.eventEmitter.emit(
      'user.activated',
      new UserActivatedEvent(user.id, user.email, user.name, dto.auth0UserId, new Date()),
    );
  }

  return this.mapToDto(user, roles);
}
```

### 3.10 Frontend — Componentes

#### 3.10.1 Diálogo de invitación (`InviteUserDialog.tsx`)

Se activará desde el botón `UserPlus` ya existente en `UsersTab.tsx`:

```typescript
// src/components/admin/InviteUserDialog.tsx

interface InviteUserDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  sectors: Sector[];              // Sectores activos (se reciben de AdminView)
  onSuccess: () => void;          // Refresca lista tras invitación exitosa
}

export function InviteUserDialog({ open, onOpenChange, sectors, onSuccess }: InviteUserDialogProps) {
  const [email, setEmail] = useState('');
  const [name, setName] = useState('');
  const [role, setRole] = useState<'admin' | 'manager' | 'user'>('user');
  const [selectedSectorIds, setSelectedSectorIds] = useState<string[]>([]);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const activeSectors = sectors.filter(s => s.status === 'active');

  const handleSubmit = async () => {
    setIsSubmitting(true);
    try {
      await invitationApi.create({ email, name, role, sectorIds: selectedSectorIds });
      toast.success(`Invitation sent to ${email}`);
      onSuccess();
      onOpenChange(false);
    } catch (err) {
      toast.error(extractErrorMessage(err));
    } finally {
      setIsSubmitting(false);
    }
  };

  // Formulario:
  // - Input: email (required)
  // - Input: name (required)
  // - Select: role (admin | manager | user, default: user)
  // - Multi-select checkboxes: sectores activos
  //   → Usa checkboxes dentro de un ScrollArea para selección múltiple
  //   → Muestra nombre + icono de cada sector
  //   → Badge count de sectores seleccionados
  //   → Si role === 'admin', sectores se auto-seleccionan todos y se deshabilitan
  //     (admins tienen acceso a todos los sectores automáticamente)
  // - Botón: "Send Invitation" (disabled si email/name vacíos o isSubmitting)
}
```

#### 3.10.2 Sistema de notificaciones en el header

```typescript
// src/components/shared/NotificationBell.tsx
// - Icono de campana en el header con badge de count
// - Dropdown con lista de notificaciones recientes
// - Polling cada 30s a GET /notifications/unread-count
// - Click en notificación → marca como leída

// src/components/shared/NotificationDropdown.tsx
// - Lista scrolleable de notificaciones
// - Cada item: tipo (icono), título, mensaje, fecha relativa
// - Botón "Mark all as read"
// - Link "View all" → future notifications page
```

#### 3.10.3 Componentes UI necesarios

| Componente | Repositorio | Descripción |
|------------|-------------|-------------|
| `InviteUserDialog.tsx` | `context-ai-front` | Modal de invitación con formulario (email, nombre, rol, multi-select sectores) |
| `NotificationBell.tsx` | `context-ai-front` | Icono campana + badge en header |
| `NotificationDropdown.tsx` | `context-ai-front` | Lista desplegable de notificaciones |
| `NotificationItem.tsx` | `context-ai-front` | Ítem individual de notificación |
| `InvitationsTab.tsx` | `context-ai-front` | Tab en admin para ver invitaciones pendientes (opcional v1.3) |

### 3.11 Dependencias a instalar

```bash
# API — Auth0 Management SDK + Event Emitter
pnpm add auth0 @nestjs/event-emitter

# Tipos para Auth0
pnpm add -D @types/auth0
```

### 3.12 Estructura de módulos (API)

```
src/modules/
├── invitations/                          # NUEVO módulo
│   ├── invitations.module.ts
│   ├── domain/
│   │   ├── entities/invitation.entity.ts
│   │   └── events/invitation.events.ts
│   ├── application/
│   │   ├── services/invitation.service.ts
│   │   └── dtos/
│   │       ├── create-invitation.dto.ts
│   │       └── invitation-response.dto.ts
│   ├── infrastructure/
│   │   ├── auth0/auth0-management.service.ts
│   │   └── persistence/
│   │       ├── models/invitation.model.ts
│   │       └── repositories/invitation.repository.ts
│   └── api/
│       └── controllers/invitation.controller.ts
│
├── notifications/                        # NUEVO módulo
│   ├── notifications.module.ts
│   ├── domain/
│   │   └── entities/notification.entity.ts
│   ├── application/
│   │   ├── services/notification.service.ts
│   │   ├── listeners/
│   │   │   ├── invitation-notification.listener.ts
│   │   │   └── user-notification.listener.ts
│   │   └── dtos/
│   │       └── notification-response.dto.ts
│   ├── infrastructure/
│   │   └── persistence/
│   │       ├── models/notification.model.ts
│   │       └── repositories/notification.repository.ts
│   └── api/
│       └── controllers/notification.controller.ts
│
└── users/                                # MODIFICAR
    └── application/
        └── services/user.service.ts      # Integrar aceptación de invitación
```

### 3.13 Migración de Base de Datos

```typescript
// src/migrations/1740000000000-CreateInvitationsAndNotificationsTables.ts
export class CreateInvitationsAndNotificationsTables1740000000000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // Tabla invitations
    await queryRunner.createTable(new Table({
      name: 'invitations',
      columns: [
        { name: 'id', type: 'uuid', isPrimary: true, default: 'uuid_generate_v4()' },
        { name: 'email', type: 'varchar', length: '255' },
        { name: 'name', type: 'varchar', length: '255' },
        { name: 'role', type: 'varchar', length: '50', default: "'user'" },
        { name: 'status', type: 'varchar', length: '20', default: "'pending'" },
        { name: 'token', type: 'varchar', length: '255', isUnique: true },
        { name: 'invited_by', type: 'uuid' },
        { name: 'auth0_user_id', type: 'varchar', length: '255', isNullable: true },
        { name: 'expires_at', type: 'timestamp with time zone' },
        { name: 'accepted_at', type: 'timestamp with time zone', isNullable: true },
        { name: 'created_at', type: 'timestamp with time zone', default: 'CURRENT_TIMESTAMP' },
        { name: 'updated_at', type: 'timestamp with time zone', default: 'CURRENT_TIMESTAMP' },
      ],
      foreignKeys: [
        { columnNames: ['invited_by'], referencedTableName: 'users', referencedColumnNames: ['id'] },
      ],
    }));

    // Tabla notifications
    await queryRunner.createTable(new Table({
      name: 'notifications',
      columns: [
        { name: 'id', type: 'uuid', isPrimary: true, default: 'uuid_generate_v4()' },
        { name: 'user_id', type: 'uuid' },
        { name: 'type', type: 'varchar', length: '50' },
        { name: 'title', type: 'varchar', length: '255' },
        { name: 'message', type: 'text' },
        { name: 'is_read', type: 'boolean', default: false },
        { name: 'metadata', type: 'jsonb', isNullable: true },
        { name: 'created_at', type: 'timestamp with time zone', default: 'CURRENT_TIMESTAMP' },
      ],
      foreignKeys: [
        { columnNames: ['user_id'], referencedTableName: 'users', referencedColumnNames: ['id'] },
      ],
    }));

    // Tabla invitation_sectors (Many-to-Many: invitations <-> sectors)
    await queryRunner.createTable(new Table({
      name: 'invitation_sectors',
      columns: [
        { name: 'invitation_id', type: 'uuid', isPrimary: true },
        { name: 'sector_id', type: 'uuid', isPrimary: true },
      ],
      foreignKeys: [
        {
          columnNames: ['invitation_id'],
          referencedTableName: 'invitations',
          referencedColumnNames: ['id'],
          onDelete: 'CASCADE',
        },
        {
          columnNames: ['sector_id'],
          referencedTableName: 'sectors',
          referencedColumnNames: ['id'],
          onDelete: 'CASCADE',
        },
      ],
    }));

    // Indexes
    await queryRunner.createIndex('invitations', new TableIndex({
      name: 'idx_invitations_email', columnNames: ['email'],
    }));
    await queryRunner.createIndex('invitations', new TableIndex({
      name: 'idx_invitations_status', columnNames: ['status'],
    }));
    await queryRunner.createIndex('invitation_sectors', new TableIndex({
      name: 'idx_invitation_sectors_sector', columnNames: ['sector_id'],
    }));
    await queryRunner.createIndex('notifications', new TableIndex({
      name: 'idx_notifications_user_read', columnNames: ['user_id', 'is_read'],
    }));
    await queryRunner.createIndex('notifications', new TableIndex({
      name: 'idx_notifications_user_created', columnNames: ['user_id', 'created_at'],
    }));
  }
}
```

### 3.14 Variables de entorno

| Variable | Descripción | Secret Manager |
|----------|-------------|----------------|
| `AUTH0_MGMT_CLIENT_ID` | M2M Client ID para Management API | ✅ |
| `AUTH0_MGMT_CLIENT_SECRET` | M2M Client Secret | ✅ |
| `AUTH0_MGMT_DOMAIN` | Auth0 tenant domain | env var |
| `AUTH0_DB_CONNECTION` | Nombre de la conexión de BD en Auth0 | env var |

### 3.15 Archivos a crear/modificar

**API (`context-ai-api`):**

| Archivo | Acción | Descripción |
|---------|--------|-------------|
| `src/modules/invitations/` (todo el módulo) | Crear | Módulo de invitaciones (service, controller, DTOs, repository) |
| `src/modules/invitations/infrastructure/auth0/auth0-management.service.ts` | Crear | Servicio Auth0 Management API |
| `src/modules/notifications/` (todo el módulo) | Crear | Módulo de notificaciones |
| `src/modules/users/application/services/user.service.ts` | Modificar | Aceptar invitación en sync |
| `src/app.module.ts` | Modificar | Importar EventEmitterModule, InvitationsModule, NotificationsModule |
| `src/config/auth.config.ts` | Modificar | Agregar config M2M |
| `src/migrations/1740000000000-CreateInvitationsAndNotificationsTables.ts` | Crear | Migración BD (invitations, invitation_sectors, notifications) |
| `.github/workflows/deploy-production.yml` | Modificar | Agregar secrets M2M a Secret Manager |

**Frontend (`context-ai-front`):**

| Archivo | Acción | Descripción |
|---------|--------|-------------|
| `src/components/admin/InviteUserDialog.tsx` | Crear | Modal de invitación (email, nombre, rol, multi-select sectores) |
| `src/components/admin/UsersTab.tsx` | Modificar | Habilitar botón Invite |
| `src/components/shared/NotificationBell.tsx` | Crear | Campana + badge |
| `src/components/shared/NotificationDropdown.tsx` | Crear | Dropdown de notificaciones |
| `src/components/shared/NotificationItem.tsx` | Crear | Ítem de notificación |
| `src/lib/api/invitation.api.ts` | Crear | API client invitaciones |
| `src/lib/api/notification.api.ts` | Crear | API client notificaciones |
| `src/components/layout/Header.tsx` (o similar) | Modificar | Agregar NotificationBell |

### 3.16 Acceptance Criteria

**Invitaciones:**
- [ ] Admin puede invitar usuario con email, nombre, rol y **sectores** desde el panel de admin
- [ ] El formulario de invitación muestra multi-select de sectores activos
- [ ] Si el rol es `admin`, los sectores se auto-seleccionan (admins tienen acceso completo)
- [ ] La API crea el usuario en Auth0 via Management API
- [ ] El usuario recibe email de password-reset de Auth0
- [ ] La invitación queda registrada con estado `pending` en la BD, con sectores asociados en `invitation_sectors`
- [ ] Cuando el usuario se loguea por primera vez, la invitación pasa a `accepted`
- [ ] Al aceptar, se asignan automáticamente el rol y los sectores de la invitación al nuevo usuario (en `user_sectors`)
- [ ] No se pueden crear invitaciones duplicadas para el mismo email
- [ ] Las invitaciones expiran después de 7 días
- [ ] El admin puede revocar invitaciones pendientes
- [ ] El signup público está deshabilitado en Auth0

**Notificaciones:**
- [ ] Se crea notificación in-app cuando se envía una invitación
- [ ] Se crea notificación para todos los admins cuando un usuario acepta la invitación
- [ ] El usuario ve un icono de campana con badge de notificaciones no leídas
- [ ] Al abrir el dropdown se muestran las notificaciones recientes
- [ ] Se pueden marcar notificaciones como leídas (individual y todas)
- [ ] Los eventos se procesan de forma asíncrona (no bloquean el flujo principal)
- [ ] Tests unitarios para InvitationService, NotificationService y listeners

---

## 4. Feature 3: Mejora de Respuesta Fallback (Sin Información)

### 4.1 Problema actual

Cuando la IA no encuentra información relevante en los documentos del sector, retorna un mensaje genérico hardcodeado:

```typescript
// Actual (rag-query.flow.ts)
const FALLBACK_RESPONSE =
  "I don't have information about that in the current documentation. Please contact HR or your manager for more specific guidance.";
```

**Problemas:**
1. **Estático**: No se adapta al idioma del usuario ni al contexto de la pregunta
2. **Genérico**: No indica qué documentos se consultaron ni sugiere alternativas
3. **No aprovecha la IA**: El fallback es un string, no pasa por el LLM
4. **Sin metadata**: El frontend no puede distinguir una respuesta "sin información" de una respuesta normal

### 4.2 Diseño propuesto

#### 4.2.1 Fallback inteligente con LLM

En vez de retornar un string estático, pasar por el LLM con un prompt especializado para generar una respuesta empática y contextual:

```typescript
// Nuevo prompt para fallback
function buildFallbackPrompt(query: string, sectorName?: string): string {
  return `You are a helpful onboarding assistant. The user asked a question, but no relevant documentation was found in the knowledge base${sectorName ? ` for the "${sectorName}" sector` : ''}.

USER QUESTION: "${query}"

INSTRUCTIONS:
- Acknowledge that you don't have specific information about this topic in the available documentation
- Be empathetic and helpful in your response
- Suggest general alternatives (e.g., contact HR, check the company intranet, ask their manager)
- If the question seems related to common onboarding topics, mention that the documentation might not have been uploaded yet
- Keep the response concise (2-3 sentences max)
- Respond in the SAME LANGUAGE as the user's question

RESPONSE:`;
}
```

#### 4.2.2 Metadata de tipo de respuesta

Agregar un campo `responseType` al output para que el frontend pueda distinguir respuestas con y sin contexto:

```typescript
// Nuevo enum
export enum RagResponseType {
  ANSWER = 'answer',           // Respuesta con contexto documental
  NO_CONTEXT = 'no_context',   // Sin documentos relevantes
  ERROR = 'error',             // Error en el procesamiento
}

// Agregar al output schema
export const ragQueryOutputSchema = z.object({
  response: z.string(),
  responseType: z.nativeEnum(RagResponseType),
  sources: z.array(fragmentSchema),
  // ... resto del schema
});
```

### 4.3 Archivos a modificar

| Archivo | Cambio |
|---------|--------|
| `src/shared/genkit/flows/rag-query.flow.ts` | Nuevo prompt de fallback, responseType, llamada al LLM |
| `src/modules/interaction/presentation/dtos/query-assistant.dto.ts` | Agregar `responseType` al DTO |
| `src/modules/interaction/presentation/mappers/interaction-dto.mapper.ts` | Mapear responseType |
| `src/modules/interaction/application/use-cases/query-assistant.use-case.ts` | Propagar responseType |
| Frontend: tipos y MessageList | Renderizado visual diferenciado |

### 4.4 Acceptance Criteria

- [ ] Cuando no hay documentos relevantes, el LLM genera una respuesta contextual
- [ ] La respuesta de fallback se genera en el mismo idioma de la pregunta
- [ ] El campo `responseType` indica si la respuesta tiene contexto o no
- [ ] El frontend muestra un indicador visual cuando la respuesta es `no_context`
- [ ] El fallback no bloquea si el LLM falla (degrada al string estático actual)
- [ ] Tests unitarios para el nuevo flujo de fallback

---

## 5. Feature 4: Respuestas RAG Estructuradas (Genkit Structured Output)

### 5.1 Descripción

Usar la funcionalidad de **Structured Output** de Genkit (`ai.generate` con `output.schema`) para que el LLM retorne respuestas JSON tipadas en lugar de texto libre. Esto permite al frontend renderizar las respuestas con mejor formato, secciones, listas, y metadata.

**Referencia**: https://examples.genkit.dev/structured-output

### 5.2 Concepto de Genkit Structured Output

Genkit permite definir un schema Zod como `output.schema` en `ai.generate()`. El LLM retorna JSON que se valida automáticamente contra el schema:

```typescript
const result = await ai.generate({
  model: 'vertexai/gemini-2.5-flash',
  prompt: '...',
  output: {
    schema: z.object({
      answer: z.string(),
      sections: z.array(z.object({
        title: z.string(),
        content: z.string(),
      })),
    }),
  },
});

// result.output es tipado y validado
const structured = result.output;
```

### 5.3 Schema de Respuesta Estructurada

Estructura de respuesta con secciones tipadas, ideal para organizar respuestas que puedan tener múltiples aspectos (procedimientos, advertencias, tips).

```typescript
export const structuredResponseSchema = z.object({
  summary: z.string().describe('Brief 1-2 sentence summary answering the question directly'),
  sections: z.array(z.object({
    title: z.string().describe('Section heading'),
    content: z.string().describe('Section content in markdown format'),
    type: z.enum(['info', 'steps', 'warning', 'tip']).describe('Section type for UI rendering'),
  })).describe('Detailed sections expanding on the answer'),
  keyPoints: z.array(z.string()).describe('Key takeaways as bullet points').optional(),
  relatedTopics: z.array(z.string()).describe('Related topics the user might want to explore').optional(),
});
```

**Ejemplo de salida:**
```json
{
  "summary": "Para solicitar vacaciones, debes usar el portal HR con al menos 15 días de anticipación.",
  "sections": [
    {
      "title": "Proceso de solicitud",
      "content": "1. Accede al portal HR\n2. Selecciona 'Solicitar vacaciones'\n3. Ingresa las fechas deseadas\n4. Adjunta justificación",
      "type": "steps"
    },
    {
      "title": "Requisitos importantes",
      "content": "Debes tener al menos 6 meses en la empresa y saldo de días disponible.",
      "type": "warning"
    }
  ],
  "keyPoints": [
    "Mínimo 15 días de anticipación",
    "Aprobación del manager directo",
    "Máximo 15 días consecutivos"
  ],
  "relatedTopics": ["Días de enfermedad", "Permisos especiales", "Política de trabajo remoto"]
}
```

**Ventajas de esta estructura:**

| Criterio | Valoración |
|----------|-----------|
| **Complejidad frontend** | Media — componentes reutilizables por tipo de sección |
| **Calidad visual** | Alta — secciones diferenciadas con iconos por tipo |
| **Fiabilidad LLM** | Alta — schema simple, Gemini lo respeta consistentemente |
| **Flexibilidad** | Buena — cubre la mayoría de casos de uso de onboarding |
| **Costo tokens** | Medio — overhead razonable vs texto libre |

> **Nota:** Se evaluó también una Propuesta C (respuesta híbrida/adaptativa) donde el LLM decide el formato según la complejidad de la pregunta. Se documentó en [020-future-improvements.md](./020-future-improvements.md) como mejora futura para v2, ya que requiere un schema más complejo y mayor esfuerzo de frontend.

### 5.4 Implementación

#### 5.4.1 Cambios en el RAG Flow

```typescript
// src/shared/genkit/flows/rag-query.flow.ts

// Nuevo schema de structured output
export const structuredRagResponseSchema = z.object({
  summary: z.string().describe('Brief 1-2 sentence answer'),
  sections: z.array(z.object({
    title: z.string(),
    content: z.string(),
    type: z.enum(['info', 'steps', 'warning', 'tip']),
  })),
  keyPoints: z.array(z.string()).optional(),
  relatedTopics: z.array(z.string()).optional(),
});

// Uso en executeQuery
const result = await ai.generate({
  model: GENKIT_CONFIG.LLM_MODEL,
  prompt: buildPrompt(validatedInput.query, relevantFragments),
  output: {
    schema: structuredRagResponseSchema,
  },
  config: GENKIT_CONFIG.RAG_GENERATION_CONFIG,
});

// result.output es tipado automáticamente
const structured = result.output;
```

#### 5.4.2 Nuevo prompt adaptado para structured output

```typescript
function buildStructuredPrompt(query: string, fragments: FragmentResult[]): string {
  const context = fragments
    .map((f, index) => `[${index + 1}] ${f.content}`)
    .join('\n\n');

  return `You are an onboarding assistant for the company. Answer the following question based ONLY on the provided documentation.

DOCUMENTATION CONTEXT:
${context}

USER QUESTION:
${query}

INSTRUCTIONS:
- Provide a brief summary (1-2 sentences) directly answering the question
- Organize detailed information into logical sections
- Each section should have a type: "info" (general), "steps" (procedures), "warning" (important notes), "tip" (helpful advice)
- Include key takeaways as bullet points when relevant
- Suggest related topics the user might want to explore
- Use markdown formatting within section content
- Respond in the SAME LANGUAGE as the user's question
- If the documentation doesn't fully cover the topic, be transparent about it`;
}
```

#### 5.4.3 Cambios en el DTO de respuesta

```typescript
// Nuevo DTO para secciones
export class ResponseSectionDto {
  @ApiProperty({ description: 'Section title' })
  title!: string;

  @ApiProperty({ description: 'Section content (markdown)' })
  content!: string;

  @ApiProperty({ description: 'Section type', enum: ['info', 'steps', 'warning', 'tip'] })
  type!: 'info' | 'steps' | 'warning' | 'tip';
}

// Extender QueryAssistantResponseDto
export class QueryAssistantResponseDto {
  @ApiProperty({ description: 'Plain text response (backward compatible)' })
  response!: string;

  @ApiProperty({ description: 'Structured response', required: false })
  structured?: {
    summary: string;
    sections: ResponseSectionDto[];
    keyPoints?: string[];
    relatedTopics?: string[];
  };

  // ... campos existentes
}
```

### 5.5 Impacto en Frontend

El frontend deberá adaptar el `MessageList` para renderizar respuestas estructuradas cuando estén disponibles:

```typescript
// Lógica en MessageList.tsx
{isAssistant && message.structured ? (
  <StructuredResponse data={message.structured} />
) : (
  <MarkdownRenderer content={message.content} />
)}
```

Componentes nuevos sugeridos:
- `StructuredResponse.tsx` — Container principal
- `ResponseSection.tsx` — Renderiza cada sección con icono según tipo
- `KeyPoints.tsx` — Lista de puntos clave
- `RelatedTopics.tsx` — Chips/badges de temas relacionados

### 5.6 Compatibilidad hacia atrás

El campo `response` (texto plano) se mantiene para compatibilidad. El campo `structured` es opcional. El frontend puede hacer fallback al `MarkdownRenderer` si `structured` no está presente.

### 5.7 Archivos a crear/modificar

**API (`context-ai-api`):**

| Archivo | Acción | Descripción |
|---------|--------|-------------|
| `src/shared/genkit/schemas/structured-response.schema.ts` | Crear | Schema Zod para structured output |
| `src/shared/genkit/flows/rag-query.flow.ts` | Modificar | Usar `output.schema` en `ai.generate` |
| `src/modules/interaction/presentation/dtos/query-assistant.dto.ts` | Modificar | Agregar DTOs de secciones |
| `src/modules/interaction/presentation/mappers/interaction-dto.mapper.ts` | Modificar | Mapear structured response |
| `src/modules/interaction/application/use-cases/query-assistant.use-case.ts` | Modificar | Propagar structured data |

**Frontend (`context-ai-front`):**

| Archivo | Acción | Descripción |
|---------|--------|-------------|
| `src/types/message.types.ts` | Modificar | Agregar tipos de structured response |
| `src/components/chat/StructuredResponse.tsx` | Crear | Componente de respuesta estructurada |
| `src/components/chat/ResponseSection.tsx` | Crear | Sección individual |
| `src/components/chat/KeyPoints.tsx` | Crear | Puntos clave |
| `src/components/chat/RelatedTopics.tsx` | Crear | Temas relacionados |
| `src/components/chat/MessageList.tsx` | Modificar | Condicional structured/markdown |

### 5.8 Acceptance Criteria

- [ ] El RAG flow usa `output.schema` de Genkit con el schema Zod definido
- [ ] Las respuestas del LLM se validan automáticamente contra el schema
- [ ] El campo `response` sigue presente (backward compatible)
- [ ] El campo `structured` contiene la respuesta organizada en secciones
- [ ] El frontend renderiza secciones con iconos diferenciados por tipo
- [ ] Fallback graceful: si structured output falla, se usa el texto plano
- [ ] Tests unitarios para el nuevo schema y mapeo
- [ ] API contract (Swagger) actualizado con nuevos DTOs

---

## 6. Feature 5: Mejora del Workflow de Release Automático

### 6.1 Descripción

Se creó/mejoró el workflow `release.yml` de GitHub Actions para automatizar la creación de GitHub Releases cuando se publica un tag de versión (`v*`). El workflow valida la calidad del código antes de crear el release y genera un changelog automático basado en los commits.

### 6.2 Archivo

**`context-ai-api/.github/workflows/release.yml`**

### 6.3 Flujo del Workflow

```
Push tag v* (ej: v1.3.0)
    │
    ▼
┌──────────────────────────────────┐
│  Job: Create Release             │
│  ├─ Checkout (full history)      │
│  ├─ Setup Node 22 + pnpm 10     │
│  ├─ Install (frozen-lockfile)    │
│  │   → GitHub Packages auth      │
│  ├─ Build (validación TypeScript)│
│  ├─ Run tests                    │
│  ├─ Extraer versión del tag      │
│  ├─ Generar changelog automático │
│  │   (commits desde tag anterior)│
│  └─ Crear GitHub Release         │
│      → softprops/action-gh-release│
└──────────────────────────────────┘
```

### 6.4 Características clave

| Característica | Detalle |
|----------------|---------|
| **Trigger** | Push de tags con formato `v*` (ej: `v1.3.0`, `v1.3.1-rc.1`) |
| **Validación pre-release** | Ejecuta `pnpm build` + `pnpm test` antes de crear el release |
| **GitHub Packages** | Usa `PKG_READ_TOKEN` para autenticar `@context-ai-project/shared` |
| **Changelog automático** | Genera lista de commits entre el tag anterior y HEAD |
| **Primer release** | Si no existe tag previo, incluye todos los commits del historial |
| **Full history** | Checkout con `fetch-depth: 0` para acceso completo al historial git |
| **GitHub Release** | Usa `softprops/action-gh-release@v2` con body auto-generado |

### 6.5 Generación del Changelog

El changelog se genera automáticamente comparando los commits entre el tag anterior y el tag actual:

```bash
# Si existe tag previo
git log "${PREVIOUS_TAG}..HEAD" --pretty=format:"- %s (%h)"

# Si es el primer release
git log --pretty=format:"- %s (%h)"
```

**Formato de salida del release:**

```markdown
## Changes in v1.3.0

- feat: add CI/CD deploy to Cloud Run (a1b2c3d)
- feat: add new user creation notification (d4e5f6g)
- feat: improve fallback response for no-context queries (h7i8j9k)
- feat: add structured output with Genkit (l0m1n2o)
- fix: correct CORS configuration for production (p3q4r5s)

## Installation

git clone https://github.com/gromeroalfonso/context-ai-api.git
cd context-ai-api
git checkout v1.3.0
pnpm install
```

### 6.6 Permisos del Workflow

| Permiso | Nivel | Razón |
|---------|-------|-------|
| `contents` | `write` | Crear GitHub Releases |
| `packages` | `read` | Leer `@context-ai-project/shared` desde GitHub Packages |

### 6.7 GitHub Secrets requeridos

| Secret | Descripción |
|--------|-------------|
| `PKG_READ_TOKEN` | Personal Access Token con permiso `read:packages` para GitHub Packages |
| `GITHUB_TOKEN` | Token automático de GitHub Actions (usado por softprops/action-gh-release) |

### 6.8 Cómo crear un release

```bash
# 1. Asegurar que main tiene los cambios finales
git checkout main
git pull origin main

# 2. Crear y pushear el tag
git tag v1.3.0
git push origin v1.3.0

# 3. El workflow se ejecuta automáticamente:
#    - Valida build + tests
#    - Genera changelog
#    - Crea GitHub Release en https://github.com/gromeroalfonso/context-ai-api/releases
```

### 6.9 Acceptance Criteria

- [x] Workflow se dispara al pushear tags `v*`
- [x] Ejecuta build y tests antes de crear el release
- [x] Autentica con GitHub Packages para la dependencia shared
- [x] Genera changelog automático desde el tag anterior
- [x] Crea un GitHub Release con versión, changelog e instrucciones de instalación
- [x] No crea releases como draft ni pre-release (publicación directa)

---

## 7. Plan de Implementación

### 7.1 Orden de ejecución

```
Feature 1 (CI/CD Deploy)       ████████░░  [Workflow YML creado ✓, falta setup GCP]
Feature 2 (Invitaciones+Notif) ░░░░░░░░░░  [Diseño completo ✓, pendiente implementar]
Feature 3 (Fallback mejorado)  ░░░░░░░░░░  [Pendiente]
Feature 4 (Structured Output)  ░░░░░░░░░░  [Análisis listo ✓, pendiente implementar]
Feature 5 (Release Workflow)   ██████████  [Completado ✓]
```

### 7.2 Ramas de feature

| Feature | Rama | Base |
|---------|------|------|
| CI/CD Deploy | `feature/v1.3-cicd-cloud-run` | `develop` |
| Invitaciones + Notificaciones | `feature/v1.3-invitations-notifications` | `develop` |
| Fallback mejorado | `feature/v1.3-improved-fallback` | `develop` |
| Structured Output | `feature/v1.3-structured-responses` | `develop` |
| Release Workflow | `feature/v1.3-release-workflow` | `develop` |

### 7.3 Estimación de esfuerzo

| Feature | Estimación | Detalle |
|---------|-----------|---------|
| CI/CD Deploy | 4h | YML creado, falta setup GCP + test |
| Invitaciones + Notificaciones | 18h | Auth0 M2M + invitations module + notifications module + event bus + frontend (dialog + bell) + migration + tests |
| Fallback mejorado | 3h | Prompt + responseType + tests |
| Structured Output | 8h | Schema + flow + DTOs + frontend |
| Release Workflow | 2h | Completado ✓ |
| **Total** | **35h** | ~4-5 días de trabajo |

---

## 8. Validación Pre-Release

### 8.1 Checklist de calidad

- [ ] `pnpm lint` pasa sin errores (API)
- [ ] `pnpm build` pasa sin errores de tipos (API)
- [ ] `pnpm test` pasa con cobertura >= 85% funciones (API)
- [ ] `pnpm lint` pasa sin errores (Frontend)
- [ ] `pnpm build` pasa sin errores (Frontend)
- [ ] Tests de frontend pasan
- [ ] Smoke test manual del flujo completo
- [ ] GitHub Actions workflow ejecuta correctamente

### 8.2 Test manual post-deploy

1. **CI/CD**: Push a main → verificar que deploy ejecuta correctamente en Cloud Run
2. **Invitaciones**: Admin invita usuario (con rol + sectores seleccionados) → verificar cuenta creada en Auth0 → usuario recibe email de password-reset de Auth0 → configura contraseña → loguea → invitación se marca como accepted → rol y sectores asignados automáticamente → verificar en PermissionsTab que los sectores están activos
3. **Notificaciones**: Verificar que el admin ve notificación de invitación enviada y de usuario aceptado en la campana
4. **Restricción signup**: Intentar registrarse directamente en Auth0 → verificar que está bloqueado
5. **Fallback**: Preguntar algo sin documentos relevantes → verificar respuesta contextual
6. **Structured Output**: Hacer pregunta con documentos → verificar secciones en UI
7. **Release Workflow**: Crear tag `v1.3.0` → verificar que se crea GitHub Release con changelog

---

## 9. Rollback Plan

### API (Cloud Run)

```bash
# Listar revisiones
gcloud run revisions list --service=context-ai-api --region=us-central1

# Rollback a la revisión anterior
gcloud run services update-traffic context-ai-api \
  --region=us-central1 \
  --to-revisions=PREVIOUS_REVISION=100
```

### Frontend (Vercel)

Vercel permite rollback instantáneo desde el dashboard:
1. Vercel Dashboard → Deployments
2. Seleccionar deployment anterior
3. Click "Promote to Production"

---

## 10. Notas de breaking changes

**No hay breaking changes para usuarios existentes.** Cambios importantes a tener en cuenta:

- El workflow de CI/CD es un archivo nuevo
- El campo `responseType` es opcional en la respuesta (backward compatible)
- El campo `structured` es opcional en la respuesta (backward compatible)
- El campo `response` (texto plano) se mantiene para clientes existentes
- El workflow `release.yml` es un archivo nuevo que no afecta workflows existentes
- **Signup público deshabilitado**: Nuevos usuarios solo pueden registrarse vía invitación de admin. Esto es un cambio intencional de comportamiento (no un breaking change técnico)
- Nuevos endpoints de invitaciones y notificaciones son aditivos
- El event bus (`@nestjs/event-emitter`) es una nueva dependencia; los listeners no bloquean flujos existentes
- Nuevas tablas (`invitations`, `invitation_sectors`, `notifications`) requieren ejecutar migración antes del deploy

