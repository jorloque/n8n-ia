# n8n Workflows — Talleres IES Juan de Garay

Colección de 4 talleres prácticos sobre automatización con **n8n**, cubriendo desde pipelines ETL hasta agentes de IA con RAG y bots de Telegram.

---

## 📋 Índice

- [Taller 1 — Pokemon Scraper](#taller-1--pokemon-scraper)
- [Taller 2 — Wikipedia Agent](#taller-2--wikipedia-agent)
- [Taller 3 — Sistema RAG Estándar](#taller-3--sistema-rag-estándar)
- [Taller 4 — Telegram Bot](#taller-4--telegram-bot)
- [Requisitos generales](#requisitos-generales)
- [Comparativa de workflows](#comparativa-de-workflows)

---

## Taller 1 — Pokemon Scraper

**Patrón:** Pipeline ETL  
**Trigger:** Manual

### ¿Qué hace?

Automatiza la consulta y el almacenamiento de datos de Pokémons. Lee una lista de IDs desde un Google Sheet, filtra los registros que aún no tienen datos, consulta la [PokéAPI](https://pokeapi.co/) por cada uno, extrae la información relevante, la escribe de vuelta en el Sheet y finalmente envía un correo de resumen vía Gmail.

### Nodos (8 en total)

| # | Nodo | Tipo | Descripción |
|---|------|------|-------------|
| 1 | Click | Manual Trigger | Punto de arranque manual del workflow |
| 2 | Obtener IDs de Pokémons | Google Sheets Read | Lee todas las filas del Sheet (ID, Nombre, Tipo, Sprites) |
| 3 | Filtrar Pokémons a buscar | Filter | Solo pasa filas con Nombre vacío (no procesadas) |
| 4 | Obtener información Pokémon | HTTP Request | Consulta `https://pokeapi.co/api/v2/pokemon/{ID}` |
| 5 | Extraer datos del Pokémon | Set / Edit Fields | Mapea los 5 campos necesarios del JSON de la API |
| 6 | Actualizar información | Google Sheets Update | Actualiza la fila correspondiente buscando por columna ID |
| 7 | Agrupar resultados | Aggregate | Consolida todos los ítems en uno para enviar un solo correo |
| 8 | Enviar correo de resultado | Gmail | Envía resumen HTML del proceso |

### Requisitos previos

- **Google Sheets:** Crear una hoja con columnas `ID`, `Nombre`, `Tipo`, `Sprite Frontal`, `Sprite Posterior`. Configurar credenciales OAuth2 en n8n (`Settings > Credentials > New > Google Sheets OAuth2`).
- **Gmail:** Configurar credenciales OAuth2 de Gmail en n8n.
- **PokéAPI:** No requiere API key. Límite de 100 peticiones/minuto (añadir nodo `Wait` si se procesan más de 100 Pokémons).
- **Google Cloud:** Crear un proyecto en [console.cloud.google.com](https://console.cloud.google.com), habilitar las APIs de Google Sheets, Drive y Gmail, y generar credenciales OAuth2.

### Flujo de datos

```
Click → Google Sheets (Read) → Filter → HTTP Request → Edit Fields → Google Sheets (Update) → Aggregate → Gmail
```

---

## Taller 2 — Wikipedia Agent

**Patrón:** Agente reactivo con herramientas  
**Trigger:** Chat webhook

### ¿Qué hace?

Implementa un agente de IA llamado **Ana** que responde preguntas buscando información en Wikipedia en tiempo real. El agente mantiene memoria de conversación (últimos 10 mensajes) y está disponible a través de una interfaz de chat web embebible (`index.html`).

### Arquitectura

```
Chat Trigger (Webhook público)
        ↓
   AI Agent (razonamiento + decisión)
   ├── Chat Model: Ollama / OpenAI / Google Gemini
   ├── Memory: Simple Memory (10 mensajes)
   └── Tool: Wikipedia
```

### Nodos (7 en total, 2 opcionales deshabilitados)

| Nodo | Descripción |
|------|-------------|
| Chat Trigger | Webhook público que recibe mensajes del frontend |
| AI Agent | Cerebro del workflow, gestiona el ciclo razonamiento-acción |
| Ollama Chat Model *(activo)* | Modelo LLM local sin coste |
| OpenAI GPT-4.1-mini *(off)* | Alternativa de pago |
| Google Gemini *(off)* | Alternativa gratuita con límites |
| Simple Memory | Historial de los últimos 10 mensajes por sesión |
| Wikipedia Tool | Herramienta de búsqueda en Wikipedia |

### System Prompt de Ana

```
## Objetivo
Eres un asistente amable, tu nombre es Ana, ayudas a buscar información en Wikipedia.

## Reglas
- Devuelve siempre respuestas de dos párrafos o menos.
- Siempre pon las referencias de los artículos que consultas en Wikipedia.

## Tools
- Tienes una "tool" para conectarte a Wikipedia y traer información.
```

### Modelos LLM disponibles (gratuitos)

| Proveedor | URL | Notas |
|-----------|-----|-------|
| Ollama | Local (`http://ollama:11434`) | Sin límites, requiere GPU/RAM local |
| Google Gemini | [aistudio.google.com](https://aistudio.google.com) | 5 RPM / 20 RPD en capa gratuita |
| Groq | [console.groq.com](https://console.groq.com) | Varios modelos gratuitos |
| OpenRouter | [openrouter.ai](https://openrouter.ai) | Modelos `*:free` disponibles |

### Frontend (index.html)

El archivo `index.html` usa la librería [`@n8n/chat`](https://www.npmjs.com/package/@n8n/chat). Solo hay que actualizar la propiedad `webhookUrl` con la URL del Chat Trigger del workflow.

---

## Taller 3 — Sistema RAG Estándar

**Patrón:** Retrieval-Augmented Generation (2 fases)  
**Caso de uso:** Asistente legal que consulta documentos PDF internos

### ¿Qué hace?

Implementa un sistema RAG completo que permite a un abogado hacer preguntas en lenguaje natural sobre documentos PDF propios. El sistema carga los PDFs, los vectoriza y los almacena; luego, ante cada pregunta, recupera los fragmentos más relevantes y genera una respuesta fundamentada con referencia al documento.

### Arquitectura (dos pipelines)

**Fase 1 — Ingestión** *(se ejecuta una vez por lote de documentos)*
```
Form Trigger → Simple Vector Store (insert)
                  ├── Default Data Loader (Binary PDF)
                  ├── Recursive Character Text Splitter
                  └── Embeddings Ollama
```

**Fase 2 — Consumo RAG** *(se activa con cada pregunta del chat)*
```
Chat Trigger → AI Agent → [respuesta]
                  ├── Ollama Chat Model
                  ├── Simple Memory (10 mensajes)
                  └── Simple Vector Store (retrieve-as-tool)
                             └── Embeddings Ollama (compartido)
```

### Nodos (10 en total)

| # | Nodo | Fase | Descripción |
|---|------|------|-------------|
| 1 | On form submission | Ingestión | Formulario web para subir PDFs |
| 2 | Default Data Loader | Ingestión | Extrae texto binario del PDF |
| 3 | Recursive Character Text Splitter | Ingestión | Divide en chunks de 1000 chars, overlap 250 |
| 4 | Embeddings Ollama | Compartido | Vectoriza chunks y preguntas (mismo modelo) |
| 5 | Simple Vector Store (insert) | Ingestión | Almacena embeddings en memoria (`vector_store_key`) |
| 6 | Chat Trigger | Consumo | Recibe preguntas del abogado |
| 7 | AI Agent | Consumo | Orquesta RAG: retrieve + augment + generate |
| 8 | Ollama Chat Model | Consumo | Genera la respuesta final |
| 9 | Simple Memory | Consumo | Historial de conversación (10 mensajes) |
| 10 | Simple Vector Store (retrieve) | Consumo | Busca chunks similares por distancia coseno |

### Configuración del Text Splitter

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| Chunk size | 1000 | Tamaño máximo de cada fragmento en caracteres |
| Chunk overlap | 250 | Solapamiento para mantener contexto entre chunks |

> ⚠️ **Importante:** Usar **Ollama 3.1** para los embeddings. Modelos posteriores no gestionan bien la vectorización en n8n.

> ⚠️ El vector store en memoria es **volátil**: si se reinicia n8n hay que volver a cargar los documentos. Para producción, usar Pinecone, Qdrant o Supabase (pgvector).

### System Prompt del Asistente Legal

```
# Rol: Asistente Legal

## Descripcion
Eres un asistente legal para ayudar a un abogado a consultar documentos legales internos.
Respuestas precisas, claras y fundamentadas. Tono profesional y formal.
Si no conoces la respuesta, indica que no dispones de la información.

## Capacidades
- Búsqueda y recuperación de información legal relevante.
- Respuestas concisas con fundamento en los documentos.
- Inclusión de referencia del documento consultado (nombre de archivo, cláusula).

## Comportamiento
1. Recepción de la consulta del abogado.
2. Búsqueda en la base de conocimiento.
3. Si hay información: resumen conciso + referencia.
   Si no hay información: "No puedo encontrar la respuesta en los recursos disponibles."
4. Confidencialidad total.
```

### Orden de uso

1. Ejecutar **Fase 1**: subir PDFs desde el formulario del Form Trigger.
2. Esperar a que termine la ingestión.
3. Usar el **chat** para hacer preguntas sobre los documentos.

---

## Taller 4 — Telegram Bot

**Patrón:** Bot conversacional con enrutamiento  
**Trigger:** Telegram webhook

### ¿Qué hace?

Asistente de cultura general que opera dentro de Telegram usando un modelo de lenguaje local (llama3.1 vía Ollama), **sin APIs de pago**. Incluye sticker animado de bienvenida (Giphy), memoria individual por usuario (por `chat_id`) e indicador visual de "escribiendo...".

### Flujo principal

```
Telegram Trigger
        ↓
   Switch (¿Qué tipo de mensaje?)
   ├── /start → Obtener sticker (Giphy) → Descargar GIF → Enviar sticker
   └── text-message → Escribiendo... → Edit Fields → Unir datos
                                                          ↓
                                                      AI Agent
                                                      ├── Ollama Chat Model
                                                      ├── Simple Memory (10 msgs, key=chat_id)
                                                      └── Telegram Tool (Send Message)
                                                          ↓
                                                      Edit Fields → Respuesta JS → Send text
```

### Nodos destacados

| Nodo | Descripción |
|------|-------------|
| Telegram Trigger | Escucha todos los mensajes entrantes al bot |
| Switch | Enruta según `/start` (texto exacto) o existencia de texto |
| HTTP Request (Giphy) | `GET https://api.giphy.com/v1/stickers/random` |
| HTTP Request (download) | Descarga el GIF en binario para enviarlo como sticker |
| Send Chat Action | Envía la acción `typing` para mostrar "escribiendo..." |
| Edit Fields | Extrae `text`, `type` y campo `ai` del mensaje |
| Code (Unir datos) | Fusiona los datos de Edit Fields en un único objeto |
| AI Agent | Genera la respuesta usando Ollama + memoria por usuario |
| Telegram Tool | Herramienta interna del agente para enviar mensajes |
| Code (Respuesta JS) | Extrae el texto limpio del output del agente |

### Configuración del bot

1. Abrir [@BotFather](https://t.me/BotFather) en Telegram.
2. Ejecutar `/newbot` y seguir los pasos.
3. Copiar el **token** generado.
4. En n8n, crear una credencial `Telegram API` con ese token.
5. Activar el workflow → el webhook se registra automáticamente.

### Requisitos

- **Giphy API:** Crear cuenta en [developers.giphy.com](https://developers.giphy.com) y obtener un API key gratuito.
- **Ollama:** Instancia local o en Docker con el modelo `llama3.1` descargado.

### System Prompt

```
Eres un asistente que ayuda a las personas con problemas de cultura general.
Siempre responde los mensajes usando la herramienta de Telegram.
```

---

## Requisitos generales

- **n8n** versión 1.x (self-hosted o cloud)
- **Ollama** con los modelos `llama3.2` (o `llama3.1` para embeddings) disponibles
- Cuenta de **Google Cloud** con las APIs de Sheets, Drive y Gmail habilitadas
- Node.js para la librería `@n8n/chat` (opcional, solo Taller 2)

---

## Comparativa de workflows

| Aspecto | Taller 1 — Pokémon Scraper | Taller 2 — Wikipedia Agent | Taller 3 — Sistema RAG | Taller 4 — Telegram Bot |
|---------|---------------------------|---------------------------|------------------------|-------------------------|
| **Patrón** | Pipeline ETL | Agente reactivo | RAG (2 fases) | Bot conversacional |
| **Fuente de datos** | PokéAPI (pública) | Wikipedia en tiempo real | Documentos PDF propios | Conocimiento del modelo |
| **Uso de IA** | Sin IA | LLM + tool search | LLM + embeddings + vector DB | LLM local |
| **Memoria** | Sin estado | Buffer 10 mensajes | Buffer 10 mensajes | Buffer 10 msgs por usuario |
| **Persistencia** | Google Sheets | En memoria | En memoria (volátil) | En memoria |
| **Trigger** | Manual | Chat webhook | Form + Chat webhook | Telegram webhook |
| **Coste IA** | Ninguno | Opcional (gratuito) | Opcional (gratuito) | Ninguno (Ollama local) |

---

## 📄 Licencia

Material educativo elaborado por el **IES Juan de Garay** (Valencia) en el marco de los talleres de automatización con n8n.