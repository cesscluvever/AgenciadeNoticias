# Sticky notes del canvas — News Reporter

Transcripción de las notas adhesivas colocadas en el canvas de n8n, agrupadas por la zona del flujo a la que acompañan (ver `workflow/news-reporter.json`).

## Disparo

Dos entradas en paralelo hacia el mismo flujo:
- **Schedule Trigger**: cron diario (08:00) para el caso de uso real.
- **Manual Trigger**: para ejecutar bajo demanda en la demo/vídeo sin esperar al cron.

## Ingesta

**RSS Feed Read**: lee el feed de Xataka - Inteligencia Artificial (ya pre-filtrado por tema, no hace falta nodo Filter).

**Limit**: el RSS devuelve todos los items en cada ejecución; este nodo se queda solo con el más reciente (1 item, "first items").

## Deduplicación

**Buscar en el Log**: busca en la Google Sheet "Log de Noticias" una fila cuyo link coincida con el artículo actual. "Always Output Data" activado para que, si no hay coincidencia, el IF reciba igualmente un item (vacío) y pueda decidir.

**¿Ya procesada?**: si el link ya existe en el log, el flujo termina (rama verdadera → Fin) sin gastar llamadas de pago (Claude, Nano Banana). Si no existe, continúa.

## Fin - Ya procesada

Nodo No Operation: marca visualmente el punto donde termina la rama de artículos ya procesados. No hace nada más.

## Procesamiento IA

**AI Agent** (Anthropic Chat Model + Structured Output Parser, "Require Specific Output Format" activado): genera resumen, tono y prompt_imagen en un único JSON estructurado, a partir del titular/fecha/enlace/contenido del artículo.

## Generación de imagen

**Nano Banana - Generar Imagen**: llama a la API de Gemini 2.5 Flash Image con el prompt_imagen generado.

**Decodificar Imagen a Binario**: Gemini devuelve la imagen en base64 dentro del JSON; este Code node la convierte en un binario real de n8n para poder adjuntarla, no solo enlazarla.

## Salida y registro

**Telegram - Enviar Foto**: envía la imagen generada como foto real (no enlace) al grupo dedicado, con el resumen como caption.

**Preparar Fila del Log** + **Google Sheets - Añadir al Log**: tras el envío correcto, construye y añade la fila con las 10 columnas del log (link, título, fechas, resumen, tono, prompt_imagen, imagen, estado, canal_destino).

## Gestor de errores (workflow secundario)

Workflow secundario enganchado como *Error Workflow* del flujo principal "News Reporter" (Settings → Error Workflow). Cuando cualquier nodo del flujo principal falla, n8n dispara automáticamente el Error Trigger de este workflow y se envía una notificación por Telegram con el nombre del workflow, el nodo que falló y el mensaje de error.
