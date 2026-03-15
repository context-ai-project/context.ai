# Especificación de Requisitos V2: Generación de Cápsulas Multimedia
## Context.ai — Módulo de Contenido Multimedia con IA

> **Versión**: 2.0  
> **Fecha**: Febrero 2026  
> **Estado**: Diseño — En análisis de requisitos  
> **Prerequisito**: v1.3.0 (Admin + RBAC + Sectores + RAG Chat)  
> **Caso de Uso base**: UC3 — Generar Cápsulas Multimedia (ver `002-use-cases.md`)

---

## 1. Propósito y Contexto de Negocio

### 1.1 Visión General

La **Generación de Cápsulas Multimedia** es la evolución natural de Context.ai hacia un sistema de onboarding y formación más inmersivo. Mientras la v1 se centró en la ingesta documental y el chat RAG (texto), la v2 transforma el conocimiento escrito en **contenido audiovisual explicativo** — videos y audios generados por IA — que facilitan el aprendizaje y la retención del conocimiento.

### 1.2 Problema que resuelve

| Problema | Impacto | Solución V2 |
|----------|---------|-------------|
| Documentos extensos y poco atractivos | Baja tasa de lectura completa por nuevos integrantes | Videos explicativos de hasta 40 segundos generados por IA |
| No todos los empleados aprenden leyendo | Brecha de comprensión en perfiles no técnicos | Audio-guías con narración profesional (TTS) |
| RRHH no tiene recursos para grabar videos | Tiempo excesivo en producción de material de onboarding | Generación automatizada de guiones + video + audio |
| Contenido desactualizado rápidamente | Los manuales se actualizan pero los videos no | Crear nuevas cápsulas desde documentos actualizados y archivar las anteriores |

### 1.3 Propuesta de valor

> **"De un documento de 50 páginas a un video explicativo de 40 segundos o un audio narrado, con un clic."**

El módulo permite a los usuarios con rol **admin** o **manager** seleccionar documentos previamente subidos a un sector de conocimiento, redactar un texto introductorio personalizado, y — mediante inteligencia artificial — generar automáticamente:

1. Un **guión narrativo** estructurado a partir del contenido documental (IA generativa con Gemini)
2. Un **audio profesional** del guión (Text-to-Speech con ElevenLabs)
3. Un **video explicativo** con narrativa visual (generación de video con Google Veo 3)

Los empleados con rol **user** tendrán acceso exclusivamente a la **visualización y reproducción** de las cápsulas generadas, dentro de los sectores a los que tienen acceso.

---

## 2. Análisis del Mockup

### 2.1 Descripción visual de la interfaz

La interfaz de creación de cápsulas utiliza un **wizard por pasos (stepper)** que guía al usuario de forma secuencial. El diseño se adapta a los estilos de Context.ai.

#### Barra de progreso (Stepper) — Siempre visible en la parte superior

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│     ① Seleccionar contenido           ② Crear cápsula                           │
│     ●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━○                                         │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

#### Step 1 — Seleccionar contenido (Tipo + Sector + Documentos)

Pantalla de ancho completo dedicada a la selección del tipo de cápsula, el sector y los documentos fuente.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Crear nueva cápsula                                                             │
│  "Selecciona el tipo, sector y los documentos que servirán de base"             │
│                                                                                  │
│  Tipo de cápsula                                                                 │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐   │
│  │                      │  │                      │  │                      │   │
│  │  🎬  Video           │  │  🔊  Audio           │  │  🎬🔊 Ambos          │   │
│  │  Video explicativo   │  │  Audio narrado con   │  │  Video + Audio       │   │
│  │  con narración       │  │  voz profesional     │  │  independiente       │   │
│  │  profesional         │  │  (podcast)           │  │  descargable         │   │
│  │                      │  │                      │  │                      │   │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘   │
│                                                                                  │
│  ┌─── Sector ─────────────────────────────────────────────────────────────────┐  │
│  │  ▼  Selecciona un sector...                                                │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  Título de la cápsula                                                            │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Ej: "Onboarding — Política de vacaciones"                                │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  Documentos disponibles en el sector                                             │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  🔍 Buscar documento...                                                    │  │
│  ├────────────────────────────────────────────────────────────────────────────┤  │
│  │  ☐  📄 Manual de Vacaciones.pdf          12 págs · 2.3 MB · 15 Feb 2026  │  │
│  │  ☐  📄 Política de Trabajo Remoto.md     8 págs  · 45 KB  · 10 Feb 2026  │  │
│  │  ☐  📄 Guía de Beneficios.pdf            24 págs · 5.1 MB · 8 Feb 2026   │  │
│  │  ☐  📄 Código de Conducta.pdf            6 págs  · 1.2 MB · 1 Feb 2026   │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  Documentos seleccionados: 0                                                     │
│                                                                                  │
│                                              ┌────────────────────────────────┐  │
│                                              │       Siguiente →              │  │
│                                              └────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Comportamiento del Step 1**:
- **Tipo de cápsula**: El usuario elige entre tres opciones mediante tarjetas seleccionables (toggle cards). Solo una puede estar activa a la vez:
  - 🎬 **Video**: Genera un video explicativo con narración profesional (ElevenLabs + Veo 3). Consume 1 crédito del límite mensual de 10 videos
  - 🔊 **Audio**: Genera únicamente un audio narrado con voz profesional (ElevenLabs). No consume créditos de video
  - 🎬🔊 **Ambos**: Genera video y audio como recursos independientes. El video incluye la narración y además se genera un archivo de audio descargable por separado. Consume 1 crédito de video
- El dropdown de sector muestra solo los sectores activos a los que el usuario tiene acceso
- Al seleccionar un sector, se cargan dinámicamente los documentos con estado `COMPLETED` de ese sector
- Se puede buscar documentos por nombre dentro del listado
- Se requiere tipo de cápsula, al menos 1 documento seleccionado y un título para avanzar al paso 2
- El botón "Siguiente →" se habilita solo cuando la selección es válida (tipo + sector + título + ≥1 documento)

#### Step 2 — Crear cápsula (Split-panel: Formulario + Vista previa)

Pantalla dividida en dos columnas: formulario de contenido (izquierda) y vista previa (derecha).

```
┌────────────────────────────────────────┬─────────────────────────────────────────┐
│  Panel izquierdo — Contenido           │  Panel derecho — Vista previa           │
│                                        │                                         │
│  Docs seleccionados:                   │                                         │
│  ┌──────────────────────────────────┐  │           ╭───────────────────╮         │
│  │ 📄 Manual de Vacaciones.pdf  ✕  │  │           │                   │         │
│  │ 📄 Guía de Beneficios.pdf   ✕  │  │           │   🎵 ~~~~~~~~~~~~ │         │
│  └──────────────────────────────────┘  │           │       ▶          │         │
│                                        │           │   ~~~~~~~~~~~~    │         │
│  Texto introductorio                   │           │                   │         │
│  ┌──────────────────────────────────┐  │           ╰───────────────────╯         │
│  │ Escribe un texto personalizado   │  │                                         │
│  │ para el inicio del video...      │  │     "Genera un guión para comenzar"     │
│  └──────────────────────────────────┘  │                                         │
│                                        │                                         │
│  Guión narrativo                       │  ┌─────────────────────────────────────┐│
│  ┌──────────────────────────────────┐  │  │         ⬇ Descargar video          ││
│  │ El guión narrativo aparecerá     │  │  └─────────────────────────────────────┘│
│  │ aquí después de generarlo con    │  │                                         │
│  │ IA. También puedes escribirlo    │  │  ┌─────────────────────────────────────┐│
│  │ manualmente...                   │  │  │         🔊 Descargar audio          ││
│  └──────────────────────────────────┘  │  └─────────────────────────────────────┘│
│                                        │                                         │
│  ┌──────────────────────────────────┐  │                                         │
│  │  🤖 Generar guión con IA        │  │                                         │
│  └──────────────────────────────────┘  │                                         │
│                                        │                                         │
│  Seleccionar voz                       │                                         │
│  ┌──────────────────────────────────┐  │                                         │
│  │  ▼  María (Español, femenina)    │  │                                         │
│  └──────────────────────────────────┘  │                                         │
│                                        │                                         │
│  ┌────────────┐  ┌──────────────────┐  │                                         │
│  │ ← Anterior │  │ 📦 Generar Cáps. │  │                                         │
│  └────────────┘  └──────────────────┘  │                                         │
└────────────────────────────────────────┴─────────────────────────────────────────┘
```

**Comportamiento del Step 2**:
- Los documentos seleccionados en el Step 1 se muestran como chips removibles (se puede volver al Step 1 para cambiarlos)
- "Generar guión con IA" utiliza Gemini 2.5 Flash para crear el guión a partir de los fragmentos RAG
- El guión generado es editable manualmente en el textarea
- El selector de voz permite elegir entre las voces disponibles de ElevenLabs
- El panel derecho muestra inicialmente un placeholder ("Genera un guión para comenzar")
- Tras generar el guión, se puede previsualizar el audio
- "← Anterior" regresa al Step 1 sin perder el progreso del formulario

**Comportamiento adaptado según tipo de cápsula (seleccionado en Step 1)**:

| Elemento | 🎬 Video | 🔊 Audio | 🎬🔊 Ambos |
|----------|----------|----------|------------|
| Botón de generación | "📦 Generar Video" | "📦 Generar Audio" | "📦 Generar Video y Audio" |
| Pipeline | Guión → Audio → Video | Guión → Audio | Guión → Audio → Video + Audio independiente |
| Vista previa resultado | Reproductor video + descarga video | Reproductor audio + descarga audio | Reproductor video + descarga video y audio |
| Indicador de progreso | Audio... → Video... → Listo | Audio... → Listo | Audio... → Video... → Listo |
| Consumo de cuota | 1 crédito video/mes | Sin crédito video | 1 crédito video/mes |

### 2.2 Flujo de interacción completo (Wizard)

```
Step 1: Seleccionar contenido
├─ 1a. Seleccionar tipo de cápsula: 🎬 Video, 🔊 Audio o 🎬🔊 Ambos (toggle cards)
├─ 1b. Seleccionar sector (dropdown)
├─ 1c. Escribir título de la cápsula
├─ 1d. Seleccionar uno o más documentos del sector
└─ 1e. Clic en "Siguiente →"

Step 2: Crear cápsula (se adapta al tipo seleccionado)
├─ 2a. (Opcional) Escribir texto introductorio personalizado
├─ 2b. Clic en "Generar guión con IA" → IA genera guión narrativo
├─ 2c. (Opcional) Editar el guión manualmente
├─ 2d. Seleccionar voz TTS (ElevenLabs)
├─ 2e. Clic en "Generar" → Pipeline según tipo seleccionado
│   ├─ 🎬 Video: audio (ElevenLabs) → video (Veo 3) → merge
│   ├─ 🔊 Audio: audio (ElevenLabs) → listo
│   └─ 🎬🔊 Ambos: audio (ElevenLabs) → video (Veo 3) → merge + audio independiente
├─ 2f. Indicador de progreso adaptado al pipeline
├─ 2g. Panel derecho muestra reproductor con el resultado
└─ 2h. Botones de descarga según tipo (video, audio, o ambos)
```

### 2.3 Estados del panel de vista previa (Step 2 — Panel derecho)

| Estado | 🎬 Video | 🔊 Audio | 🎬🔊 Ambos |
|--------|----------|----------|------------|
| **Inicial** | Placeholder: "Genera un guión para comenzar" | Placeholder: "Genera un guión para comenzar" | Placeholder: "Genera un guión para comenzar" |
| **Guión generado** | Botón "Previsualizar audio" habilitado | Botón "Previsualizar audio" habilitado | Botón "Previsualizar audio" habilitado |
| **Audio generado** | Reproductor de audio (preview) | Reproductor de audio (final) + descargar | Reproductor de audio (preview) + descargar audio |
| **Generando video** | Progreso: "Generando video..." | *(no aplica)* | Progreso: "Generando video..." |
| **Todo listo** | Reproductor video + descargar video | *(no aplica)* | Reproductor video + descargar video y audio |
| **Error** | Error + botón "Reintentar" | Error + botón "Reintentar" | Error + botón "Reintentar" |

---

## 3. Requisitos Funcionales (RF)

### RF-V2-1: Generación de Guión Narrativo con IA

**Descripción**: El sistema generará automáticamente un guión narrativo estructurado a partir de uno o varios documentos seleccionados de un sector, utilizando el modelo LLM Gemini 2.5 Flash.

**Criterios de Aceptación**:
- [ ] El usuario (admin/manager) selecciona uno o más documentos de un sector específico
- [ ] El sistema utiliza el contenido del documento (fragmentos RAG existentes) para generar un guión coherente
- [ ] El guión incluye: introducción, desarrollo por secciones del documento, y cierre
- [ ] Si el usuario proporcionó un texto introductorio, se incorpora al inicio del guión
- [ ] El guión generado es editable antes de continuar con la generación multimedia
- [ ] El guión tiene un límite de **~1500 palabras** (equivalente a ~2-3 minutos de narración, alineado con `maxOutputTokens: 2048` de la configuración actual de Gemini)
- [ ] El sistema muestra un indicador de progreso durante la generación

**Flujo técnico**:
```
Documentos seleccionados → Recuperar fragmentos RAG (Pinecone)
  → Construir prompt con contexto + texto introductorio
    → Gemini 2.5 Flash genera guión narrativo
      → Retornar guión editable al frontend
```

> **📄 Nota**: El contrato del prompt (system prompt, estructura del guión, formato de salida, parámetros de temperatura, etc.) se definirá en un **documento separado** (`020-v2-prompt-contracts.md`) para permitir iteración independiente del prompt engineering sin modificar este documento de requisitos.

### RF-V2-2: Generación de Audio (Text-to-Speech)

**Descripción**: A partir del guión narrativo (generado o editado manualmente), el sistema sintetizará un audio profesional con voz natural usando la API de ElevenLabs.

**Criterios de Aceptación**:
- [ ] El sistema convierte el guión narrativo a audio de alta calidad
- [ ] Se permite seleccionar entre las voces disponibles del plan Starter de ElevenLabs (cantidad y selección pendiente de confirmar según plan contratado)
- [ ] El audio se genera en formato MP3 (compatibilidad universal)
- [ ] Se soporta texto en español e inglés (alineado con i18n de la plataforma)
- [ ] La duración máxima del audio es de ~3 minutos por cápsula (alineado con el límite del guión de ~1500 palabras)
- [ ] El audio generado se almacena y puede ser reproducido/descargado de forma independiente
- [ ] El usuario puede previsualizar el audio antes de generar el video

**Flujo técnico**:
```
Guión narrativo (texto)
  → IAudioGenerator (ElevenLabs adapter) → TTS
    → Audio MP3 de alta calidad
      → IMediaStorage (GCS adapter) → Almacenamiento en Cloud Storage
        → URL firmada para reproducción
```

### RF-V2-3: Generación de Video con IA

**Descripción**: El sistema generará un video explicativo a partir del guión narrativo y el audio generado, utilizando Google Veo 3 para la generación visual.

**Criterios de Aceptación**:
- [ ] El sistema genera un video a partir del guión narrativo y el audio generado
- [ ] El video incorpora el audio generado por ElevenLabs como narración
- [ ] Resolución mínima del video: 720p (1280x720)
- [ ] Duración máxima del video: **40 segundos** por cápsula (5 segmentos de 8 seg concatenados con FFmpeg)
- [ ] El video se genera en formato MP4
- [ ] Se muestra un indicador de progreso con estimación de tiempo
- [ ] El video generado se almacena y queda disponible para reproducción y descarga
- [ ] El sistema controla la cuota mensual: máximo **10 cápsulas de video/mes** (1000 puntos del plan Pro, 100 puntos por cápsula)

**Flujo técnico**:
```
Guión narrativo + Audio
  → IVideoGenerator (Veo 3 adapter) → Generación de segmentos de video
    → IMediaProcessor (FFmpeg adapter) → merge audio + video + concatenación
      → IMediaStorage (GCS adapter) → Almacenamiento en Cloud Storage
        → URL firmada para reproducción
```

### RF-V2-4: Gestión de Cápsulas (CRUD)

**Descripción**: Los usuarios con rol admin o manager podrán gestionar las cápsulas multimedia creadas.

**Criterios de Aceptación**:
- [ ] Crear una nueva cápsula (flujo completo descrito arriba)
- [ ] Listar cápsulas existentes por sector (con paginación y filtros)
- [ ] Ver detalle de una cápsula (metadatos, reproductor, estado)
- [ ] Eliminar una cápsula (soft delete — marca como inactiva)
- [ ] Cada cápsula tiene un título, sector asociado, guión, estado y tipo (video/audio/both)
- [ ] Se registra quién creó la cápsula y cuándo

> **Nota**: La regeneración de cápsulas (re-generar multimedia desde documentos actualizados) no se incluye en esta versión. El flujo alternativo es: archivar la cápsula existente y crear una nueva. Esto evita consumo innecesario de tokens y puntos de cuota.

**Estados de una cápsula**:

```
┌─────────┐    ┌────────────┐    ┌───────────┐    ┌───────────┐
│  DRAFT  │───▶│ GENERATING │───▶│ COMPLETED │───▶│  ACTIVE   │
└─────────┘    └────────────┘    └───────────┘    └───────────┘
                     │                                    │
                     ▼                                    ▼
               ┌──────────┐                        ┌──────────┐
               │  FAILED  │                        │ ARCHIVED │
               └──────────┘                        └──────────┘
```

| Estado | Descripción |
|--------|-------------|
| `DRAFT` | Cápsula creada, guión en edición, aún no se ha generado multimedia |
| `GENERATING` | Pipeline de generación en ejecución (audio y/o video) |
| `COMPLETED` | Generación finalizada exitosamente, pendiente de publicar |
| `ACTIVE` | Publicada y visible para los usuarios del sector |
| `FAILED` | Error en la generación — ver política de manejo de fallos abajo |
| `ARCHIVED` | Retirada de la vista de usuarios, pero conserva datos |

**Política de manejo de fallos parciales**:

El pipeline de generación tiene múltiples etapas (guión → audio → video → post-procesamiento). Si una etapa falla:

| Etapa que falla | Comportamiento | Razón |
|:----------------|:---------------|:------|
| **Generación de guión** | Se marca como `FAILED`. El usuario puede editar el prompt o escribir el guión manualmente | El guión es editable, no se pierde nada |
| **Generación de audio** | Se marca como `FAILED`. Se conserva el guión generado. El usuario puede reintentar **solo la generación de audio** | El guión ya está guardado; no se regenera para evitar consumo de tokens |
| **Generación de video** | Se marca como `FAILED`. Se conservan guión y audio. El usuario puede reintentar **solo la generación de video** | El audio ya está almacenado en GCS; no se regenera para evitar consumo de caracteres ElevenLabs |
| **Post-procesamiento (FFmpeg)** | Se marca como `FAILED`. Se conservan todos los archivos. El usuario puede reintentar **solo el merge** | Los archivos de audio y video ya existen; solo falta el merge |

> **Principio clave**: **NO regenerar etapas ya completadas**. Cada reintento retoma desde la etapa fallida, conservando los artefactos previos. Esto minimiza el consumo de tokens (Gemini), caracteres (ElevenLabs) y puntos de cuota (Veo 3).

### RF-V2-5: Selección de Tipo, Sector y Documentos Fuente (Step 1 del Wizard)

**Descripción**: El primer paso del wizard de creación de cápsulas permite al usuario elegir el tipo de cápsula (video, audio o ambos), seleccionar un sector y uno o más documentos ya indexados como fuente de contenido.

**Criterios de Aceptación**:
- [ ] Step 1 es la pantalla inicial del wizard de creación
- [ ] El usuario selecciona el tipo de cápsula mediante tarjetas: 🎬 **Video**, 🔊 **Audio** o 🎬🔊 **Ambos**
- [ ] Solo un tipo puede estar activo a la vez (toggle exclusivo)
- [ ] Si se selecciona Video o Ambos y la cuota mensual (10 videos) está agotada, se muestra advertencia y se sugiere crear Audio
- [ ] El dropdown de sector muestra solo sectores activos accesibles por el usuario
- [ ] Al seleccionar un sector, se cargan dinámicamente los documentos con estado `COMPLETED`
- [ ] Se puede buscar documentos por nombre dentro del listado del sector
- [ ] Se puede seleccionar uno o múltiples documentos como fuente (checkbox)
- [ ] Se muestra título, número de páginas, tamaño y fecha de subida de cada documento
- [ ] Se requiere un título para la cápsula antes de continuar
- [ ] El botón "Siguiente →" se habilita solo cuando hay tipo + sector + título + al menos 1 documento seleccionado
- [ ] No se permite subir documentos nuevos desde esta pantalla (se redirige al módulo de Documentos)
- [ ] El sistema extrae el contenido de los documentos seleccionados a partir de los fragmentos RAG ya indexados en Pinecone

### RF-V2-6: Reproducción de Cápsulas (Vista de Usuario)

**Descripción**: Los empleados (rol `user`) accederán a una vista de solo lectura donde podrán reproducir los videos y audios generados para los sectores a los que tienen acceso.

**Criterios de Aceptación**:
- [ ] Nueva sección "Cápsulas" en el sidebar (visible para todos los roles autenticados)
- [ ] Lista de cápsulas activas filtradas por los sectores asignados al usuario
- [ ] Reproductor de video integrado (HTML5 `<video>`)
- [ ] Reproductor de audio integrado para cápsulas de solo audio
- [ ] Información de la cápsula: título, descripción (resumen del guión), sector, fecha de creación, duración
- [ ] El usuario NO puede editar, eliminar ni crear cápsulas (solo visualizar y reproducir)
- [ ] Posibilidad de descargar el video/audio para consumo offline
- [ ] Responsive: reproducción funcional en desktop y mobile

### RF-V2-7: Vista de Creación/Gestión — Wizard por Pasos (Vista de Admin/Manager)

**Descripción**: Interfaz dedicada para que admin y manager creen cápsulas multimedia mediante un wizard de 2 pasos que guía el proceso de forma secuencial.

**Criterios de Aceptación**:

**Wizard general**:
- [ ] Barra de progreso (stepper) visible en la parte superior con los 2 pasos
- [ ] El paso actual se resalta visualmente
- [ ] Se puede navegar entre pasos sin perder el progreso del formulario
- [ ] Validación por paso: no se puede avanzar si faltan datos requeridos

**Step 1 — Seleccionar contenido** (pantalla completa):
- [ ] Selector de tipo de cápsula: tarjetas 🎬 Video / 🔊 Audio / 🎬🔊 Ambos (toggle exclusivo)
- [ ] Indicador de cuota de videos restantes (visible al seleccionar Video o Ambos)
- [ ] Dropdown de sector (solo sectores activos accesibles al usuario)
- [ ] Campo de título de la cápsula (obligatorio)
- [ ] Lista de documentos del sector con checkboxes para selección múltiple
- [ ] Buscador de documentos por nombre
- [ ] Contador de documentos seleccionados
- [ ] Botón "Siguiente →" habilitado solo con tipo + sector + título + ≥1 documento

**Step 2 — Crear cápsula** (split-panel, se adapta al tipo seleccionado en Step 1):
- [ ] Panel izquierdo: chips de documentos seleccionados, texto introductorio, guión narrativo, selector de voz
- [ ] Botón "Generar guión con IA" para generación automática del guión
- [ ] Selector de voz TTS (ElevenLabs) con opciones por idioma y género
- [ ] Botón "← Anterior" para volver al Step 1
- [ ] Botón de generación adaptado: "Generar Video", "Generar Audio" o "Generar Video y Audio" según tipo
- [ ] Panel derecho: vista previa con reproductor adaptado al tipo (video o audio)
- [ ] Botones de descarga adaptados: video (tipo Video), audio (tipo Audio), o video + audio (tipo Ambos)
- [ ] Indicadores de progreso adaptados al pipeline del tipo
- [ ] Mensajes de error claros con opción de reintentar si alguna etapa falla

---

## 4. Requisitos No Funcionales (RNF)

### RNF-V2-1: Rendimiento

| Métrica | Objetivo | Justificación |
|---------|----------|---------------|
| Generación de guión | < 15 segundos | Gemini 2.5 Flash — baja latencia |
| Generación de audio (1 min) | < 30 segundos | ElevenLabs API — procesamiento stream |
| Generación de video (40 seg) | 3-8 minutos | Veo 3 es computacionalmente intensivo (5 segmentos de 8 seg) |
| Carga de la lista de cápsulas | < 2 segundos | Query optimizada con paginación |
| Inicio de reproducción (streaming) | < 3 segundos | URLs firmadas + CDN |

### RNF-V2-2: Procesamiento Asíncrono

La generación de video es una operación de larga duración (minutos). El sistema **no debe bloquear** la interfaz del usuario:

- La generación se ejecuta en segundo plano (job asíncrono o event-driven)
- El frontend realiza **polling** periódico al endpoint `GET /capsules/:id/status` para consultar el estado de la generación
- El usuario puede navegar libremente mientras se genera la cápsula

> **Nota**: Las notificaciones in-app y por email sobre el estado de la generación se implementarán en una **versión futura** del sistema de notificaciones. En la v2 inicial, el usuario consultará el estado mediante polling desde el panel de vista previa o desde el listado de cápsulas.

### RNF-V2-3: Almacenamiento de Archivos Multimedia

| Tipo de archivo | Almacenamiento | Retención |
|-----------------|---------------|-----------|
| Audio (MP3) | Google Cloud Storage | Mientras la cápsula exista |
| Video (MP4) | Google Cloud Storage | Mientras la cápsula exista |
| Archivos multimedia generados | Google Cloud Storage | Mientras la cápsula exista |
| Guión narrativo | PostgreSQL (campo texto) | Mientras la cápsula exista |

- Los archivos multimedia se sirven mediante **URLs firmadas** (Signed URLs) con expiración de 1 hora
- Las URLs firmadas se generan bajo demanda a través del endpoint `GET /capsules/:id/download/:type`, evitando almacenar URLs que expiran
- El frontend solicita una nueva URL firmada cada vez que el usuario reproduce o descarga contenido
- Se configura un bucket dedicado: `context-ai-capsules-{env}` (acceso **privado**, no público)
- Política de ciclo de vida: archivos de cápsulas `ARCHIVED` se mueven a Coldline Storage tras 90 días

> **Justificación de URLs firmadas**: Dado que los contenidos multimedia pueden contener información sensible de la empresa (derivada de documentos internos), es necesario que los archivos en GCS **no sean públicos**. Las URLs firmadas garantizan que solo usuarios autenticados con los permisos adecuados puedan acceder al contenido, y que el acceso expire automáticamente. La alternativa de un proxy backend consumiría demasiado ancho de banda del servidor para archivos de video.

### RNF-V2-4: Escalabilidad

- Máximo de **10 cápsulas de video por mes** (plan Pro de Google: 1000 puntos/mes, 100 puntos por cápsula de 40 seg)
- Sin límite práctico para generación de audio (ElevenLabs escala según plan)
- Sin límite práctico para generación de guiones (Gemini 2.5 Flash)
- El sistema debe mostrar un **contador de cuota** para informar cuántos videos quedan disponibles en el mes
- Desglose de cuota: cada cápsula de video consume 5 segmentos × 20 puntos = 100 puntos

### RNF-V2-5: Observabilidad

- Logging detallado de cada etapa del pipeline de generación
- Métricas de duración de generación (guión, audio, video) en Sentry/Genkit UI
- Trazabilidad de costes por generación (tokens Gemini + caracteres ElevenLabs + créditos Veo 3)
- Dashboard en el admin con estadísticas: cápsulas generadas, tasa de éxito/fallo, consumo de cuota

### RNF-V2-6: Arquitectura (Alineado con v1)

El módulo de cápsulas se implementará siguiendo la misma arquitectura que el resto de Context.ai:

- **Clean Architecture**: Domain → Application → Infrastructure → API
- **Módulo NestJS**: `CapsuleModule` independiente con dependencia de `KnowledgeModule`, `SectorsModule`
- **Frontend**: Componentes React en `src/components/capsules/` con rutas protegidas
- **Estado**: Zustand store para gestión de estado del wizard de creación

#### Patrón de Abstracción para Proveedores de IA (Adapter Pattern)

Todos los servicios externos de IA se encapsularán detrás de **interfaces de dominio**, siguiendo el mismo patrón que `IVectorStore` (Pinecone) en la v1. Esto permite **cambiar de proveedor** (ej. Veo 3 → Runway, ElevenLabs → otra TTS) modificando **solo la implementación en infraestructura**, sin afectar use cases, controllers, DTOs, base de datos ni frontend.

**Interfaces de dominio** (`domain/services/`):

```typescript
// ─── IVideoGenerator ─────────────────────────────────────────────
// Contrato para generación de video. Implementaciones: Veo3Adapter, RunwayAdapter, etc.
export interface IVideoGenerator {
  generateSegment(prompt: string, durationSeconds: number): Promise<VideoSegmentResult>;
  getSegmentStatus(operationId: string): Promise<VideoGenerationStatus>;
  getQuotaInfo(): Promise<VideoQuotaInfo>;
}

export interface VideoSegmentResult {
  operationId: string;
  status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED';
  videoUrl?: string;
  durationSeconds?: number;
}

export interface VideoGenerationStatus {
  operationId: string;
  status: 'PENDING' | 'PROCESSING' | 'COMPLETED' | 'FAILED';
  progress?: number; // 0-100
  videoUrl?: string;
  errorMessage?: string;
}

export interface VideoQuotaInfo {
  used: number;      // Puntos usados este mes
  limit: number;     // Puntos totales del plan
  remaining: number; // Puntos restantes
  capsulesCost: number;     // Puntos por cápsula
  capsulesRemaining: number; // Cápsulas que se pueden generar
}
```

```typescript
// ─── IAudioGenerator ─────────────────────────────────────────────
// Contrato para generación de audio (TTS). Implementaciones: ElevenLabsAdapter, etc.
export interface IAudioGenerator {
  generateAudio(text: string, options: AudioGenerationOptions): Promise<AudioResult>;
  getAvailableVoices(): Promise<VoiceInfo[]>;
}

export interface AudioGenerationOptions {
  voiceId: string;
  language: 'es' | 'en';
  format?: 'mp3' | 'wav';
}

export interface AudioResult {
  audioBuffer: Buffer;
  durationSeconds: number;
  format: string;
}

export interface VoiceInfo {
  id: string;
  name: string;
  language: string;
  preview_url?: string;
}
```

```typescript
// ─── IMediaStorage ───────────────────────────────────────────────
// Contrato para almacenamiento de archivos multimedia. Implementación: GcsStorageAdapter.
export interface IMediaStorage {
  upload(file: Buffer, path: string, contentType: string): Promise<string>;
  getSignedUrl(path: string, expiresInMinutes?: number): Promise<string>;
  delete(path: string): Promise<void>;
}
```

**Inyección de dependencias en NestJS** (`CapsuleModule`):

```typescript
@Module({
  providers: [
    // Cambiar de proveedor = cambiar 1 línea por servicio
    { provide: 'IVideoGenerator',  useClass: Veo3VideoGeneratorService },
    { provide: 'IAudioGenerator',  useClass: ElevenLabsAudioService },
    { provide: 'IMediaStorage',    useClass: GcsStorageService },
    // Use cases inyectan las interfaces, nunca las implementaciones
    GenerateCapsuleUseCase,
    // ...
  ],
})
```

**Costo de cambio de proveedor** (ej. Veo 3 → Runway):

| Componente | ¿Cambia? | Esfuerzo estimado |
|:-----------|:--------:|:-----------------:|
| Interfaz `IVideoGenerator` (dominio) | ❌ No | 0h |
| Use cases (application) | ❌ No | 0h |
| Controllers / DTOs (presentation) | ❌ No | 0h |
| Base de datos / Modelos | ❌ No | 0h |
| Frontend (React) | ❌ No | 0h |
| BullMQ Jobs | ❌ No | 0h |
| FFmpeg post-procesamiento | ⚠️ Ajustes menores | ~1-2h |
| **Nueva implementación** (`RunwayVideoGeneratorService`) | ✅ Crear | ~4-6h |
| **Binding en Module** | ✅ 1 línea | ~15min |
| **Config / Env vars** | ✅ Nuevas keys | ~30min |
| **Tests del nuevo adapter** | ✅ Crear | ~2-3h |
| **Total estimado** | — | **~1 día (8-12h)** |

> **Principio clave**: Los use cases (`GenerateCapsuleUseCase`, `GenerateScriptUseCase`) dependen de `IVideoGenerator` y `IAudioGenerator` (interfaces del dominio), nunca de `Veo3VideoGeneratorService` ni `ElevenLabsAudioService` (implementaciones de infraestructura). Esto es el **Dependency Inversion Principle (DIP)** que ya aplica Context.ai en la v1 con `IVectorStore` → `PineconeVectorStoreService`.

#### Actualizaciones en `context-ai-shared`

El paquete compartido (`context-ai-shared`) deberá extenderse con los siguientes artefactos para mantener la consistencia de tipos entre frontend y backend:

| Artefacto | Ruta en shared | Descripción |
|-----------|---------------|-------------|
| `CapsuleType` (enum) | `src/types/enums/capsule-type.enum.ts` | `VIDEO`, `AUDIO`, `BOTH` |
| `CapsuleStatus` (enum) | `src/types/enums/capsule-status.enum.ts` | `DRAFT`, `GENERATING`, `COMPLETED`, `ACTIVE`, `FAILED`, `ARCHIVED` |
| `CapsuleGenerationStep` (enum) | `src/types/enums/capsule-generation-step.enum.ts` | `SCRIPT`, `AUDIO`, `VIDEO`, `POSTPROCESS` |
| `CapsuleDto` (DTO) | `src/dto/capsule/capsule.dto.ts` | DTO de respuesta de cápsula |
| `CreateCapsuleDto` (DTO) | `src/dto/capsule/create-capsule.dto.ts` | DTO de creación (title, sectorId, type, sourceIds) |
| `UpdateCapsuleDto` (DTO) | `src/dto/capsule/update-capsule.dto.ts` | DTO de actualización (script, introText) |
| `CapsuleQuotaDto` (DTO) | `src/dto/capsule/capsule-quota.dto.ts` | DTO de cuota (used, limit, remaining) |
| `CapsuleType` (type) | `src/types/entities/capsule.type.ts` | Tipo de entidad compartido |
| Re-exportaciones | `src/dto/capsule/index.ts`, `src/types/enums/index.ts` | Barrel exports actualizados |

### RNF-V2-7: Internacionalización (i18n)

- Toda la UI del módulo disponible en **español** e **inglés** (alineado con i18n existente via next-intl)
- Las voces TTS de ElevenLabs soportan ambos idiomas
- El guión se genera en el idioma del contenido del documento fuente
- Los prompts del LLM adaptan el idioma según la configuración del usuario

---

## 5. Restricciones Técnicas

### 5.1 Stack Tecnológico V2

| Componente | Tecnología | Justificación |
|:-----------|:-----------|:--------------|
| **Generación de video** | Google Veo 3 (API vía Vertex AI / Google AI) | Modelo de vanguardia de Google para video generativo. Capacidad de crear contenido visual a partir de prompts de texto |
| **Generación de audio (TTS)** | ElevenLabs API | Voces ultra-realistas con baja latencia. Soporte multilingüe (ES/EN). Streaming y batch |
| **Generación de guión** | Gemini 2.5 Flash (Google Genkit) | Reutiliza el LLM ya integrado en la v1 para generación de texto. Baja latencia y amplia ventana de contexto |
| **Almacenamiento multimedia** | Google Cloud Storage (GCS) | Almacenamiento escalable, URLs firmadas, integración nativa con Veo 3 |
| **Post-procesamiento video** | FFmpeg (vía fluent-ffmpeg) | Merge de audio + video, conversión de formatos, extracción de thumbnails |
| **Backend** | NestJS 11+ (TypeScript) | Mismo framework que v1. Módulo `CapsuleModule` |
| **Frontend** | Next.js 16+ (React 19) | Mismo framework que v1. Componentes en App Router |
| **Base de datos** | PostgreSQL 16 + TypeORM | Metadatos de cápsulas. Mismo stack que v1 |
| **Job processing** | BullMQ + Redis (o procesamiento event-driven) | La generación de video requiere procesamiento asíncrono |

> **📄 Nota**: Los detalles de infraestructura se documentarán en un **documento separado de infraestructura** (`019-v2-infrastructure-setup.md`).

### 5.2 APIs Externas — Detalles Técnicos

#### Google Veo 3 (Generación de Video)

| Parámetro | Valor |
|-----------|-------|
| **Proveedor** | Google (Vertex AI / Google AI Studio) |
| **Plan** | **Pro** (1000 puntos mensuales) |
| **Modelo** | `veo-3.0-generate-preview` (o versión estable disponible) |
| **Input** | Prompt de texto (max ~1500 tokens) generado a partir del guión narrativo |
| **Output** | Video MP4 |
| **Resolución** | Hasta 1080p |
| **Duración por generación** | 8 segundos de video por llamada API |
| **Coste por llamada** | 20 puntos por generación de 8 segundos |
| **Duración máxima por cápsula** | **40 segundos** (5 segmentos de 8 seg × 20 pts = 100 puntos por cápsula) |
| **Cuota mensual (plan Pro)** | **1000 puntos/mes** → **10 cápsulas de video/mes** (a 100 pts cada una) |
| **Tiempo de generación** | 3-8 minutos por cápsula de video |
| **Autenticación** | Google Cloud Service Account / API Key |
| **Latencia** | Asíncrona (se inicia operación y se consulta estado) |

**Estrategia de generación de video**: Cada cápsula de video consiste en hasta 5 segmentos de 8 segundos (máximo 40 segundos total). Los segmentos se generan con prompts derivados del guión narrativo y se concatenan con FFmpeg junto con el audio generado por ElevenLabs. El sistema controla que no se exceda el límite de 1000 puntos mensuales.

> **🔄 Abstracción de proveedor**: Veo 3 se integra detrás de la interfaz `IVideoGenerator` (ver RNF-V2-6). Si en el futuro se decide cambiar a otro proveedor (Runway, Kling, Minimax, etc.), solo se crea una nueva implementación del adapter sin modificar use cases, controllers ni base de datos. Costo estimado del swap: **~1 día de desarrollo**.

#### ElevenLabs (Text-to-Speech)

| Parámetro | Valor |
|-----------|-------|
| **Proveedor** | ElevenLabs |
| **Endpoint** | `https://api.elevenlabs.io/v1/text-to-speech/{voice_id}` |
| **Modelos disponibles** | `eleven_multilingual_v2`, `eleven_turbo_v2_5` |
| **Voces preconfiguradas** | **Pendiente de definir** según voces disponibles en plan Starter |
| **Formato output** | MP3 (128kbps o superior) |
| **Límite de caracteres por request** | ~5000 caracteres (se fragmenta si el guión es mayor) |
| **Latencia** | 2-10 segundos por fragmento (streaming disponible) |
| **Plan** | **Starter** (a confirmar límites exactos de caracteres y voces disponibles) |
| **Autenticación** | API Key (header `xi-api-key`) |

> **⚠️ Pendiente**: Revisar las voces disponibles y los límites de caracteres/mes del **plan Starter** de ElevenLabs para definir: (1) cuántas y cuáles voces ofrecer al usuario, (2) si el límite de caracteres del plan Starter es suficiente para el volumen esperado de generación. Si el plan Starter resulta insuficiente, evaluar upgrade a Creator.

**Estrategia para guiones largos**: Los guiones se dividen en bloques de ~4500 caracteres en puntos de pausa natural (fin de oración/párrafo), se generan en paralelo y se concatenan con FFmpeg.

> **🔄 Abstracción de proveedor**: ElevenLabs se integra detrás de la interfaz `IAudioGenerator` (ver RNF-V2-6). Si en el futuro se decide cambiar de proveedor TTS, solo se crea una nueva implementación del adapter.

### 5.3 Dependencias de v1 reutilizadas

| Componente v1 | Uso en v2 |
|:--------------|:----------|
| Fragmentos RAG (Pinecone) | Recuperar contenido textual de documentos para generar guiones |
| Gemini 2.5 Flash (Genkit) | Generar guión narrativo a partir de documentos |
| Sectores y RBAC | Controlar acceso a cápsulas por sector y rol |
| UserModel + RoleModel | Validar permisos de creación (admin/manager) y visualización (user) |
| KnowledgeSource model | Seleccionar documentos fuente para las cápsulas |
| API Client (frontend) | Comunicación frontend → backend |
| Auth (NextAuth + Auth0) | Autenticación y autorización |

---

## 6. Requisitos de Seguridad y Datos

### 6.1 Control de Acceso (RBAC)

| Acción | user | manager | admin |
|:-------|:----:|:-------:|:-----:|
| Ver/reproducir cápsulas de sectores asignados | ✅ | ✅ | ✅ |
| Descargar audio/video | ✅ | ✅ | ✅ |
| Crear cápsulas | ❌ | ✅ | ✅ |
| Editar guión | ❌ | ✅ | ✅ |
| Eliminar / archivar cápsulas | ❌ | ✅ | ✅ |
| Ver estadísticas de cuota | ❌ | ✅ | ✅ |
| Configurar voces TTS | ❌ | ❌ | ✅ |

**Nuevos permisos a agregar a la matriz RBAC**:

| Permiso | Descripción | user | manager | admin |
|---------|-------------|:----:|:-------:|:-----:|
| `capsule:read` | Ver y reproducir cápsulas | ✅ | ✅ | ✅ |
| `capsule:create` | Crear nuevas cápsulas | ❌ | ✅ | ✅ |
| `capsule:update` | Editar metadatos y guión de cápsulas | ❌ | ✅ | ✅ |
| `capsule:delete` | Eliminar/archivar cápsulas | ❌ | ✅ | ✅ |

### 6.2 Seguridad de Archivos Multimedia

- **URLs firmadas (Signed URLs)**: Los archivos multimedia NO se sirven con URLs públicas. Se generan URLs firmadas con expiración de **1 hora**
- **Validación de contenido**: Los archivos generados se validan antes de almacenarse (formato, integridad)
- **Sanitización del guión**: El texto del guión pasa por el sanitizador existente antes de enviarse a APIs externas (prevención de prompt injection)
- **Almacenamiento cifrado**: GCS cifra los archivos at-rest (AES-256) por defecto
- **Logs de auditoría**: Se registra quién creó, editó, eliminó o descargó cada cápsula

### 6.3 Protección de APIs Externas

- Las API keys de **ElevenLabs** y **Veo 3** se almacenan como secretos de entorno (`ELEVENLABS_API_KEY`, `GOOGLE_VEO_API_KEY`) — nunca expuestos al frontend
- Todas las llamadas a APIs externas se realizan exclusivamente desde el **backend**
- Rate limiting en el endpoint de creación: máximo **5 generaciones simultáneas** por organización
- Retry con backoff exponencial para fallos transitorios de APIs externas

### 6.4 Privacidad de Datos

- Los documentos fuente **nunca** se envían directamente a ElevenLabs o Veo 3. Solo se envía el **guión generado** (texto derivado)
- Los datos de los documentos fuente se procesan localmente y solo se envía el guión narrativo a los servicios de IA externos
- Los guiones generados pueden contener información sensible de la empresa → se aplica la misma política de privacidad que a los documentos RAG

---

## 7. Requisitos de Negocio (Business Requirements)

### RN-V2-1: Reducción de Costes de Producción de Material Formativo

**Objetivo**: Eliminar la necesidad de contratar profesionales de video/audio para crear material de onboarding.  
**Métrica**: Reducir el tiempo de creación de material multimedia de **~8 horas** (producción manual) a **~15 minutos** (generación asistida por IA).

### RN-V2-2: Mejora de la Tasa de Retención de Conocimiento

**Objetivo**: Los empleados que consumen contenido en formato video/audio retienen hasta un **65% más** de información que quienes solo leen documentos.  
**Métrica**: Incremento del ~20% en la tasa de consultas respondidas correctamente en el primer intento (medible a través del chat RAG).

### RN-V2-3: Control de Costes de APIs Externas

**Objetivo**: Mantener el coste mensual de generación multimedia dentro de un presupuesto predecible.  
**Métrica**: 
- Veo 3: **10 cápsulas de video/mes** (plan Pro: 1000 puntos/mes, 100 puntos por cápsula de 40 seg)
- ElevenLabs: Según plan Starter (a confirmar límites exactos de caracteres/mes)
- Gemini 2.5 Flash: Incluido en el plan existente de Google AI

### RN-V2-4: Time-to-Market

**Objetivo**: Implementar el módulo completo en **3-4 semanas** de desarrollo.  
**Justificación**: Reutilización significativa de infraestructura v1 (auth, RBAC, sectores, RAG, frontend layout).

---

## 8. Requisitos de Usuario (User Requirements)

### 8.1 Perfil: Admin / Manager (Creador de Contenido)

#### RU-V2-1: Creación Asistida por IA

> "Como manager, quiero poder seleccionar documentos de un sector y que la IA me genere automáticamente un guión, un audio y un video explicativo, para no tener que producir material multimedia manualmente."

**Criterios de Aceptación del usuario**:
- Puedo seleccionar el sector y los documentos fuente
- Puedo escribir un texto introductorio opcional
- La IA me genera un guión que puedo editar antes de continuar
- Al hacer clic en "Generar Cápsula", el sistema crea el audio y el video
- Puedo ver el progreso de la generación en tiempo real
- Puedo descargar el resultado final

#### RU-V2-2: Gestión de Cápsulas Existentes

> "Como admin, quiero ver una lista de todas las cápsulas generadas, filtrar por sector, y poder archivar las que ya no son relevantes."

**Criterios de Aceptación del usuario**:
- Veo un listado con todas las cápsulas del sistema (si soy admin) o de mis sectores (si soy manager)
- Puedo filtrar por sector, estado (activa/archivada), y tipo (video/audio)
- Puedo buscar por título
- Puedo archivar o eliminar cápsulas obsoletas

#### RU-V2-3: Control de Cuota

> "Como admin, quiero saber cuántas generaciones de video me quedan en el mes, para planificar la creación de contenido."

**Criterios de Aceptación del usuario**:
- Veo un indicador de cuota: "X de 10 videos utilizados este mes"
- Si se alcanza el límite, el botón de generar video se desactiva con un mensaje explicativo
- Puedo seguir generando cápsulas de solo audio sin afectar la cuota de video

### 8.2 Perfil: User / Employee (Consumidor de Contenido)

#### RU-V2-4: Reproducción de Cápsulas

> "Como empleado nuevo, quiero acceder a videos y audios explicativos de mi sector para complementar mi aprendizaje autoguiado."

**Criterios de Aceptación del usuario**:
- Veo una sección "Cápsulas" en el menú lateral
- Solo veo cápsulas de los sectores a los que tengo acceso
- Puedo reproducir videos y audios directamente en el navegador
- Puedo descargar el contenido para verlo offline
- Veo el título, descripción, sector y duración de cada cápsula

#### RU-V2-5: Navegación por Sectores

> "Como empleado con acceso a múltiples sectores, quiero filtrar las cápsulas por sector para encontrar rápidamente el contenido relevante."

**Criterios de Aceptación del usuario**:
- Puedo filtrar la lista de cápsulas por sector
- Puedo buscar por título o descripción
- Las cápsulas más recientes aparecen primero
- Se muestra a qué sector pertenece cada cápsula

---

## 9. Modelo de Datos (Nuevas Entidades)

### 9.1 Tabla: `capsules`

**Responsabilidad**: Almacenar la metadata de cada cápsula multimedia generada.

| Columna | Tipo | Nullable | Default | Descripción |
|---------|------|----------|---------|-------------|
| `id` | UUID | NO | auto | Primary key |
| `title` | VARCHAR(255) | NO | — | Título de la cápsula |
| `description` | TEXT | SÍ | null | Descripción breve de la cápsula (auto-generada como resumen del guión, o escrita manualmente) |
| `sector_id` | UUID | NO | — | FK → sectors.id |
| `type` | ENUM('video', 'audio', 'both') | NO | 'video' | Tipo de cápsula: video, audio o ambos |
| `status` | ENUM | NO | 'draft' | Estado actual (ver estados arriba) |
| `intro_text` | TEXT | SÍ | null | Texto introductorio del creador |
| `script` | TEXT | SÍ | null | Guión narrativo (generado o manual) |
| `audio_url` | VARCHAR(512) | SÍ | null | Path en GCS del audio generado |
| `video_url` | VARCHAR(512) | SÍ | null | Path en GCS del video generado |
| `thumbnail_url` | VARCHAR(512) | SÍ | null | Path en GCS del thumbnail (ver `021-v2-thumbnail-strategy.md`) |
| `duration_seconds` | INTEGER | SÍ | null | Duración en segundos del contenido |
| `audio_voice_id` | VARCHAR(100) | SÍ | null | ID de la voz utilizada (ElevenLabs) |
| `generation_metadata` | JSONB | SÍ | null | Datos del pipeline (tokens, costes, tiempos) |
| `created_by` | UUID | NO | — | FK → users.id (quien creó la cápsula) |
| `published_at` | TIMESTAMPTZ | SÍ | null | Fecha de publicación (estado → ACTIVE) |
| `created_at` | TIMESTAMPTZ | NO | NOW() | Fecha de creación |
| `updated_at` | TIMESTAMPTZ | NO | NOW() | Última actualización |

**Índices**:
- `sector_id` (filtros por sector)
- `status` (filtros por estado)
- `created_by` (filtros por creador)
- `type` (filtros por tipo)
- Compuesto: `(sector_id, status)` para consultas frecuentes

**Relaciones**:
- Many-to-One con `sectors` (una cápsula pertenece a un sector)
- Many-to-One con `users` (creador de la cápsula)
- Many-to-Many con `knowledge_sources` (documentos fuente usados)

### 9.2 Tabla: `capsule_sources` (Join Table)

**Responsabilidad**: Relacionar cápsulas con los documentos fuente utilizados.

| Columna | Tipo | Nullable | Default | Descripción |
|---------|------|----------|---------|-------------|
| `capsule_id` | UUID | NO | — | FK → capsules.id |
| `source_id` | UUID | NO | — | FK → knowledge_sources.id |

**Primary Key**: Compuesta (`capsule_id`, `source_id`)

### 9.3 Tabla: `capsule_generation_logs`

**Responsabilidad**: Log detallado de cada intento de generación para auditoría y debugging.

| Columna | Tipo | Nullable | Default | Descripción |
|---------|------|----------|---------|-------------|
| `id` | UUID | NO | auto | Primary key |
| `capsule_id` | UUID | NO | — | FK → capsules.id |
| `step` | ENUM | NO | — | 'script', 'audio', 'video', 'postprocess' |
| `status` | ENUM | NO | — | 'started', 'completed', 'failed' |
| `started_at` | TIMESTAMPTZ | NO | NOW() | Inicio de la etapa |
| `completed_at` | TIMESTAMPTZ | SÍ | null | Fin de la etapa |
| `duration_ms` | INTEGER | SÍ | null | Duración en milisegundos |
| `error_message` | TEXT | SÍ | null | Mensaje de error si falló |
| `metadata` | JSONB | SÍ | null | Datos adicionales (tokens, costes, IDs externos) |

---

## 10. Diagrama de Arquitectura del Pipeline

```
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                          PIPELINE DE GENERACIÓN DE CÁPSULA                                 │
├───────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                           │
│  ┌─────────────┐    ┌──────────────────┐    ┌────────────────┐    ┌──────────────────┐   │
│  │  FRONTEND   │    │    NestJS API     │    │  INTERFACES    │    │ IMPLEMENTACIONES │   │
│  │  (Next.js)  │    │   CapsuleModule   │    │  (Dominio)     │    │ (Infraestructura)│   │
│  └──────┬──────┘    └────────┬─────────┘    └───────┬────────┘    └────────┬─────────┘   │
│         │                    │                       │                      │              │
│    1. Seleccionar       2. Crear cápsula             │                      │              │
│       docs + intro          (DRAFT)                  │                      │              │
│         │                    │                       │                      │              │
│         │── POST ───────────▶│                       │                      │              │
│         │                    │── IVectorStore ──────▶│──── Pinecone ───────▶│              │
│         │                    │   (fragmentos RAG)    │                      │              │
│         │                    │◀── Contexto ─────────│◀─────────────────────│              │
│         │                    │                       │                      │              │
│         │                    │── Genkit (Gemini) ───▶│  3. Generar guión    │              │
│         │◀── Guión ─────────│◀── Guión narrativo ──│                      │              │
│         │                    │                       │                      │              │
│    4. Editar guión      5. Confirmar                 │                      │              │
│       (opcional)            generación               │                      │              │
│         │── PATCH ──────────▶│                       │                      │              │
│         │── POST generate ──▶│                       │                      │              │
│         │                    │                       │                      │              │
│         │                    │── IAudioGenerator ───▶│── ElevenLabsAdapter ▶│              │
│         │                    │   (TTS del guión)     │   (o futuro swap)    │              │
│         │                    │◀── Audio MP3 ────────│◀─────────────────────│              │
│         │                    │                       │                      │              │
│         │                    │── IVideoGenerator ───▶│── Veo3Adapter ──────▶│              │
│         │                    │   (5 seg × 8 seg)     │   (o RunwayAdapter)  │              │
│         │                    │◀── Video MP4 ────────│◀─────────────────────│              │
│         │                    │                       │                      │              │
│         │                    │── IMediaProcessor ───▶│── FFmpegAdapter ────▶│              │
│         │                    │   (merge + concat)    │                      │              │
│         │                    │◀── Video final ──────│◀─────────────────────│              │
│         │                    │                       │                      │              │
│         │                    │── IMediaStorage ─────▶│── GcsStorageAdapter ▶│              │
│         │                    │   (upload)            │                      │              │
│         │                    │◀── URLs firmadas ────│◀─────────────────────│              │
│         │                    │                       │                      │              │
│         │◀── Polling ────────│   6. Cápsula COMPLETED│                      │              │
│         │    status: ready   │      → publicar       │                      │              │
│         │                    │                       │                      │              │
│    7. Reproducir /      8. Servir contenido          │                      │              │
│       Descargar              multimedia              │                      │              │
│         │                    │                       │                      │              │
└─────────┴────────────────────┴───────────────────────┴──────────────────────┴──────────────┘

Nota: Los use cases SOLO dependen de las interfaces del dominio (columna 3).
Las implementaciones concretas (columna 4) se inyectan vía NestJS DI.
Para cambiar de proveedor, se cambia la implementación sin tocar use cases ni controllers.
```

---

## 11. Endpoints API Propuestos

### 11.1 Cápsulas (CapsuleController)

| Método | Endpoint | Descripción | Roles |
|--------|----------|-------------|-------|
| `POST` | `/api/v1/capsules` | Crear nueva cápsula (draft) | admin, manager |
| `GET` | `/api/v1/capsules` | Listar cápsulas (paginado, filtro por sector, estado, tipo) | admin, manager, user |
| `GET` | `/api/v1/capsules/:id` | Detalle de una cápsula | admin, manager, user |
| `PATCH` | `/api/v1/capsules/:id` | Actualizar guión/metadatos | admin, manager |
| `DELETE` | `/api/v1/capsules/:id` | Eliminar/archivar cápsula | admin, manager |
| `POST` | `/api/v1/capsules/:id/generate-script` | Generar guión desde documentos fuente | admin, manager |
| `POST` | `/api/v1/capsules/:id/generate` | Iniciar pipeline completo (audio + video) | admin, manager |
| `GET` | `/api/v1/capsules/:id/status` | Consultar estado de generación (polling) | admin, manager |
| `GET` | `/api/v1/capsules/:id/download/:type` | Obtener URL firmada de descarga (type: audio\|video) | admin, manager, user |
| `POST` | `/api/v1/capsules/:id/publish` | Publicar cápsula (COMPLETED → ACTIVE) | admin, manager |
| `POST` | `/api/v1/capsules/:id/archive` | Archivar cápsula | admin, manager |
| `GET` | `/api/v1/capsules/quota` | Consultar cuota de generación del mes | admin, manager |

### 11.2 Paginación y Filtros (GET `/api/v1/capsules`)

**Query Parameters**:

| Parámetro | Tipo | Requerido | Default | Descripción |
|-----------|------|:---------:|---------|-------------|
| `page` | number | No | 1 | Número de página |
| `limit` | number | No | 10 | Resultados por página (máx: 50) |
| `sectorId` | UUID | No | — | Filtrar por sector |
| `status` | string | No | — | Filtrar por estado: `DRAFT`, `GENERATING`, `COMPLETED`, `ACTIVE`, `FAILED`, `ARCHIVED` |
| `type` | string | No | — | Filtrar por tipo: `video`, `audio`, `both` |
| `search` | string | No | — | Búsqueda por título (case-insensitive, partial match) |
| `sortBy` | string | No | `createdAt` | Campo de ordenamiento: `createdAt`, `title`, `status` |
| `sortOrder` | string | No | `DESC` | Dirección: `ASC` o `DESC` |

**Respuesta paginada** (estándar del proyecto):
```json
{
  "data": [ /* array de cápsulas */ ],
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 42,
    "totalPages": 5
  }
}
```

**Reglas de visibilidad**:
- **admin**: Ve todas las cápsulas del sistema
- **manager**: Ve cápsulas de los sectores a los que tiene acceso
- **user**: Ve solo cápsulas con estado `ACTIVE` de los sectores a los que tiene acceso

---

## 12. Rutas Frontend Propuestas

| Ruta | Componente | Descripción | Roles |
|------|-----------|-------------|-------|
| `/{locale}/capsules` | `CapsuleListView` | Lista de cápsulas (vista usuario: solo reproducir) | user, manager, admin |
| `/{locale}/capsules/:id` | `CapsulePlayerView` | Reproductor de cápsula individual | user, manager, admin |
| `/{locale}/capsules/create` | `CapsuleCreateWizard` | Wizard 2 pasos: Step 1 (selección sector/docs) → Step 2 (formulario split-panel + preview) | manager, admin |
| `/{locale}/capsules/:id/edit` | `CapsuleEditView` | Edición de guión/metadatos antes de generar | manager, admin |

**Sidebar**: Se añadirá un nuevo ítem "Capsules" / "Cápsulas" bajo la sección "Platform" (visible para todos los roles con `capsule:read`).

---

## 13. Estimación de Costes Operativos Mensuales

| Servicio | Uso estimado | Coste estimado |
|----------|-------------|---------------|
| **Google AI Pro plan** (Veo 3 + Gemini) | Plan Pro: 1000 pts/mes Veo 3 + Gemini incluido | ~$50/mes (precio plan Pro) |
| **ElevenLabs** (TTS) | Según plan Starter (a confirmar caracteres/mes) | ~$5-11/mes (plan Starter) |
| **Google Cloud Storage** | ~5-20 GB/mes (10 videos de 40 seg + audios) | ~$0.25-1.00/mes |
| **FFmpeg** (post-procesamiento) | Procesamiento en servidor | Sin coste adicional (CPU/RAM del server) |
| **Total estimado** | — | **~$55-62/mes** |

> **Nota**: Los costes son estimaciones basadas en precios publicados. Los costes reales dependerán del plan contratado y el uso efectivo.

---

## 14. Riesgos y Mitigaciones

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|:------------:|:-------:|------------|
| Cuota de Veo 3 agotada antes de fin de mes (10 videos/mes) | Alta | Medio | Contador visible en UI + alertas al llegar a 80% (8 de 10). Alternativa: generar cápsulas de solo audio |
| Tiempo de generación de video excesivo (> 10 min) | Media | Medio | Procesamiento asíncrono + polling de estado. UX clara de espera |
| Calidad del guión generado insuficiente | Media | Medio | Guión siempre editable antes de generar. Mejora iterativa de prompts |
| Latencia de ElevenLabs API | Baja | Bajo | Streaming + fragmentación del texto + retry con backoff |
| Cambios en APIs externas (breaking changes) o necesidad de cambiar proveedor | Baja | Alto | Capa de abstracción (adapter pattern): `IVideoGenerator`, `IAudioGenerator`, `IMediaStorage`. Cada API detrás de una interfaz de dominio. Costo de swap de proveedor: **~1 día** (crear nuevo adapter + tests, cambiar 1 línea de binding en el módulo NestJS) |
| Costes exceden el presupuesto | Media | Alto | Límites hard en backend. Dashboard de consumo. Alertas de gasto |
| Contenido generado inapropiado o inexacto | Baja | Alto | Review humano obligatorio (status COMPLETED antes de ACTIVE). Guión editable |
| GCS no accesible o URLs firmadas expiran | Baja | Medio | Regeneración automática de URLs. Fallback con retry |

---

## 15. Plan de Implementación Propuesto

> La implementación se divide en dos bloques principales: primero se completa el flujo de **audio** (menor complejidad, sin dependencia de Veo 3 ni FFmpeg), y luego se añade el flujo de **video** (mayor complejidad, procesamiento asíncrono, cuota limitada). Esto permite entregar valor funcional de forma incremental.

---

### 🔊 BLOQUE A — Cápsulas de Audio (Fases 1-4)

### Fase 1: Backend — Modelo de datos y CRUD (3-4 días)
- **Nueva migración TypeORM** para las tablas `capsules`, `capsule_sources`, `capsule_generation_logs` (migración independiente, no se modifican migraciones existentes)
- Seed de nuevos permisos RBAC: `capsule:read`, `capsule:create`, `capsule:update`, `capsule:delete` (actualización del `RbacSeederService`)
- Entidades de dominio, DTOs, y repositorios
- `CapsuleService` y `CapsuleController` (operaciones CRUD básicas)
- Actualización de `context-ai-shared` con enums (`CapsuleType`, `CapsuleStatus`, `CapsuleGenerationStep`) y DTOs de cápsulas
- Tests unitarios + integración

### Fase 2: Backend — Pipeline de Audio (3-4 días)
- Servicio de generación de guión narrativo (Gemini 2.5 Flash + fragmentos RAG de Pinecone)
- Implementación de `IAudioGenerator` para ElevenLabs (`ElevenLabsAudioService`):
  - `generateAudio()`: Convierte texto a audio via ElevenLabs API, fragmenta textos largos y concatena
  - `getAvailableVoices()`: Retorna voces disponibles del plan Starter
  - Registrado en `CapsuleModule` como: `{ provide: 'IAudioGenerator', useClass: ElevenLabsAudioService }`
- Implementación de `IMediaStorage` para GCS (`GcsStorageService`):
  - `upload()`: Sube archivos a Google Cloud Storage
  - `getSignedUrl()`: Genera URLs firmadas temporales para reproducción/descarga
  - `delete()`: Elimina archivos al archivar cápsulas
  - Registrado en `CapsuleModule` como: `{ provide: 'IMediaStorage', useClass: GcsStorageService }`
- Pipeline de audio: guión → audio (`IAudioGenerator`) → almacenamiento (`IMediaStorage`) → actualización de estado
- Endpoint `POST /capsules/:id/generate-script` (generación de guión)
- Endpoint `POST /capsules/:id/generate` (ejecutar pipeline de audio)
- Endpoint `GET /capsules/:id/download/audio` (URL firmada de descarga)
- Tests unitarios de adapters + tests de integración del pipeline

### Fase 3: Frontend — Wizard de Creación para Audio (3-4 días)
- Componente `CapsuleCreateWizard` con stepper de 2 pasos
- **Step 1**: `CapsuleContentSelector` — Selector de tipo (🔊 Audio por defecto), sector, título, y documentos fuente con búsqueda
- **Step 2**: `CapsuleFormPanel` (split-panel) — Texto introductorio, guión narrativo editable, selector de voz ElevenLabs + `CapsulePreviewPanel` — Reproductor de audio y descarga
- Botón "Generar guión con IA" + indicador de progreso
- Botón "Generar Audio" → polling al endpoint `/capsules/:id/status`
- Reproductor de audio HTML5 (`<audio>`) en panel de vista previa
- Zustand store para estado del wizard
- i18n (es/en)

### Fase 4: Frontend — Listado, Reproducción y Gestión de Audio (2-3 días)
- `CapsuleListView` (listado paginado con filtros por sector/estado/tipo)
- `CapsulePlayerView` (reproductor de audio integrado con metadatos)
- Sidebar: nuevo ítem "Cápsulas" (visible para todos con `capsule:read`)
- Descarga de audio
- Tests unitarios + E2E con Playwright (flujo completo de audio)

#### ✅ Entregable Bloque A: Cápsulas de audio funcionales end-to-end
- Admin/Manager puede crear cápsulas de tipo Audio (seleccionar docs → generar guión → generar audio → publicar)
- User puede listar, reproducir y descargar cápsulas de audio de sus sectores
- **Tiempo estimado Bloque A: 11-15 días**

---

### 🎬 BLOQUE B — Cápsulas de Video (Fases 5-7)

### Fase 5: Backend — Pipeline de Video (4-5 días)
- Implementación de `IVideoGenerator` para Veo 3 (`Veo3VideoGeneratorService`):
  - `generateSegment()`: Envía prompt a Veo 3 API, retorna `operationId` para polling
  - `getSegmentStatus()`: Consulta el estado de una operación de generación en Veo 3
  - `getQuotaInfo()`: Calcula puntos consumidos/restantes del plan Pro (1000 pts/mes)
  - Registrado en `CapsuleModule` como: `{ provide: 'IVideoGenerator', useClass: Veo3VideoGeneratorService }`
  - **Nota**: Para cambiar a otro proveedor (ej. Runway), solo se crea `RunwayVideoGeneratorService` implementando la misma interfaz `IVideoGenerator` y se cambia el binding en el módulo (~1 día de trabajo)
- Servicio de post-procesamiento (`FFmpegAdapter` implementando `IMediaProcessor`):
  - Merge de audio + video, concatenación de segmentos de 8 seg
- Pipeline de video: guión → audio (reutilizar) → video (5 segmentos × 8 seg) → merge FFmpeg → almacenamiento
- Sistema de cuota mensual (1000 pts/mes, 100 pts por cápsula → máx 10 cápsulas de video/mes)
- Endpoint `GET /capsules/quota` (consultar cuota restante)
- Job asíncrono (BullMQ + Redis) para el pipeline de video (larga duración)
- Manejo de fallos parciales: reintento desde la etapa fallida sin regenerar etapas previas
- Tests unitarios del adapter `Veo3VideoGeneratorService` + tests de integración del pipeline

### Fase 6: Frontend — Extensión del Wizard para Video (2-3 días)
- Ampliar Step 1 con soporte para tipo 🎬 Video y 🎬🔊 Ambos
- Indicador de cuota de videos restantes (visible al seleccionar Video o Ambos)
- Ampliar Step 2: botones "Generar Video" / "Generar Video y Audio" según tipo
- Indicador de progreso multi-etapa (Audio... → Video... → Merge... → Listo)
- Reproductor de video HTML5 (`<video>`) en panel de vista previa
- Botones de descarga adaptados: video (tipo Video), video + audio (tipo Ambos)
- Polling de estado para generación de video (operación de larga duración: 3-8 min)
- Ampliar `CapsulePlayerView` con reproductor de video
- Ampliar `CapsuleListView` con indicador de tipo (video/audio/both) y estados de generación
- Tests unitarios

### Fase 7: Integración, Testing y Deployment (2-3 días)
- Tests E2E completos (flujo audio + flujo video + flujo ambos)
- Test coverage
- Validación de seguridad (URLs firmadas, RBAC, cuota)
- Validación de manejo de fallos parciales (reintento por etapa)
- Optimización de rendimiento (polling, carga de listado)
- Documentación API (Swagger)
- Deployment + validación en staging
- Agregar variables de entorno nuevas creadas durante este bloque en el workflow de github (deploy-production.yml)

#### ✅ Entregable Bloque B: Cápsulas de video funcionales end-to-end
- Admin/Manager puede crear cápsulas de tipo Video (Sacar el ambos) (guión → audio → video → merge → publicar)
- Cuota mensual visible y controlada (10 videos/mes)
- Pipeline asíncrono con polling de estado
- **Tiempo estimado Bloque B: 8-11 días**

---

**Tiempo total estimado: 19-26 días de desarrollo**

| Bloque | Fases | Estimación | Entregable |
|--------|-------|:----------:|------------|
| 🔊 **A — Audio** | Fases 1-4 | 11-15 días | Cápsulas de audio funcionales (guión + TTS + reproducción) |
| 🎬 **B — Video** | Fases 5-7 | 8-11 días | Cápsulas de video funcionales (Veo 3 + FFmpeg + cuota) |

---

## 16. Glosario

| Término | Definición |
|---------|-----------|
| **Cápsula** | Unidad de contenido multimedia (video y/o audio) generada por IA a partir de documentos de conocimiento |
| **Guión narrativo** | Texto estructurado que sirve como guion para la narración del audio/video |
| **TTS (Text-to-Speech)** | Tecnología que convierte texto escrito en voz sintetizada |
| **Veo 3** | Modelo de Google para generación de video a partir de texto (guión narrativo) |
| **ElevenLabs** | Plataforma de síntesis de voz con voces ultra-realistas |
| **Pipeline de generación** | Secuencia de pasos automatizados: guión → audio → video → post-procesamiento → almacenamiento |
| **URL firmada (Signed URL)** | URL temporal y autenticada para acceder a archivos privados en Cloud Storage |
| **FFmpeg** | Herramienta de procesamiento multimedia para conversión, merge y edición de audio/video |
| **Cuota mensual** | Límite de 10 cápsulas de video por mes (plan Pro: 1000 puntos/mes, 100 puntos por cápsula de 40 seg) |

---

## 17. Referencias

| Documento | Relación |
|-----------|----------|
| `001-requirements.md` | RF-3 (Generación de Cápsulas Multimedia) — requisito original |
| `002-use-cases.md` | UC3 — caso de uso base |
| `006-data-model.md` | Modelo de datos existente que se extiende |
| `016-rbac-permissions-matrix.md` | Matriz RBAC que se amplía con permisos `capsule:*` |
| `005-technical-architecture.md` | Arquitectura base (Clean Architecture + NestJS + Next.js) |
| `008-roadmap.md` | Sección 11 "Post-MVP" — planificación futura |
| `019-v2-infrastructure-setup.md` | 📄 *Por crear* — Configuración de infraestructura V2 (BullMQ, Redis, GCS, FFmpeg, CI/CD) |
| `020-v2-prompt-contracts.md` | 📄 *Por crear* — Contratos de prompts para generación de guión narrativo |
| `021-v2-thumbnail-strategy.md` | 📄 *Por crear* — Estrategia de generación y almacenamiento de thumbnails |

---

> **Nota final**: Este documento cubre el análisis completo de requisitos para la funcionalidad de Generación de Cápsulas Multimedia. La implementación se divide en dos bloques incrementales: **Bloque A (Audio)** entrega valor funcional rápidamente con cápsulas de audio (guión IA + ElevenLabs TTS), sin dependencias de Veo 3, FFmpeg ni procesamiento asíncrono pesado. **Bloque B (Video)** extiende la funcionalidad con generación de video (Veo 3 + FFmpeg + BullMQ), cuota mensual y pipeline asíncrono. Esta separación permite validar el flujo completo con audio antes de invertir en la integración más compleja de video.

