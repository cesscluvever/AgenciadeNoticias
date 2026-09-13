# Sticky notes del canvas — News Reporter

Transcripción de las notas adhesivas colocadas en el canvas de n8n, agrupadas por la zona del flujo a la que acompañan (ver `workflow/news-reporter.json`).

## Disparo

Dos entradas en paralelo hacia el mismo flujo:
- **Schedule Trigger**: cron diario (08:00) para el caso de uso real.
- **Manual Trigger**: para ejecutar bajo demanda en la demo/vídeo sin esperar al cron.

## Ingesta

**RSS Feed Read**: lee el feed de Xataka - Inteligencia Artificial (ya pre-filtrado por tema, no hace falta nodo Filter).

**Limit a 6 (más recientes)**: el RSS devuelve todos los items en cada ejecución (~20); este nodo acota a un máximo de 6 candidatos por ejecución — un tope de seguridad para que un día con mucha actividad no dispare de golpe 20 llamadas de pago, sin limitarse a procesar solo 1 artículo nuevo por ejecución.

## Deduplicación (soporta varios artículos nuevos por ejecución)

**Leer Log Completo**: lee todas las filas de la Google Sheet "Log de Noticias" una sola vez por ejecución (sin filtro), en paralelo a la ingesta del RSS.

**Filtrar Noticias Nuevas**: Code node que cruza los hasta 6 candidatos del RSS contra los links ya existentes en el log y deja pasar solo los que aún no se han procesado, sin gastar llamadas de pago (Claude, Nano Banana) en los que ya están. Si los 6 ya están procesados, no pasa ningún item y el flujo termina ahí de forma natural — sin necesidad de un IF ni un nodo de fin explícito, porque un Code node que filtra hasta dejar 0 items ya detiene la rama por sí mismo.

*(Diseño anterior, sustituido): un Get Row(s) + IF por artículo funcionaba con 1 candidato, pero con varios en paralelo perdía en silencio los que no coincidían con el log — de ahí el cambio a leer el log completo una vez y filtrar en bloque.)*

## Procesamiento IA

**AI Agent** (Anthropic Chat Model + Structured Output Parser, "Require Specific Output Format" activado): genera resumen, tono y prompt_imagen en un único JSON estructurado, a partir del titular/fecha/enlace/contenido del artículo.

## Generación de imagen

**Nano Banana - Generar Imagen**: llama a la API de Gemini 2.5 Flash Image con el prompt_imagen generado. Gemini devuelve la imagen en base64 (formato texto) al nodo siguiente dentro del JSON.

**Decodificar Imagen a Binario**: Este Code node recibe la imagen en base64 y la convierte en un binario real de n8n para poder adjuntarla. De esta manera podrá ser enviada como formato png a Telegram y, por tanto, como imagen real, junto al texto del resumen.

## Salida y registro

**Telegram - Enviar Foto**: envía la imagen generada como foto real (no enlace) al grupo dedicado, con el resumen como caption.

**Preparar Fila del Log** + **Google Sheets - Añadir al Log**: tras el envío correcto, construye y añade la fila con las 10 columnas del log (link, título, fechas, resumen, tono, prompt_imagen, imagen, estado, canal_destino).

## Gestor de errores (workflow secundario)

Workflow secundario enganchado como *Error Workflow* del flujo principal "News Reporter" (Settings → Error Workflow). Cuando cualquier nodo del flujo principal falla, n8n dispara automáticamente el Error Trigger de este workflow y se envía una notificación por Telegram con el nombre del workflow, el nodo que falló y el mensaje de error.
