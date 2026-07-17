# Crónica diaria de fútbol europeo (n8n + Gemini API)

Workflow de n8n que genera cada día una crónica periodística en español sobre las noticias
más relevantes de **Premier League, Serie A, Bundesliga y Ligue 1**, excluyendo
deliberadamente La Liga española. Combina fuentes RSS reales con la API de Gemini para
sintetizar y redactar el resumen final.

## Arquitectura

```
Trigger diario (cron 8:00)
        │
        ├──► RSS Premier League ──┐
        │                          ├──► Unir feeds ──► Filtrar y limitar ──► Construir prompt
        └──► RSS Fútbol Europeo ──┘                                                │
                                                                                    ▼
                                                                        Llamar a Gemini API
                                                                                    │
                                                                                    ▼
                                                                          Extraer texto
                                                                                    │
                                                                                    ▼
                                                                        Guardar como .md
                                                                                    │
                                                                                    ▼
                                                                        Subir a GitHub
```

El último nodo ("Subir a GitHub") crea un commit automático en el propio repo cada vez que
se ejecuta el workflow, usando el nodo nativo de GitHub de n8n (API REST, no requiere `git`
instalado ni configurado en la máquina que corre n8n).

**Fuentes RSS usadas** (verificadas, BBC Sport):
- Premier League: `https://feeds.bbci.co.uk/sport/football/premier-league/rss.xml`
- Fútbol Europeo (cubre Serie A, Bundesliga, Ligue 1, etc.): `https://feeds.bbci.co.uk/sport/football/european/rss.xml`

El nodo "Filtrar y limitar" añade una capa extra de seguridad: descarta cualquier noticia
que mencione La Liga o clubes españoles (por si el feed europeo la incluyera puntualmente),
elimina duplicados y se queda con las 12 noticias más recientes.

## Importar el workflow

1. Abre n8n (local o self-hosted).
2. Menú superior derecho → **Import from File** (o `Ctrl+O` según versión) → selecciona `workflow.json`.
3. Revisa que los 9 nodos y sus conexiones se han cargado correctamente (compáralo con el diagrama de arriba).

> Nota: dependiendo de tu versión exacta de n8n, algún nodo (`merge`, `convertToFile`) puede
> pedirte reconfigurar un parámetro menor al abrirlo por primera vez, ya que estos nodos han
> tenido cambios de versión recientes. Si algo no carga bien, en la sección "Configuración manual"
> más abajo tienes cada nodo detallado para recrearlo a mano en 5 minutos.

## Configurar la credencial de Gemini API

El nodo **"Llamar a Gemini API"** usa autenticación por header (`x-goog-api-key`), no un nodo
nativo de Google, para que el ejemplo sea explícito y fácil de defender en una entrevista.

1. En n8n, ve a **Credentials → New → Header Auth**.
2. Nombre del header: `x-goog-api-key`
3. Valor: tu API key de Google (la generas en [console.cloud.google.com](https://console.cloud.google.com))
4. Guarda la credencial con el nombre `Gemini API Key (x-goog-api-key)`.
5. Abre el nodo "Llamar a Gemini API" en el workflow y selecciona esa credencial en el campo de autenticación (al importar, queda con un placeholder que debes reasignar).

**Sobre el modelo:** el body de la petición usa `"model": "gemini-2.0-flash-exp"`. Antes de tu
entrevista, verifica en la [documentación de Google](https://ai.google.dev/gemini-api/docs/models/gemini)
cuál es el identificador de modelo vigente, por si ha cambiado.

## Configurar la subida automática a GitHub

1. Genera un **fine-grained personal access token** en GitHub (Settings → Developer settings
   → Personal access tokens), con acceso limitado solo a este repositorio y permiso
   **Contents: Read and write**.
2. En n8n, crea una credencial **GitHub API** (tipo Access Token) con ese token.
3. En el nodo "Subir a GitHub", asigna esa credencial y reemplaza `TU_USUARIO_GITHUB` por tu
   usuario real de GitHub.
4. La ruta de destino es `examples/cronica_futbol_<fecha>.md` — asegúrate de que la carpeta
   `examples/` exista en el repo (aunque sea con un `.gitkeep`) antes de la primera ejecución.

**Nota:** la operación `Create` del nodo falla si el fichero ya existe (por ejemplo, si
ejecutas el workflow dos veces el mismo día durante pruebas). Para depurar, borra el fichero
de prueba entre ejecuciones o añade temporalmente la hora al nombre del fichero.

## Probar el workflow

No hace falta esperar al cron: abre el workflow y pulsa **"Test workflow"** — se ejecuta
inmediatamente de principio a fin independientemente del trigger programado.

Revisa nodo a nodo el resultado (n8n te deja inspeccionar el output de cada paso):
1. Los dos RSS deben devolver ítems reales de noticias recientes.
2. "Filtrar y limitar" debe devolver máximo 12 ítems, ninguno con contenido español.
3. "Construir prompt" debe mostrar el prompt final que se envía a Gemini.
4. "Llamar a Gemini API" debe devolver un 200 con el objeto de respuesta de Google.
5. "Guardar como .md" genera el fichero final descargable desde la ejecución.

## Configuración manual (si el import falla)

| Nodo | Tipo | Parámetro clave |
|---|---|---|
| Trigger diario | Schedule Trigger | Cron: `0 8 * * *` |
| RSS Premier League | RSS Feed Read | URL de arriba |
| RSS Fútbol Europeo | RSS Feed Read | URL de arriba |
| Unir feeds | Merge | Mode: Combine / Combine All |
| Filtrar y limitar | Code | Ver `jsCode` en `workflow.json` |
| Construir prompt | Code | Ver `jsCode` en `workflow.json` |
| Llamar a Gemini API | HTTP Request | POST a `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent`, headers `x-goog-api-key: TU_API_KEY`, body JSON con `contents` y `generationConfig` |
| Extraer texto | Code | Ver `jsCode` en `workflow.json` |
| Guardar como .md | Convert to File | Operation: To Text, source: `cronica` |
| Subir a GitHub | GitHub | Resource: File, Operation: Create, Path: `examples/cronica_futbol_<fecha>.md`, Binary Data: ON |

