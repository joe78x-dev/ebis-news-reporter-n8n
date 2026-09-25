# 📰 News Reporter – Agente de generación de contenidos (n8n)

**Autor:** Johanny Espinoza · Máster EBIS – Hiperautomatización

Flujo de n8n que transforma noticias reales de tecnología en piezas listas para distribuir: obtiene la noticia, la reescribe con IA, genera una ilustración y la publica en Telegram.

## Archivos

| Archivo | Descripción |
|---|---|
| `News_Reporter_johannyespinoza.json` | Workflow principal (08 News Reporter) |
| `Alertas_de_error_johannyespinoza.json` | Workflow de gestión de errores (09 Alertas de error) |

## 1. Qué hace el flujo

1. **Disparo:** cada día a las 8:00 (America/Costa_Rica) o manualmente.
2. **Obtención:** HTTP Request al RSS de Tecnología de EL PAÍS y conversión de XML a JSON.
3. **Selección:** un nodo Code filtra solo noticias de la sección Tecnología, excluye contenido patrocinado, ordena por fecha, elige la más reciente y limpia el HTML.
4. **Redacción con IA:** un AI Agent (gpt-4o-mini) genera titular, resumen en tono informativo, cercano y divulgativo, "por qué importa", hashtags y el prompt de la imagen, en JSON estructurado.
5. **Imagen:** OpenAI gpt-image-1-mini crea una ilustración editorial de 1024x1024 a partir de ese prompt.
6. **Publicación en Telegram:** se envía la imagen con titular y hashtags; en paralelo, la imagen se aloja en ImgBB y se envía el resumen con el enlace público de la imagen y la fuente original.

## 2. Decisiones clave

- **Dos triggers (Schedule + Manual):** producción automática diaria y ejecución bajo demanda para pruebas, sobre la misma lógica.
- **HTTP Request + XML en lugar del nodo RSS Read:** cumple el requisito de usar HTTP Request y da control total sobre la respuesta.
- **Filtro en código:** el feed incluye noticias de otras secciones, contenido patrocinado (`/branded/`) y no viene ordenado por fecha.
- **gpt-4o-mini con temperatura 0.5:** rápido y económico para una tarea diaria; temperatura media para adaptar el tono sin inventar datos.
- **System prompt con reglas explícitas:** no inventar datos, tono definido, límites de longitud.
- **Prompt de imagen en inglés, sin texto, logos ni personas reales:** los modelos de imagen rinden mejor en inglés, y las políticas de OpenAI rechazan generar figuras públicas identificables.
- **Esquema de salida plano (solo textos):** con listas en el esquema, gpt-4o-mini cerraba mal el JSON y el parser fallaba. Con todos los campos como texto, el formato es estable.
- **gpt-image-1-mini:** DALL·E 3 ya no estaba disponible; este modelo equilibra calidad, coste y velocidad.
- **Imagen enviada como archivo, enlace vía ImgBB:** los modelos gpt-image devuelven binario, no URL. Enviar la foto a Telegram por URL de ImgBB fallaba de forma intermitente, así que la foto se envía como archivo y ImgBB se usa solo para obtener el enlace público.
- **API key de ImgBB como credencial (Query Auth):** no queda expuesta en la URL ni en el JSON exportado.
- **Dos mensajes en Telegram:** el pie de foto admite máximo 1024 caracteres.

## 3. Gestión de errores

El flujo tiene cuatro capas:

1. **Reintentos automáticos (Retry On Fail):** RSS (3 intentos), generación de imagen (2) e ImgBB (3), para absorber fallos transitorios de red o de las APIs.
2. **Auto-Fix Format en el parser:** si la respuesta del modelo no cumple el esquema JSON, n8n se la devuelve al modelo para que la corrija.
3. **Fallback sin imagen:** los nodos Generar imagen y Subir imagen ImgBB usan *On Error → Continue (using error output)*. Si fallan tras los reintentos, la rama de error envía la noticia completa como texto con el aviso "Imagen no disponible". La noticia nunca se pierde. *Probado forzando una URL inválida en ImgBB.*
4. **Workflow de errores (09 Alertas de error):** configurado como Error Workflow del flujo principal. Si cualquier nodo falla en una ejecución de producción, un Error Trigger envía a Telegram el nombre del workflow, el nodo que falló, el mensaje de error, la hora y el enlace a la ejecución.

Además, el nodo Code lanza un error controlado si el feed no contiene noticias válidas, que es capturado por la capa 4.

## 4. Cómo replicarlo

1. Importar ambos JSON en n8n (probado en n8n 2.23.4, self-hosted).
2. Crear las credenciales:
   - **OpenAI API** (modelo de texto e imagen).
   - **Telegram API** con el token de un bot creado en @BotFather.
   - **ImgBB:** credencial *Query Auth* con Name `key` y Value = API key de api.imgbb.com.
3. Sustituir el Chat ID de Telegram por el propio (enviar `/start` al bot antes).
4. Publicar primero `09 Alertas de error` y seleccionarlo como Error Workflow en los ajustes de `08 News Reporter`.
5. Publicar `08 News Reporter`.

## 5. Limitaciones conocidas y mejoras futuras

- Algunas piezas comerciales se publican con URL de noticia normal y no son detectadas por el filtro `/branded/`. Mejora: que el agente clasifique si la pieza es publicitaria y la descarte.
- Añadir memoria de noticias ya publicadas (por ejemplo, en Google Sheets) para evitar duplicados si el feed no se actualiza entre ejecuciones.
- Ampliar a varias fuentes RSS (innovación y negocios) y elegir la noticia más relevante con IA.
