# AgenciadeNoticias — News Reporter

Agente n8n que vigila el feed de IA de Xataka, genera con Claude un resumen editorial + una imagen (Nano Banana / Gemini) y lo publica en Telegram, manteniendo un log en Google Sheets para no repetir noticias. Caso práctico del máster (EBIS).

## Qué hace el flujo

1. **Disparo**: Schedule Trigger (diario, 08:00) y Manual Trigger en paralelo, ambos hacia el mismo pipeline.
2. **Ingesta**: RSS Feed Read sobre el feed de Xataka — Inteligencia Artificial, seguido de un Limit a 6 ítems como tope de seguridad (el feed puede devolver ~20 items en cada ejecución).
3. **Deduplicación (multi-artículo)**: "Leer Log Completo" lee toda la hoja "Log de Noticias", ubicado en un drive de Google, una vez por ejecución; un Code node ("Filtrar Noticias Nuevas") cruza en bloque los hasta 6 candidatos contra los links ya registrados y deja pasar solo los totalmente nuevos — soporta procesar varios artículos nuevos en la misma ejecución (p. ej. si se publicó más de uno desde la última vez), no solo el más reciente.
4. **Procesamiento IA**: un AI Agent (Anthropic Chat Model + Structured Output Parser, "Require Specific Output Format" activado) convierte cada artículo en un JSON con `resumen`, `tono` y `prompt_imagen`.
5. **Generación de imagen**: HTTP Request a la API de Gemini (Nano Banana 2 Lite, `gemini-3.1-flash-lite-image`) con el `prompt_imagen`; un Code node decodifica la imagen (almacenado en base64 dentro del JSON) a un binario real de n8n.
6. **Salida**: Telegram — Send Photo, con la imagen como adjunto real (no un enlace) y el resumen como caption. Con reintentos automáticos activados, porque subir varias fotos en la misma ejecución puede tener algún corte de conexión transitorio.
7. **Registro**: tras el envío correcto, se añade una fila al log de Google Sheets con las 10 columnas (link, título, fechas, resumen, tono, prompt_imagen, imagen, estado, canal_destino) para cada noticia reportada.
8. **Gestión de errores**: un workflow secundario (`workflow/news-reporter-errors.json`) con un Error Trigger, enganchado como *Error Workflow* del flujo principal, notifica por Telegram el nombre del workflow, el nodo que falló y el mensaje de error.

Cada nodo del canvas tiene una sticky note explicando su función — ver la transcripción completa en [`docs/sticky-notes.md`](docs/sticky-notes.md).

## Decisiones clave

- **Fuente de noticias**: RSS de Xataka — Inteligencia Artificial, ya pre-filtrado por tema, sin necesidad de un nodo Filter adicional.
- **LLM de procesamiento**: Claude (Anthropic API vía Claude Console, no la suscripción Pro), con Structured Output Parser para forzar el JSON de salida.
- **Generación de imagen**: Nano Banana 2 Lite (con modelo`gemini-3.1-flash-lite-image`, el reemplazo recomendado por Google del `gemini-2.5-flash-image` original) vía HTTP Request, ya que no existe nodo nativo en n8n. **Importante**: al tener problemas con la cuota gratuita de Gemini para *generación de imagen*  (0 sin facturación vinculada al proyecto de Google Cloud) — hubo que activar un crédito (20$) para que este flujo y sucesivos funcionase (coste real: céntimos de dólar por imagen).
- **Canal de salida**: Telegram con un bot y grupo dedicados (no un DM personal), para simular el canal de distribución de un equipo editorial. El bot publica en el canal Telegram de Grupo al que pertenece (Agencia Noticias IA).
- **Deduplicación y log**: Google Sheets, cruzando el log completo contra los candidatos del RSS antes de realizar llamada de pago — así un artículo ya procesado no vuelve a consumir tokens de Claude ni cuota de Gemini.
- **Orquestación**: n8n autoalojado en Docker de mi PC, publicado y accesible a través de Cloudflare Tunnel, con control de acceso, siendo la URL: `n8n.cesscluv.com`. Cloudflare Access protege todo el dominio excepto `/webhook*` y bloquea las llamadas REST externas incluso con API key. Por ello se ha desarrollado el flujo con Claude Code a través del CLI de n8n, en lugar de acceder al dominio por REST API.
- **Gestor de errores**: workflow secundario con Error Trigger, mismo canal de salida (Telegram), sin añadir complejidad al flujo principal.

En el documento `docs/arquitectura-y-decisiones.md`se recogen las decisiones de arquitectura y sus justificaciones. Se añaden, así mismo alternativas descartadas, para mayor claridad. 

## Estructura del repositorio

```
AgenciadeNoticias/
├── README.md
├── workflow/
│   ├── news-reporter.json          # flujo principal, exportado de n8n
│   └── news-reporter-errors.json   # subflujo secundario de gestión de errores
├── docs/
│   ├── arquitectura-y-decisiones.md   #Decisiones de arquitectura y justificaciones
│   └── sticky-notes.md                # Copia de las sticky notes incluidas en el n8n. Sirve como referencia de la estructura del flujo
└── media/
    └── demo-video                  # enlace al vídeo de la demo
```

## Replicarlo

1. Crear las credenciales en n8n: Anthropic, Telegram (bot vía BotFather) y Google Sheets OAuth2.
2. Crear una credencial Header Auth para Gemini (`x-goog-api-key`), con facturación activada en el proyecto de Google Cloud.
3. Crear la Google Sheet "Log de Noticias" con las 10 columnas ya pegadas en la fila 1.
4. Importar `workflow/news-reporter-errors.json` primero, y luego `workflow/news-reporter.json` (referencia al workflow de errores por ID en Settings → Error Workflow — revisar y volver a enlazar si el ID cambia al importar).
5. Ajustar las referencias de credenciales de cada nodo a las credenciales recién creadas.
6. Apuntar al fichero "Log de Noticias" en los nodos del flujo correspondientes.
7. Activar/Publicar ambos workflows.
