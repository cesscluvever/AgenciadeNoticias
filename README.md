# AgenciadeNoticias — News Reporter

Agente n8n que vigila el feed de IA de Xataka, genera con Claude un resumen editorial + una imagen (Nano Banana / Gemini) y lo publica en Telegram, manteniendo un log en Google Sheets para no repetir noticias. Caso práctico del máster (EBIS).

## Qué hace el flujo

1. **Disparo**: Schedule Trigger (diario, 08:00) y Manual Trigger en paralelo, ambos hacia el mismo pipeline.
2. **Ingesta**: RSS Feed Read sobre el feed de Xataka — Inteligencia Artificial, seguido de un Limit a 1 ítem (el más reciente).
3. **Deduplicación**: Google Sheets "Get Row(s)" comprueba si el link del artículo ya está en el log ("Log de Noticias"). Si ya existe, el flujo termina ahí sin gastar llamadas de pago; si no, continúa.
4. **Procesamiento IA**: un AI Agent (Anthropic Chat Model + Structured Output Parser, "Require Specific Output Format" activado) convierte el artículo en un JSON con `resumen`, `tono` y `prompt_imagen`.
5. **Generación de imagen**: HTTP Request a la API de Gemini (Nano Banana 2 Lite, `gemini-3.1-flash-lite-image`) con el `prompt_imagen`; un Code node decodifica la imagen (viene en base64 dentro del JSON) a un binario real de n8n.
6. **Salida**: Telegram — Send Photo, con la imagen como adjunto real (no un enlace) y el resumen como caption.
7. **Registro**: tras el envío correcto, se añade una fila al log de Google Sheets con las 10 columnas (link, título, fechas, resumen, tono, prompt_imagen, imagen, estado, canal_destino).
8. **Gestión de errores**: un workflow secundario (`workflow/news-reporter-errors.json`) con un Error Trigger, enganchado como *Error Workflow* del flujo principal, notifica por Telegram el nombre del workflow, el nodo que falló y el mensaje de error.

Cada nodo del canvas tiene una sticky note explicando su función — ver la transcripción completa en [`docs/sticky-notes.md`](docs/sticky-notes.md).

## Decisiones clave

- **Fuente de noticias**: RSS de Xataka — Inteligencia Artificial, ya pre-filtrado por tema, sin necesidad de un nodo Filter adicional.
- **LLM de procesamiento**: Claude (Anthropic API vía Claude Console, no la suscripción Pro), con Structured Output Parser para forzar el JSON de salida.
- **Generación de imagen**: Nano Banana 2 Lite (`gemini-3.1-flash-lite-image`, el reemplazo recomendado por Google del `gemini-2.5-flash-image` original) vía HTTP Request, ya que no existe nodo nativo en n8n. **Importante**: la cuota gratuita de Gemini para *generación de imagen* es 0 sin facturación vinculada al proyecto de Google Cloud — hubo que activarla para que el flujo funcionase (coste real: céntimos de dólar).
- **Canal de salida**: Telegram con un bot y grupo dedicados (no un DM personal), para simular el canal de distribución de un equipo editorial.
- **Deduplicación y log**: Google Sheets, filtrando por columna `link` antes de gastar ninguna llamada de pago — así un artículo ya procesado no vuelve a consumir tokens de Claude ni cuota de Gemini.
- **Orquestación**: n8n autoalojado en Docker, ya en marcha (Cloudflare Tunnel + Access). El flujo se construyó con la CLI de n8n vía `docker exec` en lugar de su API REST pública, porque Cloudflare Access protege todo `n8n.cesscluv.com` salvo `/webhook*` y bloquea las llamadas REST externas incluso con API key.
- **Gestor de errores**: workflow secundario con Error Trigger, mismo canal de salida (Telegram), sin añadir complejidad al flujo principal.

El razonamiento completo detrás de cada decisión (incluidas las alternativas descartadas) está en [`docs/arquitectura-y-decisiones.md`](docs/arquitectura-y-decisiones.md).

## Estructura del repositorio

```
AgenciadeNoticias/
├── README.md
├── workflow/
│   ├── news-reporter.json          # flujo principal, exportado de n8n
│   └── news-reporter-errors.json   # workflow secundario de gestión de errores
├── docs/
│   ├── arquitectura-y-decisiones.md
│   └── sticky-notes.md
└── media/
    └── demo-video                  # enlace al vídeo de la demo
```

## Replicarlo

1. Crear las credenciales en n8n: Anthropic, Telegram (bot vía BotFather) y Google Sheets OAuth2.
2. Crear una credencial Header Auth para Gemini (`x-goog-api-key`), con facturación activada en el proyecto de Google Cloud.
3. Crear la Google Sheet "Log de Noticias" con las 10 columnas ya pegadas en la fila 1.
4. Importar `workflow/news-reporter-errors.json` primero, y luego `workflow/news-reporter.json` (referencia al workflow de errores por ID en Settings → Error Workflow — revisar y volver a enlazar si el ID cambia al importar).
5. Ajustar las referencias de credenciales de cada nodo a las credenciales recién creadas.
6. Activar ambos workflows.
