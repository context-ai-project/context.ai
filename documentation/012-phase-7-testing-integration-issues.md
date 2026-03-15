---
name: Fase 7 - Consolidación de Tests e Integración
overview: "Descomposición de la Fase 7 del MVP en issues granulares para consolidar la suite de tests, implementar tests E2E completos, accessibility testing, security testing, visual regression, smoke tests, configurar coverage thresholds, y validar criterios de aceptación del MVP."
phase: 7
parent_phase: "009-plan-implementacion-detallado.md"
total_issues: 16
---

# Fase 7: Consolidación de Tests e Integración

Descomposición en issues manejables para consolidar y validar la calidad del MVP mediante testing exhaustivo.

**Nota:** Con TDD, muchos tests ya están escritos en fases anteriores. Esta fase se enfoca en:
- Tests E2E de flujos completos
- Integración entre módulos
- Coverage consolidado
- Validación de criterios de aceptación del MVP

---

## Issue 7.1: Configure Test Infrastructure and Coverage Thresholds

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 4 horas

### Descripción

Configurar infraestructura completa de testing con Jest (backend) y Vitest (frontend), establecer thresholds de coverage mínimos y scripts de testing optimizados.

### Acceptance Criteria

- [ ] Jest configurado con coverage thresholds (80%)
- [ ] Vitest configurado en frontend con coverage
- [ ] Scripts de testing organizados en package.json
- [ ] Configuración separada para unit/integration/e2e tests
- [ ] Coverage reports en formato HTML y LCOV
- [ ] Exclusión de archivos no críticos del coverage
- [ ] Tests corren en paralelo para mejor performance
- [ ] Documentación de comandos de testing

### Files to Create

```
context-ai-api/jest.config.js                    # Configuración Jest principal
context-ai-api/test/jest-unit.json               # Config para unit tests
context-ai-api/test/jest-integration.json        # Config para integration tests
context-ai-api/test/jest-e2e.json                # Config para E2E tests
context-ai-front/vitest.config.ts                # Configuración Vitest
docs/TESTING_INFRASTRUCTURE.md                   # Documentación
```

### Technical Notes

```javascript
// jest.config.js
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: [
    '**/*.ts',
    '!**/*.module.ts',
    '!**/*.interface.ts',
    '!**/*.dto.ts',
    '!**/*.entity.ts',
    '!main.ts',
    '!**/*.config.ts',
  ],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};

// Scripts recomendados
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:unit": "jest --config ./test/jest-unit.json",
    "test:integration": "jest --config ./test/jest-integration.json",
    "test:e2e": "jest --config ./test/jest-e2e.json --runInBand",
    "test:all": "pnpm test:unit && pnpm test:integration && pnpm test:e2e"
  }
}
```

---

## Issue 7.2: Implement E2E Test Helpers and Utilities

**Prioridad:** Alta  
**Dependencias:** 7.1  
**Estimación:** 6 horas

### Descripción

Crear utilidades y helpers reutilizables para tests E2E, incluyendo factories de datos de prueba, funciones de setup/teardown y utilidades comunes.

### Acceptance Criteria

- [ ] TestApp helper para inicializar app completa
- [ ] Database seeding utilities para datos de prueba
- [ ] Auth helpers para obtener tokens de test
- [ ] Factory functions para crear entidades de prueba
- [ ] Cleanup utilities para limpiar BD entre tests
- [ ] Wait utilities para operaciones asíncronas
- [ ] Tests de las utilities mismas
- [ ] Documentación de uso

### Files to Create

```
test/helpers/test-app.helper.ts                  # Helper para app de test
test/helpers/auth-test.helper.ts                 # Helpers de autenticación
test/helpers/database-test.helper.ts             # Helpers de BD
test/helpers/wait.helper.ts                      # Utilidades de espera
test/factories/user.factory.ts                   # Factory de usuarios
test/factories/knowledge-source.factory.ts       # Factory de sources
test/factories/conversation.factory.ts           # Factory de conversaciones
test/fixtures/test-documents.ts                  # Documentos de prueba
test/README.md                                   # Documentación de testing
```

### Technical Notes

```typescript
// test/helpers/test-app.helper.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import { AppModule } from '../../src/app.module';

export class TestAppHelper {
  private app: INestApplication;
  private moduleRef: TestingModule;

  async createTestApp(): Promise<INestApplication> {
    this.moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    this.app = this.moduleRef.createNestApplication();
    this.app.useGlobalPipes(new ValidationPipe());
    await this.app.init();

    return this.app;
  }

  async closeApp(): Promise<void> {
    await this.app?.close();
  }

  getApp(): INestApplication {
    return this.app;
  }
}

// test/helpers/auth-test.helper.ts
export class AuthTestHelper {
  static async getTestAccessToken(): Promise<string> {
    // Mock token o token real de Auth0 test tenant
    return 'test-access-token';
  }

  static async createTestUser(roles: string[] = ['user']): Promise<User> {
    // Crear usuario de prueba con roles específicos
  }
}

// test/helpers/wait.helper.ts
export async function waitForProcessing(
  sourceId: string,
  maxWaitTime: number = 30000,
): Promise<void> {
  const startTime = Date.now();
  while (Date.now() - startTime < maxWaitTime) {
    const source = await getSource(sourceId);
    if (source.status === 'processed') {
      return;
    }
    await sleep(1000);
  }
  throw new Error('Timeout waiting for processing');
}
```

---

## Issue 7.3: Implement Complete RAG Flow E2E Test (Backend)

**Prioridad:** Alta  
**Dependencias:** 7.2  
**Estimación:** 8 horas

### Descripción

Crear test E2E completo que valida el flujo RAG desde la carga de documento hasta la consulta y respuesta del asistente, incluyendo validación de fuentes.

### Acceptance Criteria

- [ ] Test: Subir documento PDF completo
- [ ] Test: Documento se procesa y genera embeddings
- [ ] Test: Consulta al asistente retorna respuesta
- [ ] Test: Respuesta incluye fuentes correctas
- [ ] Test: Fuentes coinciden con documento subido
- [ ] Test: Score de similitud es adecuado (>0.7)
- [ ] Test: Respuesta contiene información del documento
- [ ] Test completo corre en <60 segundos

### Files to Create

```
test/e2e/flows/complete-rag-flow.e2e.spec.ts     # Test principal RAG
test/fixtures/documents/manual-vacaciones.pdf    # Documento de prueba
test/fixtures/documents/manual-tech.md           # Documento MD de prueba
test/fixtures/expected-responses.ts              # Respuestas esperadas
```

### Technical Notes

```typescript
describe('Complete RAG Flow (E2E)', () => {
  let app: INestApplication;
  let authToken: string;
  let testHelper: TestAppHelper;

  beforeAll(async () => {
    testHelper = new TestAppHelper();
    app = await testHelper.createTestApp();
    authToken = await AuthTestHelper.getTestAccessToken();
  });

  afterAll(async () => {
    await testHelper.closeApp();
  });

  it('should complete full workflow: upload -> process -> query -> response', async () => {
    // 1. Subir documento
    const uploadResponse = await request(app.getHttpServer())
      .post('/api/knowledge/sources')
      .set('Authorization', `Bearer ${authToken}`)
      .attach('file', 'test/fixtures/documents/manual-vacaciones.pdf')
      .field('sectorId', 'rrhh')
      .field('title', 'Manual de Vacaciones')
      .expect(201);

    const sourceId = uploadResponse.body.id;
    expect(sourceId).toBeDefined();

    // 2. Esperar procesamiento
    await waitForProcessing(sourceId);

    // 3. Verificar que se generaron fragmentos
    const fragments = await getFragmentsBySourceId(sourceId);
    expect(fragments.length).toBeGreaterThan(0);
    expect(fragments[0].embedding).toBeDefined();

    // 4. Hacer consulta
    const queryResponse = await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        message: '¿Cómo pido vacaciones?',
        sectorId: 'rrhh',
      })
      .expect(200);

    // 5. Validar respuesta
    expect(queryResponse.body.response).toBeDefined();
    expect(queryResponse.body.response).toContain('15 días');
    expect(queryResponse.body.sources).toHaveLength(5);
    expect(queryResponse.body.sources[0].sourceId).toBe(sourceId);
    expect(queryResponse.body.sources[0].similarity).toBeGreaterThan(0.7);
    expect(queryResponse.body.conversationId).toBeDefined();
  });
});
```

---

## Issue 7.4: Implement Sector Isolation E2E Test

**Prioridad:** Alta  
**Dependencias:** 7.3  
**Estimación:** 5 horas

### Descripción

Crear tests E2E que validan que la información está correctamente aislada por sectores y que los usuarios solo pueden acceder a información de sus sectores autorizados.

### Acceptance Criteria

- [ ] Test: Usuario solo ve documentos de su sector
- [ ] Test: Consulta no retorna info de otros sectores
- [ ] Test: Vector search filtra correctamente por sector
- [ ] Test: Intento de acceso a otro sector retorna 403
- [ ] Test: Admin puede acceder a múltiples sectores
- [ ] Test: Cambio de sector funciona correctamente
- [ ] Coverage de casos edge (sector inexistente, etc.)

### Files to Create

```
test/e2e/flows/sector-isolation.e2e.spec.ts      # Tests de aislamiento
test/helpers/sector-test.helper.ts               # Helpers específicos
```

### Technical Notes

```typescript
describe('Sector Isolation (E2E)', () => {
  it('should not return information from other sectors', async () => {
    // 1. Subir documento en sector "Tech"
    const techDoc = await uploadDocument({
      file: 'tech-manual.pdf',
      sectorId: 'tech',
      title: 'Tech Manual',
    });

    // 2. Subir documento en sector "RRHH"
    const rrhhDoc = await uploadDocument({
      file: 'rrhh-manual.pdf',
      sectorId: 'rrhh',
      title: 'RRHH Manual',
    });

    await waitForProcessing(techDoc.id);
    await waitForProcessing(rrhhDoc.id);

    // 3. Usuario con acceso solo a RRHH hace consulta
    const rrhhUser = await createUserWithSectorAccess(['rrhh']);
    const rrhhToken = await getTokenForUser(rrhhUser);

    const response = await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${rrhhToken}`)
      .send({
        message: 'contenido específico del tech manual',
        sectorId: 'rrhh',
      })
      .expect(200);

    // 4. Verificar que no retorna info de Tech
    const sourceIds = response.body.sources.map(s => s.sourceId);
    expect(sourceIds).not.toContain(techDoc.id);
    expect(sourceIds).toContain(rrhhDoc.id);
  });

  it('should deny access to documents from unauthorized sector', async () => {
    const techUser = await createUserWithSectorAccess(['tech']);
    const techToken = await getTokenForUser(techUser);

    await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${techToken}`)
      .send({
        message: 'test query',
        sectorId: 'rrhh', // Intenta acceder a sector no autorizado
      })
      .expect(403);
  });
});
```

---

## Issue 7.5: Implement Authentication & Authorization E2E Tests

**Prioridad:** Alta  
**Dependencias:** 7.2  
**Estimación:** 6 horas

### Descripción

Crear suite completa de tests E2E para validar flujos de autenticación con Auth0 y autorización basada en roles y permisos.

### Acceptance Criteria

- [ ] Test: Login completo con Auth0 (mock o test tenant)
- [ ] Test: Token JWT válido permite acceso
- [ ] Test: Token inválido retorna 401
- [ ] Test: Token expirado retorna 401
- [ ] Test: Usuario sin permiso retorna 403
- [ ] Test: Usuario se crea en BD en primer login
- [ ] Test: Roles se asignan correctamente
- [ ] Test: Permisos se validan correctamente

### Files to Create

```
test/e2e/auth/authentication-flow.e2e.spec.ts    # Tests de auth
test/e2e/auth/authorization-flow.e2e.spec.ts     # Tests de authz
test/e2e/auth/jwt-validation.e2e.spec.ts         # Tests de JWT
test/mocks/auth0-mock.ts                         # Mock de Auth0
```

### Technical Notes

```typescript
describe('Authentication Flow (E2E)', () => {
  it('should authenticate user with valid Auth0 token', async () => {
    const validToken = await getValidAuth0Token();

    const response = await request(app.getHttpServer())
      .get('/api/users/me')
      .set('Authorization', `Bearer ${validToken}`)
      .expect(200);

    expect(response.body.email).toBeDefined();
    expect(response.body.auth0UserId).toBeDefined();
  });

  it('should create user in database on first login', async () => {
    const newUserToken = await getTokenForNewUser();

    // Primera request crea el usuario
    await request(app.getHttpServer())
      .get('/api/users/me')
      .set('Authorization', `Bearer ${newUserToken}`)
      .expect(200);

    // Verificar que se creó en BD
    const user = await userRepository.findByAuth0Id(extractSubFromToken(newUserToken));
    expect(user).toBeDefined();
    expect(user.roles).toContainEqual(expect.objectContaining({ name: 'user' }));
  });

  it('should reject request with invalid token', async () => {
    await request(app.getHttpServer())
      .get('/api/users/me')
      .set('Authorization', 'Bearer invalid-token')
      .expect(401);
  });
});

describe('Authorization Flow (E2E)', () => {
  it('should allow access with correct permission', async () => {
    const adminUser = await createUserWithRole('admin');
    const adminToken = await getTokenForUser(adminUser);

    await request(app.getHttpServer())
      .post('/api/knowledge/sources')
      .set('Authorization', `Bearer ${adminToken}`)
      .attach('file', 'test.pdf')
      .field('sectorId', 'tech')
      .expect(201);
  });

  it('should deny access without required permission', async () => {
    const viewerUser = await createUserWithRole('viewer');
    const viewerToken = await getTokenForUser(viewerUser);

    await request(app.getHttpServer())
      .post('/api/knowledge/sources')
      .set('Authorization', `Bearer ${viewerToken}`)
      .attach('file', 'test.pdf')
      .field('sectorId', 'tech')
      .expect(403);
  });
});
```

---

## Issue 7.6: Implement Frontend Component Tests

**Prioridad:** Alta  
**Dependencias:** 7.1  
**Estimación:** 8 horas

### Descripción

Crear tests completos para todos los componentes React del frontend usando Vitest y Testing Library, con coverage mínimo de 80%.

### Acceptance Criteria

- [ ] Tests para todos los componentes de chat
- [ ] Tests para componentes de autenticación
- [ ] Tests para componentes UI reutilizables
- [ ] Mocks de API y hooks personalizados
- [ ] Tests de interacciones de usuario
- [ ] Tests de estados de loading y error
- [ ] Coverage >= 80% en componentes
- [ ] Tests corren en <30 segundos

### Files to Create

```
src/components/chat/MessageList.test.tsx         # Tests de lista
src/components/chat/Message.test.tsx             # Tests de mensaje
src/components/chat/MessageInput.test.tsx        # Tests de input
src/components/chat/SourceCard.test.tsx          # Tests de fuentes
src/components/auth/LoginButton.test.tsx         # Tests de login
src/lib/test-utils.tsx                           # Utilidades de testing
src/lib/mocks/api-mocks.ts                       # Mocks de API
```

### Technical Notes

```typescript
// MessageList.test.tsx
import { render, screen } from '@testing-library/react';
import { MessageList } from './MessageList';

describe('MessageList', () => {
  it('should render user and assistant messages', () => {
    const messages = [
      { 
        id: '1',
        role: 'user', 
        content: '¿Cómo pido vacaciones?',
        timestamp: new Date(),
      },
      { 
        id: '2',
        role: 'assistant', 
        content: 'Debes pedirlas con 15 días de antelación',
        sources: [{ 
          title: 'Manual RRHH', 
          page: 5,
          similarity: 0.95,
        }],
        timestamp: new Date(),
      }
    ];

    render(<MessageList messages={messages} />);

    expect(screen.getByText('¿Cómo pido vacaciones?')).toBeInTheDocument();
    expect(screen.getByText(/15 días/)).toBeInTheDocument();
    expect(screen.getByText('Manual RRHH')).toBeInTheDocument();
  });

  it('should show empty state when no messages', () => {
    render(<MessageList messages={[]} />);
    expect(screen.getByText(/Start a conversation/i)).toBeInTheDocument();
  });

  it('should auto-scroll to latest message', () => {
    const scrollIntoViewMock = jest.fn();
    HTMLElement.prototype.scrollIntoView = scrollIntoViewMock;

    const messages = [/* ... */];
    render(<MessageList messages={messages} />);

    expect(scrollIntoViewMock).toHaveBeenCalled();
  });
});

// MessageInput.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('MessageInput', () => {
  it('should send message on Enter key', async () => {
    const onSendMock = jest.fn();
    render(<MessageInput onSend={onSendMock} />);

    const input = screen.getByRole('textbox');
    await userEvent.type(input, 'Test message{Enter}');

    await waitFor(() => {
      expect(onSendMock).toHaveBeenCalledWith('Test message');
    });
  });

  it('should add new line on Shift+Enter', async () => {
    render(<MessageInput onSend={jest.fn()} />);

    const input = screen.getByRole('textbox');
    await userEvent.type(input, 'Line 1{Shift>}{Enter}{/Shift}Line 2');

    expect(input).toHaveValue('Line 1\nLine 2');
  });

  it('should disable input while loading', () => {
    render(<MessageInput onSend={jest.fn()} isLoading={true} />);

    const input = screen.getByRole('textbox');
    const button = screen.getByRole('button');

    expect(input).toBeDisabled();
    expect(button).toBeDisabled();
  });
});
```

---

## Issue 7.7: Implement Frontend E2E Tests with Playwright

**Prioridad:** Alta  
**Dependencias:** 7.6  
**Estimación:** 10 horas

### Descripción

Crear suite completa de tests E2E para el frontend usando Playwright, validando flujos de usuario completos en navegador real.

### Acceptance Criteria

- [ ] Test: Flujo completo de login
- [ ] Test: Enviar mensaje y recibir respuesta
- [ ] Test: Ver fuentes de respuesta
- [ ] Test: Navegación entre páginas
- [ ] Test: Manejo de errores visual
- [ ] Test: Responsive en mobile
- [ ] Tests corren en múltiples navegadores
- [ ] Screenshots de fallos automáticos

### Files to Create

```
tests/e2e/chat-flow.spec.ts                      # Test principal de chat
tests/e2e/auth-flow.spec.ts                      # Test de autenticación
tests/e2e/error-handling.spec.ts                 # Test de errores
tests/helpers/playwright-helpers.ts              # Helpers para Playwright
playwright.config.ts                             # Configuración (actualizar)
```

### Technical Notes

```typescript
// tests/e2e/chat-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Chat Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Mock de Auth0 o usar test user
    await page.goto('/chat');
  });

  test('user can send message and receive response', async ({ page }) => {
    // Escribir mensaje
    const input = page.locator('[data-testid="message-input"]');
    await input.fill('¿Cómo pido vacaciones?');

    // Enviar
    const sendButton = page.locator('[data-testid="send-button"]');
    await sendButton.click();

    // Verificar mensaje del usuario aparece
    await expect(page.locator('text=¿Cómo pido vacaciones?')).toBeVisible();

    // Verificar loading indicator
    await expect(page.locator('[data-testid="typing-indicator"]')).toBeVisible();

    // Esperar respuesta del asistente
    await expect(page.locator('[data-testid="assistant-message"]')).toBeVisible({
      timeout: 10000,
    });

    // Verificar fuentes
    const sourceCards = page.locator('[data-testid="source-card"]');
    await expect(sourceCards).toHaveCount(5);
  });

  test('user can expand source cards', async ({ page }) => {
    // Enviar mensaje y esperar respuesta
    await sendMessageAndWaitForResponse(page, 'test query');

    // Hacer clic en primera fuente
    const firstSource = page.locator('[data-testid="source-card"]').first();
    await firstSource.click();

    // Verificar que se expande y muestra contenido
    await expect(page.locator('[data-testid="source-content"]')).toBeVisible();
  });

  test('should handle network errors gracefully', async ({ page }) => {
    // Simular error de red
    await page.route('**/api/chat/query', route => route.abort());

    await page.locator('[data-testid="message-input"]').fill('test');
    await page.locator('[data-testid="send-button"]').click();

    // Verificar mensaje de error
    await expect(page.locator('text=Error sending message')).toBeVisible();
  });

  test('should work on mobile viewport', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });

    // Verificar que UI es responsive
    await expect(page.locator('[data-testid="chat-container"]')).toBeVisible();

    // Enviar mensaje en mobile
    await sendMessageAndWaitForResponse(page, 'mobile test');
    await expect(page.locator('[data-testid="assistant-message"]')).toBeVisible();
  });
});
```

---

## Issue 7.8: Implement API Contract Tests

**Prioridad:** Media  
**Dependencias:** 7.1  
**Estimación:** 6 horas

### Descripción

Crear tests que validan que los contratos de API (DTOs, responses, status codes) se mantienen consistentes y cumplen con la especificación OpenAPI/Swagger.

### Acceptance Criteria

- [ ] Tests validan estructura de DTOs
- [ ] Tests validan status codes correctos
- [ ] Tests validan error responses
- [ ] Tests validan headers requeridos
- [ ] Tests validan tipos de contenido
- [ ] Validación contra schema OpenAPI
- [ ] Tests de versioning de API
- [ ] Documentación de contratos

### Files to Create

```
test/integration/api/knowledge-api-contract.spec.ts   # Contratos Knowledge
test/integration/api/chat-api-contract.spec.ts        # Contratos Chat
test/integration/api/auth-api-contract.spec.ts        # Contratos Auth
test/helpers/api-contract-validator.ts                # Validador de contratos
```

### Technical Notes

```typescript
describe('Chat API Contract', () => {
  it('POST /api/chat/query should return correct response structure', async () => {
    const response = await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${token}`)
      .send({
        message: 'test query',
        sectorId: 'test-sector',
      })
      .expect(200)
      .expect('Content-Type', /json/);

    // Validar estructura de respuesta
    expect(response.body).toMatchObject({
      response: expect.any(String),
      sources: expect.arrayContaining([
        expect.objectContaining({
          id: expect.any(String),
          content: expect.any(String),
          sourceId: expect.any(String),
          similarity: expect.any(Number),
        }),
      ]),
      conversationId: expect.any(String),
      timestamp: expect.any(String),
    });

    // Validar tipos
    expect(typeof response.body.response).toBe('string');
    expect(Array.isArray(response.body.sources)).toBe(true);
    expect(response.body.sources.length).toBeLessThanOrEqual(5);
  });

  it('should return 400 for invalid request', async () => {
    const response = await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${token}`)
      .send({
        // Missing required fields
      })
      .expect(400);

    expect(response.body).toMatchObject({
      statusCode: 400,
      message: expect.any(Array),
      error: 'Bad Request',
    });
  });
});
```

---

## Issue 7.9: Implement Performance and Load Tests

**Prioridad:** Media  
**Dependencias:** 7.3  
**Estimación:** 8 horas

### Descripción

Crear tests de performance y carga para validar que el sistema mantiene tiempos de respuesta aceptables bajo carga y identificar bottlenecks.

### Acceptance Criteria

- [ ] Tests de tiempo de respuesta de endpoints
- [ ] Tests de carga concurrente (10, 50, 100 usuarios)
- [ ] Tests de performance de vector search
- [ ] Tests de tiempo de procesamiento de documentos
- [ ] Benchmarks de generación de embeddings
- [ ] Identificación de queries lentas en BD
- [ ] Reports de performance generados
- [ ] Métricas comparables entre runs

### Files to Create

```
test/performance/chat-query-performance.spec.ts      # Performance de chat
test/performance/vector-search-performance.spec.ts   # Performance de búsqueda
test/performance/document-processing.spec.ts         # Performance de ingesta
test/performance/load-testing.spec.ts                # Tests de carga
test/helpers/performance-metrics.ts                  # Utilidades de métricas
```

### Technical Notes

```typescript
describe('Chat Query Performance', () => {
  it('should respond to query in less than 3 seconds', async () => {
    const startTime = Date.now();

    await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${token}`)
      .send({
        message: 'test query',
        sectorId: 'test',
      })
      .expect(200);

    const duration = Date.now() - startTime;
    expect(duration).toBeLessThan(3000);
  });

  it('should handle 10 concurrent queries', async () => {
    const queries = Array(10).fill(null).map((_, i) => 
      request(app.getHttpServer())
        .post('/api/chat/query')
        .set('Authorization', `Bearer ${token}`)
        .send({
          message: `concurrent query ${i}`,
          sectorId: 'test',
        })
    );

    const startTime = Date.now();
    const results = await Promise.all(queries);
    const duration = Date.now() - startTime;

    // Todas las queries exitosas
    results.forEach(result => {
      expect(result.status).toBe(200);
    });

    // Tiempo total razonable
    expect(duration).toBeLessThan(10000);
  });
});

describe('Vector Search Performance', () => {
  it('should complete similarity search in less than 500ms', async () => {
    const queryEmbedding = await generateTestEmbedding();

    const startTime = Date.now();
    const results = await vectorSearchService.findSimilar(
      queryEmbedding,
      'test-sector',
      5,
    );
    const duration = Date.now() - startTime;

    expect(results).toHaveLength(5);
    expect(duration).toBeLessThan(500);
  });
});
```

---

## Issue 7.10: Implement Test Data Management and Fixtures

**Prioridad:** Media  
**Dependencias:** 7.2  
**Estimación:** 5 horas

### Descripción

Crear sistema robusto de gestión de datos de prueba con fixtures, seeders y cleanup automatizado para mantener tests determinísticos y rápidos.

### Acceptance Criteria

- [ ] Database seeders para diferentes escenarios
- [ ] Fixtures organizadas por módulo
- [ ] Cleanup automático entre tests
- [ ] Transactions para aislar tests
- [ ] Factory builders con datos realistas
- [ ] Documentación de fixtures disponibles
- [ ] Tests de los seeders mismos
- [ ] Performance: setup < 1 segundo

### Files to Create

```
test/fixtures/database-seeds.ts                  # Seeds principales
test/fixtures/knowledge/sources.fixture.ts       # Fixtures de sources
test/fixtures/interaction/conversations.fixture.ts  # Fixtures de conversaciones
test/fixtures/auth/users.fixture.ts              # Fixtures de usuarios
test/helpers/database-cleaner.ts                 # Limpieza de BD
test/builders/entity-builders.ts                 # Builders de entidades
```

### Technical Notes

```typescript
// test/fixtures/database-seeds.ts
export class DatabaseSeeds {
  static async seedBasicData(dataSource: DataSource): Promise<void> {
    // Seed roles
    await this.seedRoles(dataSource);
    
    // Seed sectors
    await this.seedSectors(dataSource);
    
    // Seed test users
    await this.seedUsers(dataSource);
  }

  private static async seedRoles(dataSource: DataSource): Promise<void> {
    const roleRepository = dataSource.getRepository(Role);
    
    const roles = [
      { name: 'admin', permissions: ['*'] },
      { name: 'user', permissions: ['knowledge:read', 'chat:query'] },
      { name: 'viewer', permissions: ['knowledge:read'] },
    ];

    await roleRepository.save(roles);
  }
}

// test/helpers/database-cleaner.ts
export class DatabaseCleaner {
  static async cleanAll(dataSource: DataSource): Promise<void> {
    const entities = dataSource.entityMetadatas;

    for (const entity of entities) {
      const repository = dataSource.getRepository(entity.name);
      await repository.query(`TRUNCATE TABLE "${entity.tableName}" CASCADE;`);
    }
  }

  static async cleanTable(
    dataSource: DataSource,
    tableName: string,
  ): Promise<void> {
    await dataSource.query(`TRUNCATE TABLE "${tableName}" CASCADE;`);
  }
}

// Usage en tests
beforeEach(async () => {
  await DatabaseSeeds.seedBasicData(dataSource);
});

afterEach(async () => {
  await DatabaseCleaner.cleanAll(dataSource);
});
```

---

## Issue 7.11: Consolidate Test Reports and CI/CD Integration

**Prioridad:** Alta  
**Dependencias:** Todas las anteriores  
**Estimación:** 6 horas

### Descripción

Configurar generación consolidada de reportes de tests, integración con CI/CD, y badges de coverage para visualización de calidad.

### Acceptance Criteria

- [ ] Reports HTML de coverage generados
- [ ] Reports JUnit para CI/CD
- [ ] Badge de coverage en README
- [ ] Tests corren automáticamente en CI
- [ ] Bloqueo de merge si tests fallan
- [ ] Bloqueo de merge si coverage < 80%
- [ ] Notificaciones de fallos de tests
- [ ] Dashboards de métricas de tests

### Files to Create

```
.github/workflows/backend-tests.yml              # Workflow backend tests
.github/workflows/frontend-tests.yml             # Workflow frontend tests
scripts/generate-test-reports.sh                 # Script de reports
docs/CI_CD_TESTING.md                            # Documentación CI/CD
```

### Technical Notes

```yaml
# .github/workflows/backend-tests.yml
name: Backend Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8
      
      - name: Install dependencies
        run: pnpm install
        working-directory: ./context-ai-api
      
      - name: Run unit tests
        run: pnpm test:unit
        working-directory: ./context-ai-api
      
      - name: Run integration tests
        run: pnpm test:integration
        working-directory: ./context-ai-api
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
      
      - name: Run E2E tests
        run: pnpm test:e2e
        working-directory: ./context-ai-api
      
      - name: Generate coverage report
        run: pnpm test:cov
        working-directory: ./context-ai-api
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./context-ai-api/coverage/lcov.info
          flags: backend
      
      - name: Check coverage threshold
        run: |
          if [ $(cat coverage/coverage-summary.json | jq '.total.lines.pct') -lt 80 ]; then
            echo "Coverage is below 80%"
            exit 1
          fi
```

---

## Issue 7.12: MVP Validation Checklist and Acceptance Testing

**Prioridad:** Alta  
**Dependencias:** Todas las anteriores  
**Estimación:** 8 horas

### Descripción

Ejecutar checklist completo de validación del MVP, verificar todos los criterios de aceptación, y crear reporte final de calidad y completitud.

### Acceptance Criteria

- [ ] Todos los criterios de aceptación del MVP validados
- [ ] Suite completa de tests pasa (unit + integration + E2E)
- [ ] Coverage >= 80% en backend y frontend
- [ ] Linter pasa sin errores (pnpm lint)
- [ ] Build exitoso en ambos proyectos (pnpm build)
- [ ] Performance dentro de límites aceptables
- [ ] Seguridad: no hay secretos expuestos
- [ ] Documentación completa y actualizada
- [ ] Reporte de validación generado

### Files to Create

```
docs/MVP_VALIDATION_REPORT.md                    # Reporte de validación
scripts/validate-mvp.sh                          # Script de validación
test/e2e/mvp-acceptance/mvp-criteria.e2e.spec.ts # Tests de criterios MVP
checklists/mvp-validation-checklist.md           # Checklist manual
```

### Technical Notes

```bash
#!/bin/bash
# scripts/validate-mvp.sh

echo "🚀 Starting MVP Validation..."

# 1. Backend Tests
echo "Running backend tests..."
cd context-ai-api
pnpm test:all || exit 1
pnpm test:cov || exit 1

# 2. Frontend Tests
echo "Running frontend tests..."
cd ../context-ai-front
pnpm test || exit 1
pnpm test:e2e || exit 1

# 3. Linting
echo "Running linters..."
cd ../context-ai-api
pnpm lint || exit 1
cd ../context-ai-front
pnpm lint || exit 1

# 4. Builds
echo "Testing builds..."
cd ../context-ai-api
pnpm build || exit 1
cd ../context-ai-front
pnpm build || exit 1

# 5. Coverage Check
echo "Checking coverage thresholds..."
cd ../context-ai-api
COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
if (( $(echo "$COVERAGE < 80" | bc -l) )); then
  echo "❌ Coverage is below 80%: $COVERAGE%"
  exit 1
fi

echo "✅ MVP Validation Complete!"
echo "Coverage: $COVERAGE%"
```

```typescript
// test/e2e/mvp-acceptance/mvp-criteria.e2e.spec.ts
describe('MVP Acceptance Criteria', () => {
  describe('UC2: Upload Documents', () => {
    it('✓ Admin can upload PDF document to a sector', async () => {
      // Test implementation
    });

    it('✓ Admin can upload MD document to a sector', async () => {
      // Test implementation
    });

    it('✓ Document is processed and embeddings are generated', async () => {
      // Test implementation
    });
  });

  describe('UC5: Chat with RAG', () => {
    it('✓ User can ask questions to the assistant', async () => {
      // Test implementation
    });

    it('✓ Assistant responds based on uploaded documents', async () => {
      // Test implementation
    });

    it('✓ Response shows which sources were used', async () => {
      // Test implementation
    });

    it('✓ Sources show similarity score and document metadata', async () => {
      // Test implementation
    });
  });

  describe('Authentication', () => {
    it('✓ User can authenticate with Auth0', async () => {
      // Test implementation
    });

    it('✓ Protected routes require authentication', async () => {
      // Test implementation
    });

    it('✓ Access token is validated on backend', async () => {
      // Test implementation
    });
  });

  describe('Authorization', () => {
    it('✓ Users can only access their authorized sectors', async () => {
      // Test implementation
    });

    it('✓ Role-based permissions work correctly', async () => {
      // Test implementation
    });
  });

  describe('Sector Isolation', () => {
    it('✓ Information is not leaked between sectors', async () => {
      // Test implementation
    });
  });
});
```

---

## Issue 7.13: Implement Accessibility (a11y) Testing

**Prioridad:** Alta  
**Dependencias:** 7.6, 7.7  
**Estimación:** 6 horas

### Descripción

Implementar suite de tests de accesibilidad usando axe-core para garantizar WCAG 2.1 AA compliance y mejorar la experiencia para usuarios con discapacidades.

### Acceptance Criteria

- [ ] Tests automáticos con axe-core en componentes
- [ ] Tests de navegación por teclado
- [ ] Tests de lectores de pantalla (ARIA labels)
- [ ] Contraste de colores validado
- [ ] Focus management verificado
- [ ] Tests E2E con Playwright axe
- [ ] Score de Lighthouse Accessibility >= 90
- [ ] Documentación de mejoras de a11y

### Files to Create

```
test/accessibility/components-a11y.spec.ts    # Tests de componentes
test/accessibility/keyboard-navigation.spec.ts # Tests de teclado
test/e2e/accessibility/chat-a11y.spec.ts      # Tests E2E a11y
lib/test-utils/a11y-helpers.ts                # Utilidades a11y
docs/ACCESSIBILITY_GUIDELINES.md              # Guía de accesibilidad
```

### Technical Notes

```typescript
// test/accessibility/components-a11y.spec.ts
import { render } from '@testing-library/react';
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

describe('MessageList Accessibility', () => {
  it('should not have accessibility violations', async () => {
    const { container } = render(<MessageList messages={mockMessages} />);
    const results = await axe(container);
    expect(results).toHaveNoViolations();
  });
});

// test/e2e/accessibility/chat-a11y.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('chat page should be accessible', async ({ page }) => {
  await page.goto('/chat');
  
  const accessibilityScanResults = await new AxeBuilder({ page }).analyze();
  
  expect(accessibilityScanResults.violations).toEqual([]);
});

test('keyboard navigation should work', async ({ page }) => {
  await page.goto('/chat');
  
  // Tab through interactive elements
  await page.keyboard.press('Tab');
  await expect(page.locator('[data-testid="message-input"]')).toBeFocused();
  
  await page.keyboard.press('Tab');
  await expect(page.locator('[data-testid="send-button"]')).toBeFocused();
});

// Dependencias
pnpm add -D jest-axe @axe-core/playwright
```

---

## Issue 7.14: Implement Security Testing

**Prioridad:** Alta  
**Dependencias:** 7.3, 7.5  
**Estimación:** 8 horas

### Descripción

Crear suite de tests de seguridad para validar protección contra vulnerabilidades comunes (SQL Injection, XSS, CSRF, JWT tampering, rate limiting).

### Acceptance Criteria

- [ ] Tests de SQL Injection en endpoints
- [ ] Tests de XSS en inputs
- [ ] Tests de CSRF protection
- [ ] Tests de JWT tampering y token inválidos
- [ ] Tests de rate limiting efectivo
- [ ] Tests de autorización (bypass attempts)
- [ ] Tests de input validation
- [ ] Security audit report generado

### Files to Create

```
test/security/sql-injection.spec.ts           # Tests SQL Injection
test/security/xss-protection.spec.ts          # Tests XSS
test/security/csrf-protection.spec.ts         # Tests CSRF
test/security/jwt-security.spec.ts            # Tests JWT
test/security/rate-limiting.spec.ts           # Tests rate limiting
test/security/authorization-bypass.spec.ts    # Tests authz bypass
scripts/security-audit.sh                     # Script de auditoría
docs/SECURITY_TEST_REPORT.md                  # Reporte
```

### Technical Notes

```typescript
// test/security/sql-injection.spec.ts
describe('SQL Injection Protection', () => {
  it('should reject SQL injection in search queries', async () => {
    const maliciousInput = "'; DROP TABLE users; --";
    
    await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${token}`)
      .send({
        message: maliciousInput,
        sectorId: 'test',
      })
      .expect(400); // Should be rejected by validation
  });

  it('should sanitize database queries', async () => {
    const maliciousInput = "1' OR '1'='1";
    
    const response = await request(app.getHttpServer())
      .get(`/api/knowledge/sources?search=${maliciousInput}`)
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    // Verify it doesn't return all records
    expect(response.body.length).toBeLessThan(100);
  });
});

// test/security/xss-protection.spec.ts
describe('XSS Protection', () => {
  it('should sanitize HTML in user inputs', async () => {
    const xssPayload = '<script>alert("XSS")</script>';
    
    const response = await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${token}`)
      .send({
        message: xssPayload,
        sectorId: 'test',
      });

    // Verify HTML is escaped in response
    expect(response.body.response).not.toContain('<script>');
  });
});

// test/security/jwt-security.spec.ts
describe('JWT Security', () => {
  it('should reject tampered JWT tokens', async () => {
    const tamperedToken = validToken.slice(0, -5) + 'xxxxx';
    
    await request(app.getHttpServer())
      .get('/api/users/me')
      .set('Authorization', `Bearer ${tamperedToken}`)
      .expect(401);
  });

  it('should reject expired tokens', async () => {
    const expiredToken = await generateExpiredToken();
    
    await request(app.getHttpServer())
      .get('/api/users/me')
      .set('Authorization', `Bearer ${expiredToken}`)
      .expect(401);
  });
});

// test/security/rate-limiting.spec.ts
describe('Rate Limiting', () => {
  it('should block after exceeding rate limit', async () => {
    // Send requests up to the limit
    for (let i = 0; i < 10; i++) {
      await request(app.getHttpServer())
        .post('/api/chat/query')
        .set('Authorization', `Bearer ${token}`)
        .send({ message: 'test', sectorId: 'test' })
        .expect(200);
    }

    // Next request should be rate limited
    await request(app.getHttpServer())
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${token}`)
      .send({ message: 'test', sectorId: 'test' })
      .expect(429);
  });
});
```

---

## Issue 7.15: Implement Visual Regression Testing

**Prioridad:** Media  
**Dependencias:** 7.7  
**Estimación:** 5 horas

### Descripción

Configurar visual regression testing con Playwright para detectar cambios no intencionados en la UI y mantener consistencia visual.

### Acceptance Criteria

- [ ] Visual snapshots configurados en Playwright
- [ ] Snapshots de componentes principales
- [ ] Snapshots en diferentes viewports (mobile, tablet, desktop)
- [ ] Snapshots en diferentes estados (loading, error, empty)
- [ ] Comparación automática en CI
- [ ] Documentación de cómo actualizar snapshots
- [ ] Tests pasan en CI

### Files to Create

```
tests/visual/chat-visual.spec.ts              # Tests visuales de chat
tests/visual/auth-visual.spec.ts              # Tests visuales de auth
tests/visual/components-visual.spec.ts        # Tests de componentes
playwright.config.ts                          # Config (actualizar)
.gitignore                                    # Ignorar snapshots locales
```

### Technical Notes

```typescript
// tests/visual/chat-visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Chat Visual Regression', () => {
  test('chat page should match snapshot', async ({ page }) => {
    await page.goto('/chat');
    await page.waitForLoadState('networkidle');
    
    await expect(page).toHaveScreenshot('chat-page.png');
  });

  test('message list with messages should match snapshot', async ({ page }) => {
    await page.goto('/chat');
    await sendTestMessage(page, 'Test message');
    await page.waitForSelector('[data-testid="assistant-message"]');
    
    await expect(page.locator('[data-testid="message-list"]'))
      .toHaveScreenshot('message-list-with-content.png');
  });

  test('empty state should match snapshot', async ({ page }) => {
    await page.goto('/chat');
    
    await expect(page.locator('[data-testid="empty-state"]'))
      .toHaveScreenshot('chat-empty-state.png');
  });
});

test.describe('Responsive Visual Regression', () => {
  test('chat page on mobile should match snapshot', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/chat');
    
    await expect(page).toHaveScreenshot('chat-page-mobile.png');
  });

  test('chat page on tablet should match snapshot', async ({ page }) => {
    await page.setViewportSize({ width: 768, height: 1024 });
    await page.goto('/chat');
    
    await expect(page).toHaveScreenshot('chat-page-tablet.png');
  });
});

// playwright.config.ts
export default defineConfig({
  expect: {
    toHaveScreenshot: {
      maxDiffPixels: 100,
      threshold: 0.2,
    },
  },
  // ...
});
```

---

## Issue 7.16: Implement Smoke Tests for Production

**Prioridad:** Alta  
**Dependencias:** 7.12  
**Estimación:** 4 horas

### Descripción

Crear suite mínima de smoke tests que se ejecutan en producción para validar que los servicios críticos están funcionando correctamente después de cada deployment.

### Acceptance Criteria

- [ ] Health check endpoint implementado
- [ ] Database connectivity test
- [ ] Auth0 connectivity test
- [ ] Google AI API connectivity test
- [ ] Tests críticos de cada módulo
- [ ] Script ejecutable en producción
- [ ] Alertas si smoke tests fallan
- [ ] Documentación de smoke tests

### Files to Create

```
test/smoke/health-check.smoke.ts              # Health checks
test/smoke/critical-paths.smoke.ts            # Paths críticos
scripts/run-smoke-tests.sh                    # Script de ejecución
src/health/health.controller.ts               # Health endpoint
docs/SMOKE_TESTS.md                           # Documentación
```

### Technical Notes

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, TypeOrmHealthIndicator } from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.checkAuth0(),
      () => this.checkGoogleAI(),
    ]);
  }

  private async checkAuth0() {
    // Verify Auth0 JWKS endpoint is reachable
    const response = await fetch(`https://${process.env.AUTH0_DOMAIN}/.well-known/jwks.json`);
    return { auth0: { status: response.ok ? 'up' : 'down' } };
  }

  private async checkGoogleAI() {
    // Verify Google AI API is reachable
    try {
      // Health check via Vertex AI — ADC authentication (no API key needed)
      await fetch(
        `https://${process.env.GCP_LOCATION ?? 'europe-west1'}-aiplatform.googleapis.com/v1/projects/${process.env.GCP_PROJECT_ID}/locations/${process.env.GCP_LOCATION ?? 'europe-west1'}/publishers/google/models`,
      );
      return { vertexAI: { status: 'up' } };
    } catch {
      return { vertexAI: { status: 'down' } };
    }
  }
}

// test/smoke/critical-paths.smoke.ts
describe('Critical Paths Smoke Tests', () => {
  it('should handle chat query end-to-end', async () => {
    const response = await request(process.env.API_URL)
      .post('/api/chat/query')
      .set('Authorization', `Bearer ${process.env.SMOKE_TEST_TOKEN}`)
      .send({
        message: 'smoke test query',
        sectorId: process.env.SMOKE_TEST_SECTOR_ID,
      })
      .expect(200);

    expect(response.body.response).toBeDefined();
    expect(response.body.sources).toBeDefined();
  });

  it('should authenticate user', async () => {
    await request(process.env.API_URL)
      .get('/api/users/me')
      .set('Authorization', `Bearer ${process.env.SMOKE_TEST_TOKEN}`)
      .expect(200);
  });

  it('health endpoint should return ok', async () => {
    const response = await request(process.env.API_URL)
      .get('/health')
      .expect(200);

    expect(response.body.status).toBe('ok');
    expect(response.body.info.database.status).toBe('up');
  });
});

// scripts/run-smoke-tests.sh
#!/bin/bash

echo "🔍 Running Smoke Tests in Production..."

export API_URL="https://api.contextai.com"
export SMOKE_TEST_TOKEN="$PROD_SMOKE_TEST_TOKEN"
export SMOKE_TEST_SECTOR_ID="$PROD_SMOKE_TEST_SECTOR"

pnpm test:smoke || {
  echo "❌ Smoke tests failed!"
  # Send alert to Slack/PagerDuty
  curl -X POST $SLACK_WEBHOOK_URL -d '{"text":"🚨 Production smoke tests failed!"}'
  exit 1
}

echo "✅ Smoke tests passed!"

// Dependencias
pnpm add @nestjs/terminus
```

---

## Resumen y Orden de Implementación

### Fase 1: Setup de Testing (Secuencial)
1. Issue 7.1: Configure Test Infrastructure
2. Issue 7.2: Implement E2E Test Helpers

### Fase 2: Backend E2E Tests (Secuencial)
3. Issue 7.3: Implement Complete RAG Flow E2E Test
4. Issue 7.4: Implement Sector Isolation E2E Test
5. Issue 7.5: Implement Auth & AuthZ E2E Tests

### Fase 3: Frontend Tests (Paralelo con Fase 2)
6. Issue 7.6: Implement Frontend Component Tests
7. Issue 7.7: Implement Frontend E2E Tests

### Fase 4: Especialización (Paralelo)
8. Issue 7.8: Implement API Contract Tests
9. Issue 7.9: Implement Performance Tests
10. Issue 7.10: Implement Test Data Management

### Fase 5: Calidad y Seguridad (Paralelo)
11. Issue 7.13: Implement Accessibility Testing
12. Issue 7.14: Implement Security Testing
13. Issue 7.15: Implement Visual Regression Testing
14. Issue 7.16: Implement Smoke Tests

### Fase 6: Consolidación y Validación (Secuencial)
15. Issue 7.11: Consolidate Test Reports and CI/CD
16. Issue 7.12: MVP Validation Checklist

---

## Estimación Total

**Total de horas estimadas:** 103 horas (80 + 23 de nuevos issues)  
**Total de sprints (2 semanas c/u):** ~3 sprints  
**Desarrolladores recomendados:** 2-3 (QA Engineers o Full-Stack)

---

## Test Coverage Targets

### Backend
- **Unit Tests:** >= 85%
- **Integration Tests:** >= 75%
- **E2E Tests:** Cobertura de flujos principales
- **Overall:** >= 80%

### Frontend
- **Component Tests:** >= 80%
- **E2E Tests:** Cobertura de flujos críticos
- **Overall:** >= 75%

---

## Performance Targets

- **Chat Query Response:** < 3 segundos
- **Vector Search:** < 500ms
- **Document Processing:** < 30 segundos por documento
- **Page Load (Frontend):** < 2 segundos
- **Concurrent Users:** 50 usuarios simultáneos

---

## Tools and Dependencies

### Backend Testing
```bash
# Jest y testing utilities
pnpm add -D jest @types/jest ts-jest
pnpm add -D @nestjs/testing
pnpm add -D supertest @types/supertest

# Mocks y fixtures
pnpm add -D faker @faker-js/faker
pnpm add -D jest-mock-extended
```

### Frontend Testing
```bash
# Vitest y Testing Library
pnpm add -D vitest @vitest/ui
pnpm add -D @testing-library/react @testing-library/jest-dom
pnpm add -D @testing-library/user-event
pnpm add -D jsdom

# Playwright para E2E
pnpm add -D @playwright/test

# Accessibility Testing
pnpm add -D jest-axe @axe-core/playwright

# Backend Health Checks
pnpm add @nestjs/terminus
```

---

## Criterios de Aceptación del MVP

### Funcionales
- [ ] Usuario puede autenticarse con Auth0
- [ ] Admin puede subir documentos (PDF/MD) a sectores
- [ ] Usuario puede hacer preguntas al asistente
- [ ] Asistente responde basándose en documentos
- [ ] Respuestas muestran fuentes utilizadas
- [ ] Información aislada por sectores

### No Funcionales
- [ ] Coverage >= 80%
- [ ] Performance dentro de targets
- [ ] Zero errores de linter
- [ ] Build exitoso
- [ ] Seguridad validada

### Calidad
- [ ] Tests E2E cubren flujos principales
- [ ] Tests unitarios para toda lógica de negocio
- [ ] Tests de integración para módulos principales
- [ ] Documentación completa

---

## Validación de Completitud

La Fase 7 se considera completa cuando:

- [ ] Todos los 16 issues están completados
- [ ] Suite completa de tests implementada y pasando
- [ ] Coverage >= 80% en backend y frontend
- [ ] Accessibility score >= 90 (Lighthouse)
- [ ] Security tests pasan sin vulnerabilidades críticas
- [ ] Visual regression tests configurados
- [ ] Smoke tests funcionando en producción
- [ ] Todos los criterios de aceptación del MVP validados
- [ ] CI/CD configurado y funcionando
- [ ] Performance dentro de targets establecidos
- [ ] Reporte de validación generado y aprobado
- [ ] `pnpm lint` pasa sin errores
- [ ] `pnpm build` genera builds exitosos
- [ ] `pnpm test:all` pasa al 100%
- [ ] Documentación completa y actualizada

---

## Comandos de Validación Final

```bash
# Ejecutar validación completa del MVP
./scripts/validate-mvp.sh

# O manualmente:
# Backend
cd context-ai-api
pnpm install
pnpm lint
pnpm test:all
pnpm test:cov
pnpm build

# Frontend
cd ../context-ai-front
pnpm install
pnpm lint
pnpm test
pnpm test:e2e
pnpm build

# Verificar coverage
echo "Backend coverage:" && cat context-ai-api/coverage/coverage-summary.json | jq '.total.lines.pct'
echo "Frontend coverage:" && cat context-ai-front/coverage/coverage-summary.json | jq '.total.lines.pct'
```

---

## Next Steps Post-Fase 7

Una vez completada la Fase 7 y validado el MVP:

1. **Deployment:** Desplegar a entorno de staging/producción
2. **User Acceptance Testing:** Tests con usuarios reales
3. **Performance Monitoring:** Configurar monitoreo en producción
4. **Feedback Loop:** Recopilar feedback para mejoras
5. **Post-MVP Features:** Planificar features adicionales del roadmap

---

## Documentation Links

- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Vitest Documentation](https://vitest.dev/guide/)
- [Testing Library](https://testing-library.com/docs/react-testing-library/intro/)
- [Playwright Documentation](https://playwright.dev/docs/intro)
- [NestJS Testing](https://docs.nestjs.com/fundamentals/testing)

