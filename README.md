# n8n Local AI — RAG con Ollama y Qdrant

Stack local para un asistente RAG (Retrieval-Augmented Generation): ingesta documentos PDF, los indexa como embeddings en una base vectorial y responde preguntas por chat usando solo esa información, con modelos corriendo localmente vía Ollama.

## Arquitectura

| Servicio  | Imagen                    | Rol                                                              | Puerto |
|-----------|---------------------------|-------------------------------------------------------------------|--------|
| `postgres`| `postgres:16`             | Base de datos de n8n (workflows, credenciales, ejecuciones)       | interno |
| `qdrant`  | `qdrant/qdrant:latest`    | Base de datos vectorial (almacena los embeddings de los documentos) | 6333   |
| `ollama`  | `ollama/ollama:latest`    | Sirve los modelos LLM y de embeddings localmente                 | 11434  |
| `n8n`     | `n8nio/n8n:latest`        | Motor de automatización donde vive el workflow RAG                | 5678   |

`n8n` depende de que `postgres` esté saludable y de que `qdrant`/`ollama` hayan iniciado (`depends_on` en `docker-compose.yaml`).

Los PDFs a indexar se colocan en `./documents`, montado dentro del contenedor `n8n` en `/files` (con `N8N_RESTRICT_FILE_ACCESS_TO: /files` para restringir el acceso del nodo de lectura de archivos a esa carpeta).

## Flujo de trabajo (workflow `config.json`)

El workflow de n8n tiene dos partes independientes, marcadas con sticky notes en el propio canvas:

### 1. Ingesta de documentos (rama superior)

```
Iniciar Ingesta (trigger manual)
  → Leer PDFs desde /files       (lee todos los *.pdf de /files)
  → Extraer texto del PDF        (extractFromFile, operación "pdf")
  → Qdrant Vector Store (insert) → colección "knowledge_base"
```

El nodo `Qdrant Vector Store` recibe dos entradas auxiliares:
- **Recursive Character Text Splitter** → **Default Data Loader**: parte el texto en chunks de 1000 caracteres con 150 de solapamiento.
- **Embeddings Ollama** (modelo `nomic-embed-text`): genera el vector de cada chunk antes de insertarlo en Qdrant.

Este flujo se dispara manualmente cada vez que se agregan documentos nuevos a `./documents`.

### 2. Chat / Recuperación y generación (rama inferior)

```
When chat message received (chat trigger)
  → AI Agent
       ├─ Ollama Chat Model (nemotron-3-nano:30b-cloud) → LLM de respuesta
       ├─ Simple Memory (buffer window) → memoria de la conversación
       └─ Qdrant Knowledge Base (retrieve-as-tool) → herramienta de búsqueda
              └─ Embeddings Ollama Query (nomic-embed-text) → embeddings de la consulta
```

El `AI Agent` usa el modelo de Ollama para razonar y decide cuándo invocar la herramienta `Qdrant Knowledge Base`, que busca los `topK=5` chunks más relevantes en la colección `knowledge_base` (misma colección que llenó la ingesta).

El comportamiento del agente está definido por su `systemMessage` (ver también `prompt.txt`, que contiene la misma versión del prompt para edición/control de versiones fuera de n8n):

- Responder solo con información recuperada de los documentos.
- No inventar información ni mezclarla con conocimiento general del modelo.
- Si no encuentra la respuesta en los documentos, decir explícitamente que no la encontró.
- Responder siempre en español, citando el documento fuente cuando sea posible.
- Invocar la herramienta de búsqueda con el formato `{"input": "<consulta>"}`.

## Puesta en marcha

1. Configurar variables de entorno en `.env` (usuario/clave de Postgres, encryption key de n8n, host/puerto, timezone).
2. Levantar el stack:
   ```bash
   docker compose up -d
   ```
3. Abrir n8n en `http://localhost:5678` e importar/abrir el workflow (`config.json`).
4. Crear las credenciales que el workflow referencia (actualmente con placeholders en `config.json`):
   - **Qdrant account** (`qdrantApi`) → apunta a `http://qdrant:6333`.
   - **Ollama account** (`ollamaApi`) → apunta a `http://ollama:11434`.
5. Descargar en el contenedor de Ollama los modelos usados por el workflow:
   ```bash
   docker exec -it n8n-ollama ollama pull nomic-embed-text
   docker exec -it n8n-ollama ollama pull nemotron-3-nano:30b-cloud
   ```
6. Colocar los PDFs a indexar en `./documents`.
7. Ejecutar manualmente el nodo **Iniciar Ingesta** para poblar Qdrant.
8. Usar el chat del workflow (**When chat message received**) para hacer preguntas sobre los documentos indexados.

## Notas

- La colección de Qdrant se llama `knowledge_base` y es compartida entre la rama de ingesta (escritura) y la rama de chat (lectura).
- Reindexar documentos no borra los anteriores: cada ejecución de **Iniciar Ingesta** inserta nuevos vectores; no hay borrado/actualización de la colección.
- Los datos persisten entre reinicios gracias a los volúmenes de Docker: `postgres_data`, `qdrant_data`, `ollama_data`, `n8n_data`.
