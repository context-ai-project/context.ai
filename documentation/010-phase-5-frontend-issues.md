---
name: Fase 5 - Frontend Chat Interface
overview: "Descomposición de la Fase 5 del MVP en issues granulares para implementar la interfaz de chat con Next.js 16+ (App Router), React 19, Zustand para gestión de estado, y optimización de assets. Incluye componentes UI, internacionalización con next-intl v4, integración con API backend mediante NextAuth.js v5, y testing E2E con Playwright."
phase: 5
parent_phase: "009-plan-implementacion-detallado.md"
total_issues: 15
---

# Fase 5: Frontend - Chat Interface

> **Nota de actualización (Febrero 2026):** Anotaciones `[ACTUALIZACIÓN]` reflejan cambios vs plan original:
> - **Next.js 16+** con React 19 (no Next.js 13+)
> - **NextAuth.js v5** con Auth0 provider (no `@auth0/nextjs-auth0`)
> - **next-intl v4** (no v3) con locale routing `[locale]/`
> - **Tailwind CSS 4** (no v3)
> - **Playwright** para E2E tests (no Vitest/Supertest)
> - Rutas bajo `src/app/[locale]/` para i18n

Descomposición en issues manejables para el desarrollo de la interfaz de chat del MVP.

---

## Issue 5.1: Setup Base Chat Page Structure

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 4 horas

### Descripción

Crear la estructura base de la página de chat con routing de Next.js, layout responsivo y esqueleto básico de la interfaz sin funcionalidad compleja.

### Acceptance Criteria

- [ ] Página `/chat` accesible desde el navegador
- [ ] Layout responsivo con diseño de 2 columnas (opcional: sidebar para historial)
- [ ] Área principal para mensajes (vacía inicialmente)
- [ ] Área inferior para input de mensajes (sin funcionalidad aún)
- [ ] Navbar con información del usuario autenticado
- [ ] Diseño mobile-first que funciona en pantallas pequeñas

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
src/app/[locale]/(protected)/chat/page.tsx       # Página principal del chat (Server Component)
src/app/[locale]/(protected)/layout.tsx          # Layout protegido (auth check + providers)
src/components/chat/ChatContainer.tsx            # Contenedor principal del chat (Client Component)
src/components/shared/Navbar.tsx                 # Barra de navegación con info del usuario
```

### Technical Notes

- Usar Tailwind CSS 4 para estilos responsivos
- Implementar layout con CSS Grid o Flexbox
- Usar componentes de shadcn/ui
- `[ACTUALIZACIÓN]` Rutas bajo `[locale]` para i18n con next-intl v4

**Next.js 16+ App Router:** — `[ACTUALIZACIÓN]`
```typescript
// src/app/[locale]/(protected)/chat/page.tsx - Server Component
export const dynamic = 'force-dynamic';

export default function ChatPage() {
  return <ChatContainer />;
}

// src/app/[locale]/(protected)/layout.tsx - Protected layout
import { auth } from '@/auth';
import { redirect } from 'next/navigation';

export default async function ProtectedLayout({ children }) {
  const session = await auth();
  if (!session) redirect('/auth/login');
  return <SessionProvider session={session}>{children}</SessionProvider>;
}

// src/components/chat/ChatContainer.tsx - Client Component
'use client';
export function ChatContainer() {
  // Estado, effects, event handlers aquí
}
```
**Regla General:**
- Server Components: Layouts, páginas, data fetching
- Client Components (`'use client'`): Interactividad, hooks, event handlers

---

## Issue 5.2: Implement Chat State Management with Zustand

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 6 horas

### Descripción

Configurar Zustand para manejar el estado global del chat: mensajes, conversación activa, estados de carga y errores.

### Acceptance Criteria

- [ ] Store de Zustand creado con tipos TypeScript completos
- [ ] Estado incluye: mensajes, isLoading, error, conversationId
- [ ] Acciones: sendMessage, addMessage, setLoading, setError, clearMessages
- [ ] Persistencia opcional de conversación en localStorage
- [ ] Tests unitarios del store con al menos 80% coverage
- [ ] Integración correcta de tipos desde `@context-ai-project/shared`

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
src/stores/chat.store.tsx           # Store principal de Zustand (con JSX para provider)
src/stores/user.store.tsx           # Store de usuario (sector activo, sessionStorage persist)
src/types/message.types.ts          # Tipos específicos del frontend
```

### Technical Notes

```typescript
// Estructura sugerida del store
interface ChatState {
  messages: Message[];
  conversationId: string | null;
  isLoading: boolean;
  error: string | null;
  sendMessage: (text: string, sectorId: string) => Promise<void>;
  addMessage: (message: Message) => void;
  setLoading: (loading: boolean) => void;
  setError: (error: string | null) => void;
  clearMessages: () => void;
}
```

---

## Issue 5.3: Create Message List Component

**Prioridad:** Alta  
**Dependencias:** 5.1, 5.2  
**Estimación:** 8 horas

### Descripción

Implementar el componente que renderiza la lista de mensajes del chat con scroll automático, diferenciación visual entre mensajes del usuario y asistente, y soporte para markdown.

### Acceptance Criteria

- [ ] Renderiza lista de mensajes correctamente
- [ ] Diferenciación visual clara entre mensajes user/assistant
- [ ] Scroll automático al último mensaje cuando llegan nuevos mensajes
- [ ] Renderiza markdown en respuestas del asistente (usando `react-markdown`)
- [ ] Muestra avatar o icono para cada tipo de mensaje
- [ ] Timestamps formateados correctamente
- [ ] Tests de componente con React Testing Library

### Files to Create

```
components/chat/MessageList.tsx       # Lista de mensajes
components/chat/Message.tsx           # Componente individual de mensaje
components/chat/MessageList.test.tsx  # Tests del componente
lib/utils/date-formatter.ts           # Utilidad para formatear fechas
```

### Technical Notes

- Usar `useRef` y `useEffect` para scroll automático
- Implementar virtualization con `react-window` si hay muchos mensajes
- Considerar animaciones con `framer-motion`
- Avatar user: usar datos de Auth0
- Avatar assistant: icono o logo del asistente

---

## Issue 5.4: Create Message Input Component

**Prioridad:** Alta  
**Dependencias:** 5.1, 5.2  
**Estimación:** 6 horas

### Descripción

Desarrollar el componente de input para que el usuario escriba mensajes, con validación, manejo de Enter, estado de envío y indicador visual de carga.

### Acceptance Criteria

- [ ] Textarea que se expande con el contenido (max 5 líneas)
- [ ] Botón de envío habilitado solo con texto válido
- [ ] Submit con Enter (Shift+Enter para nueva línea)
- [ ] Deshabilita input mientras está enviando
- [ ] Limpia el input después de enviar
- [ ] Muestra estado de loading con spinner o animación
- [ ] Validación: no permitir mensajes vacíos o solo espacios
- [ ] Tests del componente

### Files to Create

```
components/chat/MessageInput.tsx       # Input de mensajes
components/chat/MessageInput.test.tsx  # Tests del componente
components/ui/Button.tsx               # Componente de botón reutilizable
components/ui/Textarea.tsx             # Textarea autoexpandible
```

### Technical Notes

```typescript
// Manejo de teclas
const handleKeyDown = (e: KeyboardEvent<HTMLTextAreaElement>) => {
  if (e.key === 'Enter' && !e.shiftKey) {
    e.preventDefault();
    handleSubmit();
  }
};
```

---

## Issue 5.5: Create Source Card Component

**Prioridad:** Media  
**Dependencias:** 5.1  
**Estimación:** 5 horas

### Descripción

Implementar componente para mostrar las fuentes (chunks) utilizadas en la respuesta del asistente, con acordeón expandible y metadata del documento.

### Acceptance Criteria

- [ ] Renderiza lista de fuentes con título del documento
- [ ] Acordeón expandible para ver contenido del chunk
- [ ] Muestra metadata: página, score de similitud, fecha
- [ ] Diseño visual que indica la fuente citada
- [ ] Animación suave al expandir/colapsar
- [ ] Máximo de fuentes mostradas configurable (default: 5)
- [ ] Tests del componente

### Files to Create

```
components/chat/SourceCard.tsx        # Card individual de fuente
components/chat/SourceList.tsx        # Lista de fuentes
components/ui/Accordion.tsx           # Componente accordion (o usar Radix UI)
components/chat/SourceCard.test.tsx   # Tests
```

### Technical Notes

- Usar `@radix-ui/react-accordion` o implementar propio
- Mostrar score de similitud como porcentaje
- Iconos diferentes según tipo de fuente (PDF, MD, URL)
- Considerar badge para indicar relevancia

---

## Issue 5.6: Implement Chat API Client

**Prioridad:** Alta  
**Dependencias:** Ninguna  
**Estimación:** 6 horas

### Descripción

Crear cliente API con axios para comunicarse con el backend, incluyendo interceptores para autenticación, manejo de errores y tipado completo.

### Acceptance Criteria

- [ ] Cliente axios configurado con baseURL desde env vars
- [ ] Interceptor que añade Authorization header con access token
- [ ] Interceptor de respuesta para manejo global de errores
- [ ] Función `sendChatMessage` que consume endpoint POST /api/chat/query
- [ ] Función `getConversations` para historial
- [ ] Tipos de respuesta basados en `@context-ai-project/shared`
- [ ] Manejo de errores de red y timeouts
- [ ] Tests con mock de axios

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
src/lib/api/client.ts              # Cliente axios base con interceptores (auth, timeout, error)
src/lib/api/chat.api.ts            # Funciones específicas de chat
src/lib/api/user.api.ts            # Funciones de sincronización de usuario
src/lib/api/error-handler.ts       # Manejo centralizado de errores (APIError class)
```

### Technical Notes

```typescript
// Estructura sugerida
export const chatApi = {
  sendMessage: async (dto: ChatQueryDto): Promise<ChatResponseDto> => {
    const response = await apiClient.post('/chat/query', dto);
    return response.data;
  },
  
  getConversations: async (userId: string): Promise<Conversation[]> => {
    const response = await apiClient.get(`/chat/conversations/${userId}`);
    return response.data;
  }
};
```

---

## Issue 5.7: Integrate State with API and Components

**Prioridad:** Alta  
**Dependencias:** 5.2, 5.3, 5.4, 5.6  
**Estimación:** 8 horas

### Descripción

Conectar el store de Zustand con el API client y los componentes, implementando el flujo completo de envío de mensaje y recepción de respuesta.

### Acceptance Criteria

- [ ] Acción `sendMessage` del store llama al API client
- [ ] Mensajes se añaden optimistamente al estado
- [ ] Respuesta del backend se añade al estado al recibirse
- [ ] Manejo de errores muestra mensaje al usuario
- [ ] Loading state se refleja en la UI
- [ ] ConversationId se mantiene durante la sesión
- [ ] Tests de integración del flujo completo

### Files to Create

```
hooks/useChat.ts                      # Custom hook que conecta store y API
hooks/useChat.test.ts                 # Tests del hook
lib/utils/optimistic-updates.ts      # Utilidades para updates optimistas
```

### Technical Notes

```typescript
// Flujo de optimistic updates
const sendMessage = async (text: string) => {
  const optimisticMessage = createOptimisticMessage(text);
  addMessage(optimisticMessage);
  
  try {
    const response = await chatApi.sendMessage({ message: text, sectorId });
    updateMessage(optimisticMessage.id, response);
  } catch (error) {
    removeMessage(optimisticMessage.id);
    setError('Failed to send message');
  }
};
```

---

## Issue 5.8: Implement Error Boundaries and Error States

**Prioridad:** Media  
**Dependencias:** 5.7  
**Estimación:** 5 horas

### Descripción

Añadir error boundaries de React, estados de error visuales y recuperación graceful de errores de red o del servidor.

### Acceptance Criteria

- [ ] Error boundary que captura errores de rendering
- [ ] Pantalla de error user-friendly con opción de retry
- [ ] Mensajes de error específicos por tipo (red, auth, servidor)
- [ ] Toast/notification para errores no críticos
- [ ] Logging de errores a Sentry (configurado pero opcional)
- [ ] Tests de error scenarios

### Files to Create

```
components/shared/ErrorBoundary.tsx   # Error boundary de React
components/chat/ErrorState.tsx        # UI para estados de error
components/ui/Toast.tsx               # Sistema de notificaciones
lib/utils/error-logger.ts             # Logger de errores
```

### Technical Notes

- Usar `react-error-boundary` o implementar propio
- Integrar con Sentry para producción
- Categorías de errores: network, auth, validation, server, unknown

---

## Issue 5.9: Add Loading States and Skeletons

**Prioridad:** Media  
**Dependencias:** 5.3  
**Estimación:** 4 horas

### Descripción

Implementar estados de carga visuales con skeleton screens para mejorar la percepción de rendimiento y UX.

### Acceptance Criteria

- [ ] Skeleton para lista de mensajes durante carga inicial
- [ ] Indicador de "typing" mientras el asistente genera respuesta
- [ ] Skeleton para source cards mientras se cargan
- [ ] Animaciones suaves de transición
- [ ] Estados de carga no bloquean la UI
- [ ] Tests de estados de carga

### Files to Create

```
components/ui/Skeleton.tsx              # Componente skeleton base
components/chat/MessageSkeleton.tsx     # Skeleton para mensajes
components/chat/TypingIndicator.tsx     # Indicador de "escribiendo..."
```

### Technical Notes

- Usar animación de shimmer con Tailwind
- Typing indicator con 3 dots animados
- Skeleton debe coincidir con el tamaño aproximado del contenido real

---

## Issue 5.10: Implement Markdown Rendering for Responses

**Prioridad:** Media  
**Dependencias:** 5.3  
**Estimación:** 5 horas

### Descripción

Configurar renderizado de markdown en respuestas del asistente con sintaxis highlighting para código, links seguros y estilos personalizados.

### Acceptance Criteria

- [ ] Respuestas del asistente renderizan markdown correctamente
- [ ] Code blocks con syntax highlighting (usando `prism` o `highlight.js`)
- [ ] Links se abren en nueva pestaña con `rel="noopener noreferrer"`
- [ ] Estilos de markdown consistentes con el diseño
- [ ] Soporte para listas, tablas, énfasis, headings
- [ ] Sanitización de HTML para prevenir XSS
- [ ] Tests de rendering de diferentes elementos markdown

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
src/components/chat/MarkdownRenderer.tsx  # Wrapper para react-markdown con syntax highlighting
```
> **Nota:** Los estilos de markdown se manejan con Tailwind CSS 4, sin archivo CSS separado.

### Technical Notes

```typescript
// Dependencias necesarias
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import { Prism as SyntaxHighlighter } from 'react-syntax-highlighter';
```

---

## Issue 5.11: Add Empty State and Welcome Screen

**Prioridad:** Baja  
**Dependencias:** 5.1, 5.3  
**Estimación:** 3 horas

### Descripción

Crear pantalla de bienvenida cuando no hay mensajes, con sugerencias de preguntas y onboarding básico.

### Acceptance Criteria

- [ ] Pantalla de bienvenida cuando no hay mensajes
- [ ] Sugerencias de preguntas frecuentes (clickeables)
- [ ] Mensaje explicativo sobre el asistente
- [ ] Logo o ilustración del asistente
- [ ] Transición suave al iniciar primer mensaje
- [ ] Tests del componente

### Files to Create

```
components/chat/EmptyState.tsx          # Estado vacío del chat
components/chat/SuggestedQuestions.tsx  # Preguntas sugeridas
constants/suggested-questions.ts        # Lista de preguntas por defecto
```

### Technical Notes

- Preguntas sugeridas deben ser relevantes al contexto del usuario
- Considerar diferentes sugerencias según el sector del usuario
- Usar ilustraciones de undraw.co o similar

---

## Issue 5.12: E2E Testing for Chat Flow

**Prioridad:** Alta  
**Dependencias:** Todas las anteriores  
**Estimación:** 8 horas

### Descripción

Crear tests end-to-end que validen el flujo completo de chat desde la perspectiva del usuario.

### Acceptance Criteria

- [ ] Test: Usuario escribe mensaje y recibe respuesta
- [ ] Test: Fuentes se muestran correctamente
- [ ] Test: Manejo de errores funciona
- [ ] Test: Múltiples mensajes en conversación
- [ ] Test: Markdown se renderiza correctamente
- [ ] Test: Loading states son visibles
- [ ] Tests corren en CI/CD

### Files to Create — `[ACTUALIZACIÓN]` Estructura real:

```
e2e/chat/chat-flow.spec.ts      # Tests E2E con Playwright
e2e/auth/auth-flow.spec.ts      # Tests E2E de autenticación
e2e/dashboard/dashboard.spec.ts # Tests E2E del dashboard
e2e/visual/visual-regression.spec.ts  # Tests de regresión visual
e2e/helpers/                     # Utilidades para tests
```

### Technical Notes — `[ACTUALIZACIÓN]` Usa Playwright

```typescript
// Ejemplo de test E2E con Playwright (implementación real)
import { test, expect } from '@playwright/test';

test('user can send message and receive response', async ({ page }) => {
  // Playwright config incluye web server (pnpm dev) y baseURL
  await page.goto('/es/chat');  // [ACTUALIZACIÓN] Locale prefix
  
  // Escribir mensaje
  await page.fill('[data-testid="message-input"]', '¿Cómo pido vacaciones?');
  await page.click('[data-testid="send-button"]');
  
  // Verificar loading state
  await expect(page.locator('[data-testid="typing-indicator"]')).toBeVisible();
  
  // Verificar respuesta
  await expect(page.locator('[data-testid="assistant-message"]')).toBeVisible();
});
```

---

## Issue 5.13: Implement User Profile and Session Management

**Prioridad:** Alta  
**Dependencias:** 5.1  
**Estimación:** 6 horas

### Descripción

Crear componentes para gestión de sesión de usuario, incluyendo perfil, selector de sector activo, y logout functionality.

### Acceptance Criteria

- [ ] Componente UserProfile muestra info del usuario autenticado
- [ ] Avatar del usuario con fallback a iniciales
- [ ] Selector de sector activo (dropdown)
- [ ] Botón de logout con confirmación
- [ ] Muestra rol y permisos del usuario
- [ ] Persiste sector activo en session storage
- [ ] Animaciones y transiciones suaves
- [ ] Tests del componente

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
src/hooks/useCurrentUser.ts                # Hook combinando NextAuth session + Zustand store
src/stores/user.store.tsx                   # Store de usuario con Zustand (persist en sessionStorage)
src/components/shared/Navbar.tsx            # Incluye perfil de usuario y sector selector
```

### Technical Notes — `[ACTUALIZACIÓN]` Usa NextAuth.js v5 (no @auth0/nextjs-auth0)

```typescript
// src/hooks/useCurrentUser.ts
'use client';

import { useSession } from 'next-auth/react';  // [ACTUALIZACIÓN] NextAuth.js v5
import { useUserStore } from '@/stores/user.store';

export function useCurrentUser() {
  const { data: session, status } = useSession();
  const { currentSector, sectors, setCurrentSector } = useUserStore();

  return {
    user: session?.user,
    isLoading: status === 'loading',
    isAuthenticated: status === 'authenticated',
    currentSector,
    sectors,
    setCurrentSector,
  };
}

// src/stores/user.store.tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

interface UserState {
  currentSector: string | null;
  sectors: Sector[];
  setCurrentSector: (sector: string) => void;
  setSectors: (sectors: Sector[]) => void;
}

export const useUserStore = create<UserState>()(
  persist(
    (set) => ({
      currentSector: null,
      sectors: [],
      setCurrentSector: (sector) => set({ currentSector: sector }),
      setSectors: (sectors) => set({ sectors }),
    }),
    {
      name: 'user-storage',
      storage: createJSONStorage(() => sessionStorage),  // [ACTUALIZACIÓN] sessionStorage
    }
  )
);
```

---

## Issue 5.14: Implement Image Optimization with Next.js Image

**Prioridad:** Media  
**Dependencias:** 5.13  
**Estimación:** 3 horas

### Descripción

Optimizar carga de imágenes usando Next.js Image component para avatares, logos y assets estáticos con lazy loading y responsive images.

### Acceptance Criteria

- [ ] Next.js Image usado para avatares de usuario
- [ ] Logo del asistente optimizado
- [ ] Imágenes de empty state optimizadas
- [ ] Lazy loading configurado correctamente
- [ ] Responsive images con diferentes tamaños
- [ ] Placeholder blur mientras carga
- [ ] Tests de rendering

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
src/lib/utils/image-config.ts              # OptimizedImage component + configuración
src/components/ui/avatar.tsx               # Avatar component (shadcn/ui)
```

### Technical Notes

```typescript
// components/ui/OptimizedImage.tsx
import Image from 'next/image';

interface OptimizedImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  priority?: boolean;
}

export function OptimizedImage({
  src,
  alt,
  width = 40,
  height = 40,
  priority = false,
}: OptimizedImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      priority={priority}
      placeholder="blur"
      blurDataURL="/images/placeholder.png"
      sizes="(max-width: 768px) 100vw, 50vw"
    />
  );
}

// next.config.js
module.exports = {
  images: {
    domains: [
      's.gravatar.com', // Auth0 avatares
      'lh3.googleusercontent.com', // Google avatares
      'avatars.githubusercontent.com', // GitHub avatares
    ],
  },
};
```

---

## Issue 5.15: Implement Internationalization (i18n)

**Prioridad:** Media  
**Dependencias:** 5.1  
**Estimación:** 8 horas

### Descripción

Implementar soporte multi-idioma (español e inglés inicialmente) usando next-intl para internacionalizar toda la interfaz.

### Acceptance Criteria

- [ ] next-intl configurado y funcionando
- [ ] Middleware de Next.js detecta idioma del navegador
- [ ] Selector de idioma en navbar
- [ ] Traducciones para español e inglés completas
- [ ] URLs localizadas (/es/chat, /en/chat)
- [ ] Formateo de fechas según locale
- [ ] Formateo de números según locale
- [ ] Tests de traducciones

### Files to Create — `[ACTUALIZACIÓN]` Rutas reales:

```
middleware.ts                          # Middleware de next-intl (locale routing + cache headers)
src/i18n.ts                            # Configuración de next-intl v4
messages/es.json                       # Traducciones español (completas)
messages/en.json                       # Traducciones inglés (completas)
```

### Technical Notes — `[ACTUALIZACIÓN]` next-intl v4

```typescript
// src/i18n.ts - [ACTUALIZACIÓN] next-intl v4 API
import { getRequestConfig } from 'next-intl/server';

export const locales = ['es', 'en'] as const;
export const defaultLocale = 'es' as const;

export default getRequestConfig(async ({ requestLocale }) => {
  const locale = await requestLocale;
  return {
    locale,
    messages: (await import(`../messages/${locale}.json`)).default,
  };
});

// middleware.ts - [ACTUALIZACIÓN] Combinado con cache headers
import createMiddleware from 'next-intl/middleware';
import { locales, defaultLocale } from './src/i18n';

const intlMiddleware = createMiddleware({
  locales,
  defaultLocale,
  localePrefix: 'always',
});

export default function middleware(request: NextRequest) {
  const response = intlMiddleware(request);
  response.headers.set('Cache-Control', 'no-store, must-revalidate');
  return response;
}

export const config = {
  matcher: ['/((?!api|_next|.*\\..*).*)'],
};

// messages/es.json - Estructura real con namespaces
{
  "common": { "loading": "Cargando...", "error": "Error", "retry": "Reintentar" },
  "chat": { "title": "Chat con Asistente", "inputPlaceholder": "Escribe tu mensaje...", "sendButton": "Enviar" },
  "dashboard": { "title": "Panel de Control", ... },
  "knowledge": { "upload": { "title": "Subir Documento", ... } },
  "auth": { "login": { "title": "Iniciar Sesión", ... } },
  "landing": { "hero": { "title": "...", ... } }
}

// Uso en componentes
'use client';
import { useTranslations } from 'next-intl';

export function MessageInput() {
  const t = useTranslations('chat');
  return <input placeholder={t('inputPlaceholder')} />;
}

// Dependencias - [ACTUALIZACIÓN] v4
pnpm add next-intl@^4.8.2
```

---

## Resumen y Orden de Implementación

### Fase 1: Setup Base (Paralelo)
- Issue 5.1: Setup Base Chat Page Structure
- Issue 5.2: Implement Chat State Management
- Issue 5.6: Implement Chat API Client
- Issue 5.13: Implement User Profile and Session Management
- Issue 5.15: Implement Internationalization (i18n)

### Fase 2: Componentes Core (Paralelo)
- Issue 5.3: Create Message List Component
- Issue 5.4: Create Message Input Component
- Issue 5.5: Create Source Card Component

### Fase 3: Integración
- Issue 5.7: Integrate State with API and Components

### Fase 4: Polish y Optimización (Paralelo)
- Issue 5.8: Implement Error Boundaries
- Issue 5.9: Add Loading States
- Issue 5.10: Implement Markdown Rendering
- Issue 5.11: Add Empty State
- Issue 5.14: Implement Image Optimization

### Fase 5: Testing
- Issue 5.12: E2E Testing for Chat Flow

---

## Estimación Total

**Total de horas estimadas:** 85 horas (68 + 17 de nuevos issues)  
**Total de sprints (2 semanas c/u):** ~2-3 sprints  
**Desarrolladores recomendados:** 2-3 frontend developers

---

## Dependencias Externas

### NPM Packages Requeridos — `[ACTUALIZACIÓN]` Dependencias reales del `package.json`:

```json
{
  "dependencies": {
    "@radix-ui/react-accordion": "^1.2.12",
    "@radix-ui/react-alert-dialog": "^1.1.15",
    "@radix-ui/react-dialog": "^1.1.15",
    "@radix-ui/react-select": "^2.2.6",
    "@radix-ui/react-separator": "^1.1.8",
    "@radix-ui/react-slot": "^1.2.4",
    "@radix-ui/react-toast": "^1.2.15",
    "@radix-ui/react-tooltip": "^1.2.8",
    "@sentry/nextjs": "^8.45.1",
    "@tanstack/react-query": "^5.62.14",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "date-fns": "^4.1.0",
    "lucide-react": "^0.468.0",
    "next": "16.1.6",
    "next-auth": "5.0.0-beta.30",
    "next-intl": "^4.8.2",
    "radix-ui": "^1.4.3",
    "react": "19.2.3",
    "react-dom": "19.2.3",
    "react-markdown": "^10.1.0",
    "react-syntax-highlighter": "^16.1.0",
    "remark-gfm": "^4.0.1",
    "tailwind-merge": "^2.5.5",
    "zod": "^3.24.1",
    "zustand": "^5.0.3"
  },
  "devDependencies": {
    "@playwright/test": "^1.49.1",
    "@tailwindcss/postcss": "^4",
    "@tanstack/eslint-plugin-query": "^5.62.14",
    "@testing-library/jest-dom": "^6.9.1",
    "@testing-library/react": "^16.3.2",
    "@testing-library/user-event": "^14.6.1",
    "@types/react-syntax-highlighter": "^15.5.13",
    "@vitest/coverage-v8": "^4.0.18",
    "eslint": "^9",
    "eslint-config-next": "16.1.6",
    "eslint-config-prettier": "^9.1.0",
    "eslint-plugin-jsx-a11y": "^6.10.2",
    "eslint-plugin-sonarjs": "^3.0.1",
    "husky": "^9.1.7",
    "jsdom": "^28.0.0",
    "lint-staged": "^15.3.0",
    "prettier": "^3.4.2",
    "prettier-plugin-tailwindcss": "^0.6.10",
    "tailwindcss": "^4",
    "typescript": "^5",
    "vitest": "^4.0.18",
    "vitest-axe": "^0.1.0"
  }
}
```

> **Nota:** No se usa `axios`. El cliente HTTP se implementa con `fetch` nativo via wrappers en `src/lib/api/client.ts`. Los componentes UI se basan en **shadcn/ui** (construido sobre Radix UI primitives + Tailwind CSS + `class-variance-authority`).

---

## Notas Importantes

1. **TDD Approach**: Seguir ciclo Red-Green-Refactor en cada issue
2. **TypeScript Strict**: Todos los componentes deben tener tipos completos
3. **Accessibility**: Considerar ARIA labels y navegación por teclado
4. **Performance**: Implementar code splitting y lazy loading donde corresponda
5. **Mobile First**: Diseño responsivo desde el inicio

---

## Validación de Completitud

La Fase 5 se considera completa cuando:

- [ ] Todos los 15 issues están completados
- [ ] Tests E2E pasan exitosamente
- [ ] Coverage de tests >= 80%
- [ ] `pnpm lint` pasa sin errores
- [ ] `pnpm build` genera build exitoso
- [ ] Performance Lighthouse score >= 90
- [ ] Accesibilidad Lighthouse score >= 90
- [ ] Internacionalización funciona correctamente (ES/EN)
- [ ] Imágenes optimizadas con Next.js Image

