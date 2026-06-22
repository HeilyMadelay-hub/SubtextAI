# SubtextAI

### Motor de telemetría de comunicación humana — análisis pragmático auditable y gobernado

> **SubtextAI convierte conversaciones ambiguas en una visualización auditada de intención, emoción y subtexto**, combinando RAG con *grounding* obligatorio y gobernanza de agentes sobre Azure.
>
> No se limita a responder: **expone cómo y por qué interpreta cada mensaje**, con trazabilidad completa por `trace_id` y un **Modo de Reproducción** que funciona como un *"replay" de decisiones humanas* —recorres la conversación y ves evolucionar la emoción, la intención y la tensión a lo largo del tiempo.

`Azure` · `ASP.NET Core` · `React + Vite` · `Azure OpenAI (GPT-4o-mini)` · `Azure AI Search (RAG híbrido)` · `Azure Monitor`

---

## El problema que resuelve

La mayoría de los malentendidos no están en lo que se dice, sino en lo que se quiere decir. SubtextAI interpreta esa capa implícita —intención, emoción y subtexto— en contextos reales de **pareja, trabajo, entorno social y negociación**.

La diferencia frente a un asistente convencional es que **no es una caja negra**: cada interpretación se fundamenta en fuentes documentales reales, se calcula con un nivel de confianza objetivo y queda completamente auditada. No es un chatbot académico, sino una herramienta de interpretación del comportamiento conversacional.

---

## Posicionamiento: Human Telemetry Engine

SubtextAI puede entenderse como un **motor de telemetría aplicado a la conversación humana**. Igual que la telemetría de un sistema mecánico registra y formaliza señales continuas —velocidad, aceleración, fricción— para describir su comportamiento, SubtextAI formaliza la dinámica de un diálogo a partir de las señales que **ya calcula y persiste por mensaje** (intensidad emocional, cambios de intención, relación entre lo explícito y lo implícito, tensión).

Estas métricas **no son una capa especulativa ni un modelo nuevo**: son una formalización de datos que ya existen en las trazas. Se derivan directamente de las trazas persistidas y heredan, sin excepción, el mismo *grounding* y la misma trazabilidad por `trace_id` que el resto del sistema.

| Métrica | Definición | Cómo se deriva |
|---------|------------|----------------|
| **Velocidad emocional** | Variación de la intensidad emocional entre dos mensajes consecutivos (primera derivada de la intensidad sobre la secuencia de mensajes). | A partir de la intensidad ya anotada por mensaje. |
| **Aceleración de conflicto** | Variación de la velocidad emocional (segunda derivada). Indica si una escalada se acelera o se frena. | A partir de la serie de velocidad emocional. |
| **Fricción conversacional** | Tensión acumulada entre el significado explícito y el implícito a lo largo del diálogo. Mide la divergencia sostenida entre lo que se dice y lo que se infiere. | A partir del subtexto y la tensión ya registrados. |
| **Curvas de intención** | Trayectoria de los cambios de intención (p. ej. `neutral → defensiva → agresiva`) a lo largo de la conversación. | A partir de la intención detectada por mensaje. |

La representación visual de estas métricas es precisamente lo que ofrece el Modo de Reproducción: la línea temporal, el medidor de tensión y el *heatmap* son la lectura directa de la telemetría, no un cálculo adicional.

---

## Qué hace diferente a SubtextAI

| Capacidad | Qué aporta |
|-----------|------------|
| **Grounding obligatorio** | Ninguna respuesta se entrega sin evidencia documental. Si la confianza cae bajo el umbral, la generación se bloquea antes de invocar al modelo. |
| **Trazabilidad real por `trace_id`** | Cada respuesta es reconstruible: qué documentos influyeron, con qué score, qué prompt y versión, qué modelo y qué políticas se evaluaron. |
| **Gobernanza de agentes en código** | Las reglas de seguridad viven en el pipeline, no en el prompt: el modelo nunca se ejecuta si se viola una política crítica. |
| **Telemetría conversacional** | Formaliza la dinámica del diálogo (velocidad emocional, aceleración de conflicto, fricción, curvas de intención) sobre datos ya trazados. |
| **Telemetría en vivo de la sesión** | La misma telemetría, leída mensaje a mensaje mientras la conversación ocurre, sin esperar al cierre de la sesión. |
| **Evaluación continua automatizada** | Un job periódico ejecuta pruebas en producción y publica métricas en Azure Monitor, detectando desviaciones sin intervención manual. |
| **Modo de Reproducción visual** | Convierte cualquier conversación analizada en una línea temporal interactiva de emoción, intención y tensión. |

El corpus documental es **intercambiable y configurable**: la arquitectura de recuperación no está acoplada a ninguna fuente, así que cualquier colección de documentos compatible puede indexarse sin tocar la lógica de gobernanza.

---

## Modo de Reproducción de Conversaciones

> *"Este sistema no analiza texto. Visualiza decisiones humanas."*

El **Modo de Reproducción** es la cara más distintiva del producto. Transforma una conversación ya analizada en una visualización interactiva basada en línea temporal, en la línea de un *replay* de videojuego o una retransmisión deportiva: el usuario recorre la conversación paso a paso y observa cómo evolucionan el significado, la emoción y la intención.

Todo lo que muestra se reconstruye a partir de las trazas ya persistidas, así que la reproducción mantiene el mismo *grounding* y la misma trazabilidad que el resto del sistema: **no inventa interpretaciones, las visualiza**. La línea temporal y el medidor de tensión son, además, la representación directa de las métricas de telemetría conversacional descritas arriba.

### Qué muestra

La conversación se representa como una secuencia cronológica de **nodos**, uno por mensaje. Cada nodo expone la anotación pragmática que el sistema generó para ese mensaje:

- **Intensidad emocional** — baja / media / alta.
- **Cambio de intención** — la transición detectada respecto al mensaje previo (p. ej. `neutral → defensiva → agresiva`).
- **Subtexto** — significado explícito frente a significado implícito inferido.
- **Nivel de confianza** — derivado del score RAG y del análisis del modelo (ALTA / MEDIA / BAJA).

Sobre la línea temporal se resaltan los **momentos críticos**, los puntos de inflexión de la conversación:

| Momento crítico | Qué señala |
|-----------------|------------|
| **Primera ambigüedad** | Primer mensaje con intención incierta o significado ambiguo |
| **Primera escalada emocional** | Primer incremento significativo de intensidad |
| **Primera contradicción** | Primera tensión implícita entre lo explícito y lo implícito |
| **Desencadenante del conflicto** | Punto de quiebre hacia el conflicto o la alta tensión |

### Feedback visual

El estado emocional de cada nodo se comunica con una codificación de color coherente, acompañada de un **medidor de tensión** que evoluciona de forma continua a lo largo del diálogo:

| Color | Estado | Significado |
|-------|--------|-------------|
| 🟢 Verde | Neutral / estable | Comunicación sin tensión detectable |
| 🟡 Amarillo | Ambigüedad / intención incierta | Significado abierto a interpretación |
| 🟠 Naranja | Cambio emocional detectado | Variación relevante de intensidad o intención |
| 🔴 Rojo | Conflicto / alta tensión | Escalada o quiebre de la conversación |

### Heatmap de tensión temporal

Sobre la línea temporal se superpone una **franja de calor** que codifica por color la evolución de la tensión a lo largo de toda la conversación. Permite leer de un vistazo dónde se concentran los tramos estables y dónde se produce la escalada, sin necesidad de recorrer mensaje a mensaje.

La franja reutiliza la misma codificación de color ya definida (verde → amarillo → naranja → rojo) y es una **visualización pura sobre datos existentes**: representa la tensión por mensaje que ya registran las trazas, sin recalcular ni inferir nada nuevo. Funciona, en la práctica, como la lectura visual de la curva de fricción conversacional.

### Explicación de IA

Para cada momento crítico, el sistema responde a una pregunta concreta —**¿por qué se detectó un cambio en este punto?**— con una explicación breve fundamentada en tres fuentes: los **documentos recuperados** vía RAG (cuando estén disponibles), la **salida del razonamiento** del modelo y el **contexto completo** de la conversación hasta ese punto. Cada inflexión señalada es, por tanto, explicable y rastreable hasta su evidencia.

### Demo Mode (autoplay)

Modo de demostración pensado para comprender el sistema **sin lectura técnica previa**. La pantalla ofrece una única acción: **Reproducir conversación**. Incluye un conjunto de **3-4 conversaciones pregrabadas** representativas de los contextos del sistema (pareja, trabajo, negociación). Al iniciarse:

- la línea temporal avanza de forma automática (*autoplay*), como una narrativa visual continua;
- la vista aplica **zoom automático** sobre cada momento crítico a medida que se alcanza;
- la codificación de color, el medidor de tensión y el *heatmap* se actualizan al ritmo del avance.

Es una **capa de frontend (React) que opera sobre trazas ya persistidas**: durante la reproducción no se realiza ninguna llamada nueva al modelo. Las conversaciones pregrabadas son trazas reales previamente analizadas y almacenadas, por lo que el modo es completamente coherente con la auditoría disponible en el resto del sistema.

### Experiencia e interacción

Además del modo automático, el usuario puede inspeccionar la conversación de forma manual:

- **Animación de tensión** — la conversación "respira": la intensidad se traduce en animaciones progresivas y, en los conflictos, micro-impactos (*shake* sutil), zoom contextual y ralentización deliberada para enfatizar el momento.
- **Vista comparativa (antes / después)** — panel izquierdo con el mensaje original, panel derecho con la interpretación de la IA (intención, emoción, subtexto), haciendo visible la diferencia entre lo que se dice y lo que se quiere decir.
- **Insight Cards** — al pulsar un nodo se despliega su lectura completa (intención, estado emocional, subtexto, confianza y resumen del razonamiento), con un diseño minimalista de baja carga cognitiva.

La navegación es libre y dinámica: avanzar de forma secuencial o saltar a cualquier mensaje o momento crítico, **sin recargar la página**. El modo se alimenta del endpoint `GET /replay/{trace_id}` y, al reconstruirse desde trazas ya persistidas (sin nuevas llamadas al modelo durante la reproducción), **carga en menos de 2 segundos** para conversaciones típicas.

### Telemetría en vivo de la sesión

Mientras el Modo de Reproducción opera siempre **después del hecho**, sobre una conversación ya cerrada y persistida, el sistema ofrece también una lectura en vivo de la sesión que está ocurriendo. A medida que cada mensaje pasa por el pipeline de `POST /analizar`, el frontend acumula las métricas ya devueltas por esa respuesta (intensidad, velocidad emocional, fricción) y actualiza un panel de sesión sin recargar la página.

Es importante remarcar qué es y qué no es esta capa: **no añade ningún cálculo, modelo ni inferencia nuevos**. Cada punto del panel en vivo corresponde 1:1 a una respuesta que ya pasó por el pipeline completo de gobernanza y que ya tiene su propio `trace_id`. Es, literalmente, la telemetría conversacional descrita arriba, leída mensaje a mensaje en el momento en que se genera, en lugar de reconstruida más tarde desde el histórico completo vía `/replay`.

Indicadores del panel de sesión en vivo:

| Indicador | Qué muestra |
|-----------|-------------|
| **Tensión acumulada de la sesión** | Acumulado del medidor de tensión a lo largo de los mensajes ya analizados |
| **Velocidad emocional del último mensaje** | Variación de intensidad respecto al mensaje anterior |
| **Estado de aceleración** | Si la tensión está subiendo cada vez más rápido (aceleración de conflicto positiva) o se está frenando |

Al apoyarse en endpoints ya existentes (`/analizar` por mensaje), esta vista no requiere nueva infraestructura de *streaming*: es una capa de agregación en el frontend sobre respuestas ya entregadas y auditadas.

### Lectura narrativa de la telemetría

Para que la dinámica de una conversación se entienda sin necesidad de interpretar curvas técnicas, las métricas ya definidas (velocidad emocional, aceleración de conflicto, fricción, curvas de intención) se exponen también, de forma opcional en la capa visual, con un vocabulario narrativo:

| Término narrativo | Métrica subyacente |
|---|---|
| "Velocidad" | Velocidad emocional |
| "Fricción" | Fricción conversacional |
| "Cambio de dominio" | Cambio en qué participante marca el tono emocional dominante |
| "Punto de colisión" | Pico de conflicto / alta tensión |
| "Pausa" | Caída de tensión tras un cambio de tono |

Esta capa **no introduce ningún dato ni cálculo nuevo**: es un etiquetado alternativo sobre métricas ya calculadas y trazadas, pensado únicamente para hacer más legible el panel en vivo y el *heatmap* a un público no técnico, sin comprometer la trazabilidad ni la auditoría subyacentes.

### Posicionamiento

| Nivel | Descripción |
|-------|-------------|
| Sistemas tradicionales | Análisis de texto |
| SubtextAI sin Replay Mode | Sistema de análisis avanzado con arquitectura cloud |
| SubtextAI con Replay Mode | Motor de telemetría de la dinámica conversacional humana |
| SubtextAI con telemetría en vivo | Motor de telemetría conversacional, tanto retrospectivo (Replay) como en curso (sesión en vivo) |

---

## Arquitectura del Sistema

Arquitectura cloud sobre Azure, con separación clara entre frontend, backend y servicios de IA.

```
┌──────────────────────────────────────────────────────────────┐
│                    USUARIO (navegador)                        │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTPS
┌────────────────────────▼─────────────────────────────────────┐
│           Azure Static Web Apps (Frontend React)             │
└────────────────────────┬─────────────────────────────────────┘
                         │ REST API
┌────────────────────────▼─────────────────────────────────────┐
│            Azure App Service (.NET Backend)                  │
│                                                              │
│  Pipeline de gobernanza (cortocircuito en cascada):          │
│                                                              │
│  1. Validaciones de política (sin LLM)                       │
│     → longitud mínima, idioma, rate limit, prompt injection  │
│  2. Clasificador de crisis emocional (LLM call #1)           │
│     → GPT-4o-mini con prompt especializado (~150–200 tokens) │
│  3. RAG — Recuperación documental                            │
│     → Azure AI Search (búsqueda híbrida: vectorial+semántica)│
│  4. LLM principal — Análisis pragmático (LLM call #2)        │
│     → GPT-4o-mini con grounding obligatorio                  │
│  5. Trazabilidad                                             │
│     → Registro completo en base de datos con trace_id único  │
└──┬───────────┬──────────────────┬────────────────────────────┘
   │           │                  │
   ▼           ▼                  ▼
Azure OpenAI  Azure AI Search   Azure SQL / Cosmos DB
(GPT-4o-mini) (Índice documental (Trazas, métricas,
 clasificador  configurable con   versiones de prompts)
 + análisis)   búsqueda híbrida)        │
                                        ▼
                                 Azure Monitor ──► Azure Functions
                                 (Métricas         (Evaluación
                                  + alertas)        automatizada)
```

**Pipeline de cortocircuito en cascada:** si cualquier política se activa en un paso, el flujo se detiene y el modelo principal nunca se invoca. El orden es estricto: políticas → crisis emocional → RAG (umbral de score) → análisis pragmático → trazabilidad. Esto garantiza que el sistema no produzca respuestas fuera del control establecido.

---

## RAG y Grounding

El componente RAG es el núcleo del *grounding* documental: recupera los fragmentos más relevantes del corpus indexado **antes** de que el modelo genere nada.

**Pipeline de indexación** (independiente del corpus): los documentos se dividen en fragmentos (*chunking*), se vectorizan con Azure OpenAI Embeddings y se almacenan en Azure AI Search con **búsqueda híbrida** (vectorial + BM25). Tras la recuperación, un paso de **reordenamiento** (*reranking*) prioriza los fragmentos por relevancia contextual antes de pasarlos al modelo.

**Umbral de confianza:** el score medio de los fragmentos recuperados se traduce en un nivel de confianza explícito que acompaña a cada respuesta.

| Nivel | Score RAG medio | Comportamiento |
|-------|-----------------|----------------|
| **ALTA** | ≥ 0.75 | Respuesta completa con cita de fragmento y sección. `grounded=true`. |
| **MEDIA** | 0.40 – 0.74 | Respuesta con advertencia de contexto parcial. Se cita la fuente disponible. |
| **BAJA** | < 0.40 | Se activa `grounding_obligatorio`: el sistema no genera respuesta ni inventa. |

---

## Gobernanza del Agente

La gobernanza opera de forma independiente al modelo generativo y se apoya en cuatro mecanismos.

### Políticas explícitas

Reglas duras codificadas en el pipeline (no en el prompt). El modelo nunca se invoca si alguna se ha violado, de modo que el comportamiento de seguridad no depende del modelo ni de su configuración.

| Política | Condición de activación | Acción | HTTP |
|----------|------------------------|--------|------|
| `mensaje_minimo` | Menos de 5 palabras | Rechaza y solicita más contexto | 422 |
| `grounding_obligatorio` | Score RAG medio < 0.40 | Bloquea generación sin base documental | 422 |
| `crisis_detected` | Señales de crisis emocional | Bloquea y deriva a profesional | 422 |
| `prompt_injection` | Input malicioso detectado | Bloquea y registra en auditoría | 400 |
| `idioma_no_soportado` | Idioma distinto al configurado | Rechaza informando el idioma detectado | 422 |
| `rate_limit_usuario` | > 10 req/min por usuario/IP | Bloquea temporalmente con registro | 429 |
| `respuesta_sin_fuente` | Respuesta sin evidencia documental válida | Bloquea antes de entregarla | 422 |
| `confianza_insuficiente` | Score medio bajo el umbral | Rechaza generación | 422 |
| `politica_no_cumplida` | Inconsistencia interna entre políticas | Bloquea flujo | 500 |

### Clasificador de crisis emocional

Antes de cualquier análisis pragmático, una llamada independiente al modelo (LLM call #1) con un prompt especializado y minimalista (~150–200 tokens) evalúa si el mensaje contiene señales de crisis severa. Se optó por un clasificador basado en prompt —en lugar de un modelo *fine-tuned* o una lista de palabras clave— priorizando coherencia arquitectónica, auditabilidad y velocidad de desarrollo en el contexto de un MVP, con su evolución futura documentada. Cada activación de `crisis_detected` queda registrada con su `trace_id` y nivel de severidad.

### Trazabilidad y observabilidad

Cada interacción genera un registro persistente identificado por un `trace_id` único que permite reconstruir toda la cadena de decisiones del agente. El registro incluye: mensaje y contexto de entrada, fragmentos recuperados con sus scores individuales, prompt exacto y su versión (`prompt_version`), versión del modelo, nivel de confianza, política aplicada (si la hubo), campo `grounded`, latencia total y timestamp. Es accesible en tiempo real vía `GET /audit/{trace_id}`.

Estas trazas son, además, la **única fuente de datos** de la telemetría conversacional, del panel en vivo y del Modo de Reproducción: las tres capas leen lo ya persistido, sin generar análisis nuevo.

A nivel de sistema, **Azure Monitor** recibe las métricas agregadas generadas tanto por las interacciones en producción como por el job de evaluación, permitiendo monitorizar la evolución del comportamiento y comparar versiones de prompt de forma cuantitativa.

### Evaluación continua y versionado de prompts

Un job en **Azure Functions** ejecuta periódicamente un conjunto de preguntas de prueba y publica los resultados en Azure Monitor (porcentaje *grounded*, cumplimiento de políticas, distribución de confianza, latencia media, *cache hit rate*). Cada cambio de prompt genera una versión (`v1.1`, `v1.2`…) presente en cada respuesta y en `/audit`, lo que permite justificar cada cambio con datos observables y comparar el comportamiento antes y después.

---

## Tecnologías Utilizadas

| Capa | Tecnología |
|------|------------|
| **Frontend** | React + Vite + TypeScript, desplegado en Azure Static Web Apps |
| **Backend** | ASP.NET Core (C#) en Azure App Service — pipeline completo de gobernanza |
| **IA generativa** | Azure OpenAI Service — GPT-4o-mini (clasificación de crisis + análisis pragmático) |
| **RAG** | Azure AI Search — búsqueda híbrida (vectorial + semántica) con *reranking* |
| **Embeddings** | Azure OpenAI Embeddings |
| **Base de datos** | Azure SQL Database o Azure Cosmos DB — trazas completas por `trace_id` |
| **Observabilidad** | Azure Monitor — métricas de calidad y alertas |
| **Evaluación** | Azure Functions — job periódico de pruebas |
| **ORM** | Entity Framework Core — esquema y migraciones |
| **Seguridad** | Rate limiting por usuario/IP, detección de prompt injection, clasificación de crisis |

---

## Endpoints de la API

**Base URL:** `https://subtextai-api.azurewebsites.net/api/v1`

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| `POST` | `/analizar` | Interpreta el significado pragmático de un mensaje ambiguo |
| `POST` | `/evaluar` | Evalúa una respuesta propuesta por el usuario ante un mensaje analizado |
| `GET` | `/replay/{trace_id}` | Devuelve la conversación analizada y anotada para el Modo de Reproducción |
| `GET` | `/audit/{trace_id}` | Devuelve la trazabilidad completa y versionada de una respuesta |
| `GET` | `/metricas` | Métricas de calidad agregadas del sistema (últimos 7 días) |
| `GET` | `/prompts` | Historial de versiones de prompts con métricas comparativas |
| `GET` | `/health` | Estado del sistema y de los servicios Azure conectados |

### POST /analizar

Recibe un mensaje y un contexto, lo pasa por el pipeline completo de gobernanza y devuelve el análisis pragmático fundamentado en fuentes documentales.

**Request:**
```json
{
  "mensaje": "Solo quiero fluir y ver qué pasa",
  "contexto": "pareja | trabajo | social | negociacion"
}
```

**Response 200:**
```json
{
  "significado": "Baja implicación emocional, evasión de compromiso explícito",
  "senales": ["evasión", "ambigüedad intencional", "pasividad"],
  "nivel_alerta": "MEDIO",
  "recomendacion": "Mantener límites claros. No invertir energía emocional sin reciprocidad.",
  "fuente": {
    "documento": "<título del documento fuente>",
    "fragmento": "<sección o capítulo relevante>"
  },
  "confianza": {
    "nivel": "ALTA",
    "razon": "score_rag_medio: 0.87 — contexto documental sólido"
  },
  "grounded": true,
  "politica_aplicada": "ninguna",
  "idioma_detectado": "es",
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "metadata": {
    "latencia_ms": 1240,
    "modelo": "gpt-4o-mini",
    "version_modelo": "2024-07",
    "prompt_version": "v1.2",
    "from_cache": false,
    "sentimiento": { "label": "neutral", "score": 0.62 }
  }
}
```

**Códigos de error posibles:**

| Código | Política | Descripción |
|--------|----------|-------------|
| `422` | `mensaje_minimo` | Mensaje con menos de 5 palabras |
| `422` | `grounding_obligatorio` | Score RAG medio < 0.40; sin base documental suficiente |
| `422` | `crisis_detected` | Señales de crisis emocional; se deriva a profesional |
| `422` | `idioma_no_soportado` | Idioma distinto al español |
| `400` | `prompt_injection` | Input malicioso detectado y bloqueado |
| `429` | `rate_limit_usuario` | Más de 10 requests/minuto por usuario/IP |

### Otros endpoints

- **`POST /evaluar`** — valora una respuesta propuesta por el usuario ante un mensaje ya analizado: probabilidad de éxito, fortalezas, áreas de mejora y sugerencia alternativa, todo *grounded* en fuentes.
- **`GET /replay/{trace_id}`** — devuelve la conversación anotada para el Modo de Reproducción: anotaciones por mensaje, momentos críticos, evolución del medidor de tensión, datos del *heatmap* y explicaciones de cada inflexión. Toda la información procede de las trazas ya registradas.
- **`GET /audit/{trace_id}`** — registro completo de una interacción (fragmentos y scores, prompt y versión, modelo, políticas, latencia).
- **`GET /metricas`** — métricas agregadas de los últimos 7 días.
- **`GET /prompts`** — historial de versiones de prompts con métricas por versión.
- **`GET /health`** — estado del sistema y de cada servicio Azure conectado.

---

## Instalación

### Requisitos previos

- .NET 8 SDK o superior
- Node.js 18+ y npm
- Suscripción de Azure activa con acceso a Azure OpenAI Service, Azure AI Search y Azure App Service
- Azure CLI instalado y configurado (`az login`)
- Acceso al modelo `gpt-4o-mini` en tu instancia de Azure OpenAI
- Base de datos compatible (Azure SQL Database o Azure Cosmos DB)

### 1. Clonar el repositorio

```bash
git clone https://github.com/<tu-usuario>/subtextai.git
cd subtextai
```

### 2. Configurar el backend (.NET)

```bash
cd backend
dotnet restore
cp appsettings.Example.json appsettings.Development.json
```

Edita `appsettings.Development.json` con tus credenciales de Azure:

```json
{
  "AzureOpenAI": {
    "Endpoint": "https://<tu-instancia>.openai.azure.com/",
    "ApiKey": "<tu-api-key>",
    "DeploymentName": "gpt-4o-mini",
    "ModelVersion": "2024-07"
  },
  "AzureAISearch": {
    "Endpoint": "https://<tu-instancia>.search.windows.net",
    "ApiKey": "<tu-api-key>",
    "IndexName": "<nombre-del-indice>"
  },
  "ConnectionStrings": {
    "DefaultConnection": "<tu-cadena-de-conexion>"
  },
  "Policies": {
    "MinWordCount": 5,
    "RagScoreThreshold": 0.40,
    "RateLimitPerMinute": 10
  }
}
```

Aplica las migraciones:

```bash
dotnet ef database update
```

### 3. Indexar documentos en Azure AI Search

Coloca los documentos fuente en `scripts/docs/` (PDF o texto plano) y ejecuta el script de indexación, que se encarga del *chunking*, la vectorización y la carga:

```bash
cd scripts
dotnet run --project IndexDocuments -- --source ./docs --index <nombre-del-indice>
```

El corpus es configurable: el script acepta cualquier colección compatible sin cambios en el backend ni en la lógica de gobernanza.

### 4. Configurar el frontend (React)

```bash
cd ../frontend
npm install
cp .env.example .env.local
```

Edita `.env.local` con la URL del backend local:

```env
VITE_API_BASE_URL=https://localhost:7000/api/v1
```

---

## Ejecución en Desarrollo

```bash
# Backend (desde backend/)
dotnet run --environment Development      # → https://localhost:7000

# Frontend (desde frontend/, en otro terminal)
npm run dev                               # → http://localhost:5173
```

Estado del sistema: `https://localhost:7000/api/v1/health`.

---

## Despliegue en Azure

```bash
# Backend → Azure App Service
cd backend
dotnet publish -c Release -o ./publish
az webapp deploy --resource-group subtextai-rg --name subtextai-api --src-path ./publish

# Frontend → Azure Static Web Apps
cd frontend
npm run build
az staticwebapp deploy --app-location "." --output-location "dist" --name subtextai-frontend
```

Actualiza `VITE_API_BASE_URL` en la configuración de la Static Web App para apuntar a la URL de producción del backend.

---

## Estructura de Carpetas

```
subtextai/
│
├── backend/                          # API REST en ASP.NET Core
│   ├── Controllers/                  # Endpoints: Analizar, Evaluar, Replay, Audit, Metricas, Prompts, Health
│   ├── Pipeline/                     # Lógica del pipeline de gobernanza
│   │   ├── PolicyEngine.cs           # Validaciones de política sin LLM
│   │   ├── CrisisClassifier.cs       # Clasificador de crisis emocional (LLM call #1)
│   │   ├── RagService.cs             # Recuperación documental con Azure AI Search
│   │   ├── LlmService.cs             # Análisis pragmático principal (LLM call #2)
│   │   └── TraceStore.cs             # Persistencia de trazas con trace_id
│   ├── Models/                       # DTOs de request y response
│   ├── Prompts/                      # Prompts versionados del sistema
│   ├── Migrations/                   # Migraciones de base de datos (EF Core)
│   ├── appsettings.json              # Configuración base
│   └── appsettings.Example.json      # Plantilla de configuración (sin secretos)
│
├── frontend/                         # Aplicación React
│   ├── src/
│   │   ├── components/               # Componentes UI reutilizables (incluye panel de telemetría en vivo)
│   │   ├── pages/                    # Vistas (Análisis, Reproducción, Auditoría, Métricas)
│   │   ├── services/                 # Capa de integración con la API REST
│   │   └── types/                    # Tipos TypeScript para respuestas del backend
│   ├── .env.example
│   └── vite.config.ts
│
├── scripts/                          # Utilidades de indexación y evaluación
│   ├── IndexDocuments/               # Carga de documentos en Azure AI Search
│   └── EvaluationJob/                # Preguntas de prueba para evaluación automatizada
│
└── docs/                             # Documentación del proyecto
```

---

## Mejoras Futuras

Líneas de evolución contempladas. Salvo indicación contraria, quedan **fuera del alcance actual** del sistema, que es descriptivo y opera siempre sobre trazas ya persistidas.

- **Clasificador de crisis especializado** — al superar ~10.000 requests/día, evaluar un modelo *fine-tuned* (DistilBERT o equivalente) para reducir coste por token y mejorar precisión.
- **Soporte multilingüe** — idiomas adicionales con prompts y pipelines de evaluación específicos.
- **Expansión del corpus documental** — nuevas fuentes especializadas sin modificar la lógica de gobernanza.
- **Panel de auditoría** — interfaz para visualizar trazas, comparar versiones de prompts y revisar activaciones de políticas.
- **Mejora de observabilidad** — más indicadores en Azure Monitor para detectar tendencias y cambios de comportamiento.
- **Clasificador determinista adicional** — segunda capa basada en reglas sobre la clasificación LLM para escenarios de auditoría regulatoria.
- **Caché semántico** — cacheo por similitud semántica entre consultas para reducir llamadas al modelo en mensajes equivalentes.
- **Streaming del análisis en tiempo real** — *exploración futura*. Hoy tanto el análisis como la reproducción operan sobre trazas ya persistidas, y la telemetría en vivo descrita arriba se limita a agregar respuestas ya generadas por `/analizar`; se contempla estudiar un modo de análisis verdaderamente incremental (parcial, antes de que el mensaje se complete) durante la propia conversación. No forma parte de las capacidades actuales.
- **Motor predictivo de trayectoria / impacto de decisión** — *exploración futura*. Sobre las curvas de intención y la telemetría conversacional ya formalizadas, se contempla investigar la estimación del impacto de un mensaje sobre los próximos turnos (por ejemplo, una probabilidad estimada de escalada). Esto requeriría un modelo de proyección entrenado y validado de forma independiente, con su propio proceso de evaluación y auditoría, antes de poder exponerse como una predicción fiable. Las capacidades actuales del sistema son estrictamente descriptivas, no predictivas, y este motor queda explícitamente fuera del alcance hasta que exista esa validación.

---

## Autora

**Heily Madelay Tandazo**

---

## Licencia

Proyecto desarrollado con fines académicos. Distribuido bajo la licencia MIT — consulta [LICENSE](LICENSE) para más información.
