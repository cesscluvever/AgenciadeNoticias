# Stack tecnológico — Agente "News Reporter"

Propuesta de stack para el caso práctico del máster (EBIS), adaptada al entorno de Jose: n8n autoalojado en Docker (n8n.cesscluv.com vía túnel de Cloudflare), contenedor Ollama con qwen2.5:7b, y GitHub como repositorio del entregable. Ollama ha sido reemplazado en esta solución por soluciones IA en Cloud.

## 1. Arquitectura general

```mermaid
flowchart LR
    subgraph Disparo
        A1[Schedule Trigger\ncron diario]
        A2[Manual Trigger\ndemo/video]
    end

    subgraph Ingesta
        B[RSS Feed Read\nXataka — Inteligencia Artificial]
        B2[Limit: 6 ítems\nmás recientes]
    end

    subgraph "Procesamiento IA"
        C1[Claude API: resumen + tono]
        C2[Claude API: prompt de imagen]
    end

    subgraph "Generación visual"
        D[Nano Banana 2 Lite\ngemini-3.1-flash-lite-image]
    end

    subgraph Salida
        E[Telegram\nbot + grupo dedicado]
    end

    subgraph "Deduplicación y log"
        G1[Google Sheets\nGet Row(s): filtrar por link]
        G2{¿Ya está\nen el log?}
        G3[Google Sheets\nAppend Row]
    end

    subgraph Errores
        F[Error Trigger\nnotificación de fallo]
    end

    A1 --> B
    A2 --> B
    B --> B2
    B2 --> G1
    G1 --> G2
    G2 -- Sí --> H[Fin: ya procesada]
    G2 -- No --> C1
    C1 --> C2
    C2 --> D
    D --> E
    E --> G3
    B -. error .-> F
    C1 -. error .-> F
    D -. error .-> F
```

Todo el flujo vive en la instancia n8n ya montada; GitHub no ejecuta nada, es el repositorio donde se versiona el blueprint y la documentación del entregable.

**Actualización tras la puesta en producción — soporte para varios artículos nuevos por ejecución:** el diseño inicial asumía un único artículo por ejecución ("Limit: 1 ítem, el más reciente" + un Get Row(s)/IF por artículo). En el uso real, con un disparo diario, es habitual que se publique más de un artículo nuevo entre una ejecución y la siguiente — con el diseño de 1 ítem, los artículos adicionales se perdían sin registro. El diseño final soporta varios artículos nuevos (máximo 6) por ejecución:

- **Limit** se sube a un tope de seguridad de **6** artículos por ejecución (en vez de 1), para acotar el gasto en un día con mucha actividad sin limitarse a procesar solo el más reciente.
- La deduplicación por artículo individual (Get Row(s) + IF) se sustituye por **Leer Log Completo** (lee toda la hoja una vez, en paralelo al RSS) + **Filtrar Noticias Nuevas** (un Code node que cruza en bloque los candidatos contra los links ya registrados). El patrón anterior funcionaba con 1 candidato, pero con varios en paralelo perdía silenciosamente los que no coincidían con el log — de ahí el cambio.
- El nodo de Telegram lleva activado **Retry On Fail** (3 reintentos): con varias imágenes en la misma ejecución, una desconexión transitoria subiendo una foto no debe tirar abajo todo el envío.

## 2. Componentes y justificación

### Orquestación: n8n (autoalojado)

La instancia ya corre en Docker con volumen persistente y expuesta vía Cloudflare Tunnel — no hace falta crear nada nuevo, solo un workflow adicional en la misma instancia. Ventaja frente a Make: acceso a los nodos de LangChain (AI Agent, Basic LLM Chain, Structured Output Parser), que encajan mejor con "resumir + adaptar tono + redactar prompt de imagen" que el enfoque más rígido de Make.

### Disparador: Schedule Trigger + Manual Trigger en paralelo

El caso pide "un disparador". Se montan **dos** entradas al mismo flujo — Schedule Trigger (p. ej. cada mañana) para el caso de uso real, y Manual Trigger para poder ejecutar bajo demanda en el vídeo de la entrega sin esperar al cron. 

### Fuente de noticias: RSS

**Decisión confirmada: Xataka — etiqueta "Inteligencia Artificial"**, vía el nodo **RSS Feed Read** de n8n.

URL del feed: https://www.xataka.com/tag/inteligencia-artificial/rss2.xml

Verificado en vivo antes de cerrar la decisión: feed RSS 2.0 válido, publicando activamente contenido exclusivamente de IA. Ventaja clave frente a un feed general (El País Tecnología, Xataka general, Genbeta): **ya viene pre-filtrado por tema**, así que no hace falta un nodo Filter adicional para descartar noticias fuera de foco — encaja directamente con el "innovación, tecnología y negocios" que pide el enunciado, y además es temáticamente coherente (un agente de IA que reporta sobre noticias de IA).

Como el nodo RSS Feed Read devuelve **todos** los ítems del feed en cada ejecución (no solo los nuevos), justo después va un nodo **Limit** a 6 ítems para quedarte con los más recientes.

Sin API key, sin autenticación — el nodo solo necesita la URL, lo que reduce puntos de fallo de cara a la demo.

### Procesamiento IA: dos pasos encadenados

1. **Resumen + adaptación de tono** (informativo, cercano, divulgativo) sobre el artículo del RSS.
2. **Redacción del prompt de imagen** a partir del resumen generado.

**Decisión confirmada: Claude (Anthropic) vía API**, con Structured Output Parser (JSON con resumen, tono, prompt_imagen).

⚠️ **Importante sobre la suscripción Claude Pro:** no sirve para esto — son dos productos distintos. Claude Pro da acceso a Claude en claude.ai (web, escritorio, móvil). El nodo Anthropic de n8n necesita una **API key de la Claude Console** (console.anthropic.com), la plataforma de desarrolladores, que se factura aparte por tokens consumidos. El coste real para este proyecto es prácticamente irrelevante: cada ejecución mueve un par de miles de tokens, del orden de céntimos de dólar incluso con Sonnet.

Alternativa sin facturación: Ollama local (qwen2.5:7b) vía el nodo Ollama Chat Model — coste cero pero con más irregularidad en el JSON estructurado. Para la entrega evaluada, API de Claude es la opción más sólida.

### Generación de imagen

**Decisión confirmada: Nano Banana 2 Lite (`gemini-3.1-flash-lite-image`)**, vía HTTP Request a la API de Google (no hay nodo nativo en n8n, así que se monta como HTTP Request con la API key de Google AI Studio como credencial Header Auth). El modelo original, `gemini-2.5-flash-image`, pasó a estado legacy — Google recomienda migrar a esta versión, más rápida y barata.

**Corrección sobre el nivel gratuito (verificado en producción):** al construir el flujo, la llamada real a la API devolvía sistemáticamente `429 RESOURCE_EXHAUSTED` con `limit: 0` para el modelo de generación de imagen, en ambos modelos probados — es decir, la cuota gratuita para *generar imágenes* con Gemini es 0 en un proyecto sin facturación vinculada, a diferencia de lo que sugiere la documentación general de AI Studio (pensada sobre todo para los modelos de texto). La solución fue activar la facturación en el proyecto de Google Cloud asociado a la API key ("n8n-connection"); a partir de ahí las llamadas funcionan con normalidad. Coste real: ~0,039 $/imagen (la mitad vía Batch API) — para este proyecto, unos pocos céntimos.

Importante: se descarga la imagen generada dentro del flujo (el nodo HTTP Request trae el binario, no solo la URL/base64 en crudo) para reenviarla como adjunto real a Telegram, no como un enlace. Gemini devuelve la imagen como base64 dentro de la respuesta JSON, así que un paso posterior decodifica ese base64 a un binario real de n8n antes de enviarlo a Telegram.

### Canal de salida

**Decisión confirmada: Telegram, con cuenta personal.**

El enunciado pide "un canal de salida" de forma genérica (Slack/Discord/Email/Telegram), no exige que sea una cuenta corporativa, y el propio caso práctico es una simulación individual del proceso de un equipo editorial.

- **No se envía el resultado a un chat personal (DM).** Se creó un **bot dedicado** con BotFather y un **grupo de Telegram** propio (Agencia de Noticias IA) donde se añadió ese bot, simulando el canal de distribución de un equipo editorial.
- Nota: "se eligió Telegram por rapidez de configuración y fiabilidad para una demo individual; en un entorno real de equipo sería Slack, con el mismo patrón de nodo de salida."
- Técnicamente: nodo Telegram nativo de n8n, credencial = token del bot (BotFather), envío de imagen + caption en una sola llamada (sendPhoto con caption).

**Chat ID del grupo ya resuelto:** -5346073464 (ver valores de configuración en la sección 8).

### Registro histórico y deduplicación: Google Sheets

**Decisión confirmada:** una hoja de Google Sheets como log de noticias capturadas, con doble función — evitar reprocesar el mismo artículo y mantener un histórico consultable.

Columnas:

| Columna | Contenido |
|---|---|
| link | URL del artículo (clave de deduplicación — es estable y única por ítem del RSS) |
| titulo_original | Título tal como viene del feed |
| fecha_publicacion | pubDate/isoDate del RSS |
| fecha_captura | Timestamp de cuando el flujo lo procesó |
| resumen_generado | Salida de Claude |
| tono_aplicado | Campo del structured output (útil para auditar consistencia) |
| prompt_imagen | Prompt enviado a Nano Banana |
| imagen | Enlace/ID de Drive de la imagen si se decide persistirla |
| estado | enviado / error |
| canal_destino | Telegram (por si en el futuro se añaden más canales) |

Mecánica en el flujo (ya reflejada en el diagrama): justo después del Limit, un nodo **Get Row(s)** con filtro `link = {{ $json.link }}` comprueba si el artículo ya está en el log. Si hay coincidencia, el flujo termina ahí (rama "Fin: ya procesada") sin gastar llamadas de Claude ni de Nano Banana — esto es importante porque el filtrado ocurre *antes* de las llamadas a las APIs de pago, no después. Si no hay coincidencia, el flujo sigue el camino normal y, tras el envío correcto a Telegram, un nodo **Append Row** añade la fila nueva al log.

Credencial: **OAuth2 de Google Sheets** conectando la cuenta personal de Gmail — es la vía más simple para un proyecto individual (frente a una cuenta de servicio, que tiene sentido en un entorno de equipo/producción pero es una capa de configuración innecesaria aquí).

### Gestión de errores

Workflow secundario con **Error Trigger**, enganchado al workflow principal (Settings → Error Workflow), que envía una notificación por Telegram con el nombre del workflow, el nodo que falló y el mensaje de error. Cubre el punto "explica si se ha usado un gestor de errores y cómo funciona" sin añadir complejidad extra al flujo principal.

### GitHub: repositorio del entregable

n8n no se conecta a GitHub en tiempo de ejecución — GitHub aquí es el contenedor de la entrega, versionado y legible por el evaluador:

```
news-reporter/
├── README.md                  # qué hace el flujo, decisiones clave, cómo replicarlo
├── workflow/
│   └── news-reporter.json     # blueprint exportado de n8n
├── docs/
│   ├── sticky-notes.md        # transcripción de las sticky notes del canvas
│   └── arquitectura-y-decisiones.md
└── media/
    └── demo-video-link.md     # enlace al vídeo
```

Un commit por cada iteración relevante del flujo da además un historial de decisiones, útil si el evaluador pide ver la evolución.

## 3. Cobertura de los requisitos mínimos

| Requisito del enunciado | Componente |
|---|---|
| Disparador (programado / manual / webhook) | Schedule Trigger + Manual Trigger |
| HTTP Request de noticias reales (RSS/API) | RSS Feed Read — Xataka (Inteligencia Artificial) |
| IA: resumir, adaptar tono, redactar prompt de imagen | AI Agent (Claude API) + Structured Output Parser |
| Generación de imagen | Nano Banana 2 Lite (gemini-3.1-flash-lite-image) |
| Canal de salida con resumen + imagen | Telegram (bot dedicado) |
| Blueprint + explicación + sticky notes | Export JSON + README + sticky notes en el canvas de n8n, versionados en GitHub |
| Gestor de errores (si aplica) | Error Trigger workflow con notificación |

## 4. Almacenamiento y persistencia — ¿dónde vive cada cosa?

No se necesita ninguna carpeta local para el log — al ser una Google Sheet, por definición vive en Google Drive, en la nube de Google, no en el equipo local. La solución mezcla infraestructura propia (self-hosted) con servicios gestionados:

| Dato | Dónde vive | ¿Cloud gestionado o infraestructura propia? |
|---|---|---|
| Definición del workflow (JSON) | Base de datos interna de n8n, volumen Docker persistente | Propia — self-hosted |
| Credenciales (Claude, Google, Telegram) | Almacén de credenciales de n8n, cifrado, mismo volumen Docker | Propia — self-hosted |
| Histórico de ejecuciones de n8n | Base de datos interna de n8n, mismo volumen | Propia — self-hosted |
| Log de noticias (Sheets) | Google Drive, cuenta personal | Cloud gestionado (Google) |
| Imágenes enviadas | Servidores de Telegram | Cloud gestionado (Telegram) |
| Blueprint exportado + documentación | Repositorio Git | Cloud gestionado (GitHub) |
| Llamadas a Claude / Nano Banana | Sin persistencia propia — APIs stateless | Cloud gestionado (Anthropic / Google) |

La solución **no es "todo cloud"** — tiene una pieza autoalojada (la instancia n8n con su volumen Docker) que concentra tres tipos de dato crítico: la definición del flujo, las credenciales y el histórico de ejecuciones. Como el workflow y sus credenciales solo existen en ese volumen Docker local, **si se pierde esa máquina o el volumen, se pierde el flujo** — el histórico de noticias sobreviviría (está en Sheets, en la nube), pero no el diseño del workflow en sí. Aquí es donde el export del blueprint a GitHub deja de ser solo un requisito de entrega y pasa a ser, de facto, backup/recuperación ante desastre.

## 5. Checklist de preparación

Los cinco bloques (cuentas y credenciales externas, recursos de datos, configuración dentro de n8n, y contexto para Claude Code) están cerrados:

- Cuenta en console.anthropic.com, facturación activada, API key generada.
- Proyecto en Google Cloud con la Gemini API habilitada, y API key de AI Studio para Nano Banana.
- Google Sheets API y Google Drive API habilitadas, cliente OAuth2 creado (Redirect URI: `https://n8n.cesscluv.com/rest/oauth2-credential/callback`).
- Bot de Telegram creado con BotFather, grupo dedicado creado, bot añadido como miembro/admin.
- Repositorio en GitHub creado ("AgenciadeNoticias"), con la estructura de carpetas montada.
- Google Sheet "Log de Noticias" creada, con la fila de cabeceras (10 columnas) ya en la primera fila.
- Credenciales Anthropic, Telegram y Google Sheets OAuth2 creadas en n8n y verificadas (conectado).
- Claude Code construye el flujo llamando a la API REST de n8n — API Key de n8n generada y vigente. **Nota de ejecución:** en la práctica, Cloudflare Access protege todo `n8n.cesscluv.com` (excepto `/webhook*`), por lo que las llamadas REST externas quedan redirigidas al login de Cloudflare Access. Se optó por construir el flujo mediante `docker exec n8n n8n import:workflow`/`export:workflow` mientras se resuelve esa capa de Access para la API, manteniendo el mismo resultado (workflow versionado, sin intervención manual en el editor).

## 6. Decisiones

| Decisión | Elección | Estado |
|---|---|---|
| LLM de procesamiento | Claude (Anthropic API, vía Claude Console — no la suscripción Pro) | ✅ Confirmada |
| API de imagen | Nano Banana 2 Lite (gemini-3.1-flash-lite-image) | ✅ Confirmada |
| Canal de salida | Telegram, bot dedicado + grupo propio (no DM personal) | ✅ Confirmada |
| Medio de noticias | Xataka — RSS "Inteligencia Artificial" | ✅ Confirmada |
| Log / deduplicación | Google Sheets (OAuth2, cuenta personal) | ✅ Confirmada |
| Interacción Claude Code ↔ n8n | CLI de n8n vía Docker (API REST bloqueada por Cloudflare Access) | ✅ Confirmada |

## 7. Especificación del nodo IA — prompt de sistema y schema

### Prompt de sistema

Eres el redactor de IA del equipo de contenidos de un medio digital español especializado en innovación, tecnología y negocios. Tu trabajo es convertir un artículo de prensa real en una pieza breve, lista para publicar en el canal de distribución interno del equipo.

Recibirás el titular, la fecha de publicación, el enlace y el contenido (puede incluir HTML) de una noticia real. A partir de ese material, y solo de ese material, genera tres elementos:

1. "resumen": un resumen de la noticia en español, entre 60 y 100 palabras, en 3-4 frases. Debe:
   - Recoger los hechos y datos concretos del artículo original (cifras, nombres, fechas) sin inventar ni añadir información que no esté en el texto fuente.
   - Tener un tono informativo, cercano y divulgativo: explica cualquier término técnico en cuanto aparece, evita la jerga innecesaria, usa un lenguaje directo, y una frase de apertura que enganche sin caer en el sensacionalismo o el titular clickbait.
   - Ser una reescritura en tus propias palabras, nunca una copia ni una paráfrasis mecánica del texto original.

2. "tono": una etiqueta corta (una frase) que describa el registro aplicado en el resumen — por ejemplo "informativo, cercano y divulgativo", o una variación si el contenido lo pide (más técnico si la noticia es muy especializada, siempre dentro de un registro accesible). Este campo es para trazabilidad interna, no forma parte del texto publicado.

3. "prompt_imagen": un prompt en inglés, de entre 30 y 60 palabras, para un modelo de generación de imágenes (Nano Banana / Gemini 2.5 Flash Image), que describa una escena o composición ilustrativa relacionada con el contenido del resumen. Reglas:
   - Describe una escena, objeto o metáfora visual concreta — no una lista de palabras sueltas.
   - No incluyas texto, logotipos, marcas registradas, ni la reproducción de personas reales identificables (ni sus nombres) en la descripción.
   - Especifica un estilo visual editorial (por ejemplo "editorial illustration", "photorealistic tech photography", "flat design infographic style" — elige el que mejor encaje con la noticia) y evita pedir texto superpuesto en la imagen, porque los modelos de imagen no lo renderizan bien.

Si el contenido del artículo es muy breve o incompleto, genera el mejor resumen posible con la información disponible. Nunca inventes datos, cifras o declaraciones que no estén en el texto fuente.

Devuelve EXCLUSIVAMENTE un objeto JSON válido con esta forma exacta, sin texto adicional antes o después, sin bloques de código markdown:

```json
{
  "resumen": "...",
  "tono": "...",
  "prompt_imagen": "..."
}
```

### Plantilla del mensaje de usuario (por ejecución)

```
Titular: {{ $json.title }}

Fecha de publicación: {{ $json.pubDate }}

Enlace original: {{ $json.link }}

Contenido:

{{ $json.contentSnippet || $json.content }}
```

### Schema para el Structured Output Parser

```json
{
  "type": "object",
  "properties": {
    "resumen": {
      "type": "string",
      "description": "Resumen de la noticia en español, 60-100 palabras, tono informativo, cercano y divulgativo"
    },
    "tono": {
      "type": "string",
      "description": "Etiqueta descriptiva del registro aplicado, para trazabilidad interna"
    },
    "prompt_imagen": {
      "type": "string",
      "description": "Prompt en inglés para el modelo de generación de imagen (Nano Banana), 30-60 palabras"
    }
  },
  "required": ["resumen", "tono", "prompt_imagen"]
}
```

Notas de diseño: el `prompt_imagen` se pide en inglés porque los modelos de generación de imagen (incluido Nano Banana) suelen responder de forma más consistente a descriptores de estilo en inglés, aunque el resto de la pieza sea en español — es una decisión técnica deliberada, no una inconsistencia. Y la instrucción explícita de no reproducir personas reales, logotipos o marcas en la imagen no es solo prudencia editorial: evita generar contenido que pueda infringir derechos de imagen o de marca en una pieza que se publica como si fuera contenido real del medio.

## 8. Valores de configuración conocidos

| Parámetro | Valor |
|---|---|
| URL del feed RSS (Xataka — IA) | https://www.xataka.com/tag/inteligencia-artificial/rss2.xml |
| Telegram — chat_id del grupo destino | -5346073464 |
| Telegram — bot token | Generado vía BotFather, guardado como credencial en n8n (id: `izAUwR5Ijxitz6wW`, nombre "Telegram account") |
| Anthropic — API key | Generada en Claude Console, guardada como credencial en n8n (id: `xm5RKp4qI3NFTtbd`, nombre "Anthropic account") |
| Google Sheets — credencial OAuth2 | Ya existente en n8n, reutilizada de otros flujos (id: `gHAPDOZ8wvBZu0mg`, nombre "Google Sheets account") |
| n8n — API Key para Claude Code | Generada y vigente (uso bloqueado por Cloudflare Access en el hostname público; se usó `docker exec` como alternativa) |
| Nano Banana 2 Lite — API key de Gemini | Generada en Google AI Studio, proyecto "n8n-connection" con facturación activada (la cuota gratuita para generación de imagen es 0), guardada como credencial Header Auth en n8n |
| Google Sheet del log — ID / URL | "Log de Noticias" — ID `18h4I3BsqjcCKRxrsStSF8eHXw2xUN8aMc9Q2RucqKVs`, pestaña gid=0 — [enlace](https://docs.google.com/spreadsheets/d/18h4I3BsqjcCKRxrsStSF8eHXw2xUN8aMc9Q2RucqKVs/edit?gid=0). Cabeceras ya pegadas en la fila 1 (A:J). |
| Repositorio de GitHub | [cesscluvever/AgenciadeNoticias](https://github.com/cesscluvever/AgenciadeNoticias/tree/main) — README.md, workflow/, docs/ y media/ ya montados |
| Credenciales en n8n | Anthropic, Telegram y Google Sheets OAuth2 creadas y verificadas — todas en estado "conectado" |
| Structured Output Parser | Disponible tras activar "Require Specific Output Format" en el nodo AI Agent |
