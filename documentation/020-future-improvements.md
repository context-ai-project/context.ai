---
name: "Futuras Mejoras — Backlog de Evolución Post-v1.3"
overview: "Backlog de mejoras planificadas para versiones futuras de Context.ai. Incluye respuestas adaptativas con Genkit Structured Output (Propuesta C), y otras mejoras identificadas durante el desarrollo de v1.3."
priority: Post-v1.3
related_docs:
  - "019-release-v1.3.md"
  - "017-improvements-monitoring-observability.md"
  - "018-v2-multimedia-capsules.md"
---

# Futuras Mejoras — Backlog de Evolución Post-v1.3

Mejoras identificadas durante el desarrollo que se posponen para versiones futuras. Se priorizan según impacto en la experiencia del usuario y complejidad de implementación.

---

## FUT-1: Respuestas Adaptativas con Genkit Structured Output (Propuesta C)

**Versión objetivo:** v2.0
**Prioridad:** Media
**Estimación:** 12-16 horas
**Origen:** Análisis de propuestas de structured output en [019-release-v1.3.md](./019-release-v1.3.md) — Feature 4

### Descripción

Evolución de la estructura de respuestas RAG donde el LLM decide dinámicamente el formato más apropiado según la complejidad de la pregunta del usuario. A diferencia de la Propuesta A (implementada en v1.3) que siempre retorna secciones, la Propuesta C permite al modelo elegir entre `simple`, `detailed` o `step_by_step`, y adicionalmente extraer datos clave como `highlights` (fechas, requisitos, contactos).

### Contexto

En v1.3 se implementa la **Propuesta A** (respuesta con secciones fijas) que es fiable y de complejidad media. La Propuesta C ofrece una experiencia más rica pero requiere:

1. Un schema Zod más complejo que el LLM debe respetar consistentemente
2. Más componentes de frontend para renderizar los distintos formatos
3. Mayor consumo de tokens por la complejidad del schema
4. Testing más extensivo para validar los distintos formatos

### Schema propuesto

```typescript
import { z } from 'zod';

/**
 * Adaptive Response Schema (Propuesta C)
 *
 * El LLM elige el formato de respuesta según la complejidad de la pregunta:
 * - simple: para preguntas directas con respuesta corta
 * - detailed: para temas que requieren explicación en profundidad
 * - step_by_step: para procedimientos y procesos
 */
export const adaptiveResponseSchema = z.object({
  /** Formato elegido por el LLM según la complejidad de la pregunta */
  format: z.enum(['simple', 'detailed', 'step_by_step']).describe(
    'Response format chosen based on question complexity: "simple" for direct answers, "detailed" for topics needing explanation, "step_by_step" for procedures'
  ),

  /** Título o encabezado de la respuesta (opcional para formato simple) */
  title: z.string().describe('Response title/heading').optional(),

  /** Contenido principal en markdown */
  content: z.string().describe('Main response content in markdown format'),

  /** Pasos secuenciales (solo para formato step_by_step) */
  steps: z.array(z.object({
    order: z.number().describe('Step number (1-based)'),
    instruction: z.string().describe('Main instruction for this step'),
    detail: z.string().describe('Additional detail or explanation').optional(),
  })).describe('Step-by-step instructions, only populated when format is step_by_step').optional(),

  /** Datos clave extraídos para destacar visualmente */
  highlights: z.array(z.object({
    label: z.string().describe('Label for the data point (e.g., "Plazo", "Requisito")'),
    value: z.string().describe('Value of the data point'),
    type: z.enum(['info', 'date', 'requirement', 'contact']).describe('Type for icon/color rendering'),
  })).describe('Key data points extracted from the documentation to highlight visually').optional(),

  /** Sugerencia adicional o siguiente paso */
  suggestion: z.string().describe('Additional suggestion or recommended next step for the user').optional(),
});

export type AdaptiveResponse = z.infer<typeof adaptiveResponseSchema>;
```

### Ejemplos de salida por formato

#### Formato `simple`

Pregunta: _"¿Cuántos días de vacaciones tengo?"_

```json
{
  "format": "simple",
  "content": "Según la política de la empresa, los empleados con menos de 2 años tienen **15 días hábiles** de vacaciones al año. A partir del tercer año, se incrementan 2 días adicionales por año hasta un máximo de 25 días.",
  "highlights": [
    { "label": "Días base", "value": "15 días hábiles/año", "type": "info" },
    { "label": "Máximo", "value": "25 días hábiles/año", "type": "info" }
  ]
}
```

#### Formato `detailed`

Pregunta: _"¿Cómo funciona la política de trabajo remoto?"_

```json
{
  "format": "detailed",
  "title": "Política de Trabajo Remoto",
  "content": "La empresa ofrece un modelo de trabajo híbrido con flexibilidad para trabajar de forma remota...",
  "highlights": [
    { "label": "Modelo", "value": "Híbrido (3 oficina + 2 remoto)", "type": "info" },
    { "label": "Requisito", "value": "Mínimo 3 meses en la empresa", "type": "requirement" },
    { "label": "Contacto", "value": "rrhh@empresa.com", "type": "contact" }
  ],
  "suggestion": "Consulta con tu manager directo para definir los días específicos de oficina para tu equipo."
}
```

#### Formato `step_by_step`

Pregunta: _"¿Cómo solicito vacaciones?"_

```json
{
  "format": "step_by_step",
  "title": "Proceso de Solicitud de Vacaciones",
  "content": "Para solicitar vacaciones, sigue el proceso a continuación a través del portal HR.",
  "steps": [
    { "order": 1, "instruction": "Accede al Portal HR", "detail": "Ingresa a portal.empresa.com con tus credenciales corporativas" },
    { "order": 2, "instruction": "Selecciona 'Solicitar Vacaciones'", "detail": "En el menú lateral, sección 'Tiempo Libre'" },
    { "order": 3, "instruction": "Ingresa las fechas deseadas", "detail": "El sistema mostrará tu saldo disponible automáticamente" },
    { "order": 4, "instruction": "Envía la solicitud", "detail": "Tu manager recibirá una notificación para aprobar" }
  ],
  "highlights": [
    { "label": "Anticipación mínima", "value": "15 días hábiles", "type": "requirement" },
    { "label": "Aprobación", "value": "Manager directo", "type": "info" }
  ],
  "suggestion": "Si necesitas vacaciones con menos de 15 días de anticipación, contacta directamente a RRHH para una excepción."
}
```

### Prompt adaptado para Propuesta C

```typescript
function buildAdaptivePrompt(query: string, fragments: FragmentResult[]): string {
  const context = fragments
    .map((f, index) => `[${index + 1}] ${f.content}`)
    .join('\n\n');

  return `You are an onboarding assistant. Answer based ONLY on the provided documentation.

DOCUMENTATION CONTEXT:
${context}

USER QUESTION:
${query}

INSTRUCTIONS:
- Choose the most appropriate format for the response:
  * "simple" — for direct questions with short answers (1-2 paragraphs)
  * "detailed" — for topics that need in-depth explanation with context
  * "step_by_step" — for procedures, processes, or how-to questions
- Extract key data points as highlights (dates, requirements, contacts, important numbers)
- For step_by_step format, break down the process into clear, numbered steps
- Add a suggestion for the user's next action when relevant
- Respond in the SAME LANGUAGE as the user's question
- Use markdown formatting in content fields
- Be transparent if the documentation doesn't fully cover the topic`;
}
```

### Componentes de Frontend necesarios

| Componente | Descripción |
|------------|-------------|
| `AdaptiveResponse.tsx` | Container que renderiza según `format` |
| `SimpleResponse.tsx` | Contenido + highlights |
| `DetailedResponse.tsx` | Título + contenido + highlights + suggestion |
| `StepByStepResponse.tsx` | Título + steps + highlights + suggestion |
| `HighlightCard.tsx` | Card visual para cada highlight con icono por `type` |
| `SuggestionBanner.tsx` | Banner con la sugerencia al usuario |

### Análisis comparativo vs Propuesta A (v1.3)

| Criterio | Propuesta A (v1.3) | Propuesta C (futuro) |
|----------|-------------------|---------------------|
| **Complejidad frontend** | Media | Alta |
| **Calidad visual** | Alta | Muy alta |
| **Fiabilidad LLM** | Alta (schema simple) | Media (schema complejo, el LLM debe elegir formato) |
| **Flexibilidad** | Buena | Excelente |
| **Costo tokens** | Medio | Alto |
| **UX** | Consistente | Contextual y más intuitiva |
| **Testing** | Estándar | Requiere tests por cada formato |

### Pre-requisitos

- v1.3 implementada y estable (Propuesta A funcionando en producción)
- Validación con usuarios reales de que la estructura de secciones fijas es insuficiente
- Métricas de satisfacción de respuestas recolectadas en producción

### Acceptance Criteria

- [ ] El LLM retorna el formato apropiado según la complejidad de la pregunta
- [ ] Los tres formatos (`simple`, `detailed`, `step_by_step`) se renderizan correctamente
- [ ] Los `highlights` se muestran como cards con iconos diferenciados por tipo
- [ ] El campo `steps` solo se renderiza en formato `step_by_step`
- [ ] El `suggestion` se muestra como banner contextual
- [ ] Fallback a Propuesta A si el schema de Propuesta C falla
- [ ] Tests unitarios para cada formato de respuesta
- [ ] Tests de frontend para cada componente nuevo
- [ ] Métricas de calidad de formato elegido vs expectativa del usuario

---

## FUT-2: Notificaciones Externas vía Slack para Eventos del Sistema

**Versión objetivo:** v1.4
**Prioridad:** Baja
**Estimación:** 4 horas
**Origen:** Feature 2 de v1.3 — extensión de infraestructura de eventos y email

### Descripción

Extensión del sistema de notificaciones para enviar mensajes a **Slack** cuando ocurren eventos importantes del sistema. La infraestructura base ya está implementada en v1.3:

- ✅ Event bus (`@nestjs/event-emitter`) — v1.3
- ✅ Notificaciones in-app (BD + listeners) — v1.3
- ⬜ Slack Webhook — **este FUT-2**
- ⬜ Email transaccional (servicio de email) — **este FUT-2**

> **Nota**: En v1.3 se eliminó la dependencia de Resend para emails transaccionales. Este FUT-2 contempla agregar un servicio de email (ej: AWS SES, SendGrid, Resend u otro) junto con Slack para notificaciones externas.

### Alcance

1. **Slack Webhook**: Nuevo listener que envía mensajes a un canal de Slack para eventos clave
2. **Email transaccional**: Integrar un servicio de email para notificar admins en eventos como `document.failed`, `user.activated`, etc.

### Arquitectura

```typescript
// Nuevo listener: src/modules/notifications/application/listeners/slack-notification.listener.ts
@OnEvent('invitation.accepted')
@OnEvent('document.failed')
async handleSlackNotification(event: DomainEvent): Promise<void> {
  await this.slackService.sendMessage(channel, formatEvent(event));
}

// Email transaccional para eventos críticos
@OnEvent('document.failed')
async handleDocumentFailedEmail(event: DocumentFailedEvent): Promise<void> {
  await this.emailService.sendAdminAlert({
    subject: `Document processing failed: ${event.documentName}`,
    // ...
  });
}
```

### Pre-requisitos

- Feature 2 de v1.3 completada (módulo notifications + event bus)
- Cuenta de Slack con webhook configurado
- Servicio de email configurado (AWS SES, SendGrid u otro)

---

## FUT-3: Nivel de Confianza en Respuestas RAG

**Versión objetivo:** v1.4
**Prioridad:** Media
**Estimación:** 4 horas

### Descripción

Agregar un indicador de confianza (`confidence: high | medium | low`) a las respuestas del RAG basado en:
- Similarity score promedio de los fragmentos usados
- Cantidad de fragmentos relevantes encontrados
- Score de evaluación de faithfulness/relevancy

Esto permite al frontend mostrar un badge visual que indica qué tan segura es la respuesta.

### Lógica propuesta

```typescript
function calculateConfidence(fragments: FragmentResult[], evaluation?: RagEvaluationResult): 'high' | 'medium' | 'low' {
  const avgSimilarity = fragments.reduce((sum, f) => sum + f.similarity, 0) / fragments.length;
  const fragmentCount = fragments.length;
  const faithfulness = evaluation?.faithfulness?.score ?? 0;

  if (avgSimilarity >= 0.85 && fragmentCount >= 3 && faithfulness >= 0.8) return 'high';
  if (avgSimilarity >= 0.75 && fragmentCount >= 2) return 'medium';
  return 'low';
}
```

---

## FUT-4: Preguntas de Seguimiento Sugeridas

**Versión objetivo:** v1.4 o v2.0
**Prioridad:** Baja
**Estimación:** 4 horas

### Descripción

El LLM sugiere preguntas de seguimiento relevantes al final de cada respuesta. Estas se muestran como botones clickeables en el chat para guiar al usuario hacia información relacionada.

### Schema

```typescript
followUpQuestions: z.array(z.string())
  .max(3)
  .describe('Suggested follow-up questions related to the topic')
  .optional(),
```

### Frontend

Renderizar como chips/buttons debajo de la respuesta del asistente que al hacer click envían automáticamente la pregunta al chat.

---

## FUT-5: Caché de Respuestas RAG

**Versión objetivo:** v2.0
**Prioridad:** Media
**Estimación:** 8 horas
**Relacionado:** [017-improvements-monitoring-observability.md](./017-improvements-monitoring-observability.md) — IMP-6

### Descripción

Implementar un sistema de caché para respuestas RAG frecuentes. Si una pregunta similar (por embedding similarity) ya fue respondida recientemente, retornar la respuesta cacheada en lugar de ejecutar todo el pipeline RAG.

### Beneficios

- Reducción de latencia para preguntas repetidas
- Ahorro en costos de API de Gemini
- Mejor experiencia de usuario con respuestas instantáneas

### Pre-requisitos

- Redis configurado (ver IMP-6 en 017)
- Métricas de preguntas frecuentes recolectadas

---

## Resumen y Priorización

| # | Mejora | Versión | Prioridad | Estimación | Dependencias |
|---|--------|---------|-----------|------------|--------------|
| FUT-1 | Respuestas Adaptativas (Propuesta C) | v2.0 | Media | 12-16h | v1.3 Feature 4 estable |
| FUT-2 | Notificaciones Email/Slack | v1.4 | Baja | 6h | v1.3 Feature 2 |
| FUT-3 | Nivel de Confianza | v1.4 | Media | 4h | v1.3 Feature 4 |
| FUT-4 | Preguntas de Seguimiento | v1.4/v2.0 | Baja | 4h | v1.3 Feature 4 |
| FUT-5 | Caché de Respuestas RAG | v2.0 | Media | 8h | Redis (IMP-6) |
| | | | **Total** | **34-38h** | |

