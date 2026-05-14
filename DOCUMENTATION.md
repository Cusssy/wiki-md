# WordFinder_Twitch - Documentación

## 1. Requisitos Previos

- **Node.js** (v18 o superior)
- **Python** (v3.10 o superior)
- **MongoDB** (local o remota)
- **Qdrant** (base de datos vectorial)
- **LMStudio** (para generación de embeddings)

---

## 2. Configuración del Entorno

### 2.1 LMStudio (Embeddings)

LMStudio se utiliza para generar embeddings vectoriales a partir de transcripciones de clips, permitiendo búsquedas semánticas.

1. Descargar LMStudio desde [https://lmstudio.ai](https://lmstudio.ai)
2. Abrir LMStudio y buscar el modelo `text-embedding-all-minilm-l6-v2-embedding`
3. Cargar el modelo en LMStudio
4. Iniciar el servidor local de inferencia en el puerto **1234** (menú "Local Inference Server" -> Start)
5. Verificar que funciona correctamente con:

```bash
curl http://localhost:1234/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "text-embedding-all-minilm-l6-v2-embedding",
    "input": "Hola mundo"
  }'
```

La respuesta debe incluir un array `data[0].embedding` con 384 valores numéricos.

### 2.2 MongoDB

El proyecto almacena clips, streamers, transcripciones, usuarios y changelogs en MongoDB.

- Por defecto, el proyecto se conecta a una instancia remota en `db.cusssy.com:27017`
- Es posible usar una instancia local modificando la variable `uri` en los archivos:
  - `clientes/backend/services/dbmanager.js`
  - `dashboard/backend/services/dbmanager.js`

Las credenciales de MongoDB se encuentran actualmente hardcodeadas en dichos archivos.

### 2.3 Qdrant (Vector Database)

Qdrant es la base de datos vectorial utilizada para almacenar y consultar embeddings de transcripciones.

**Instalación con Docker:**

```bash
docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant
```

**Instalación local:**

Descargar desde [https://qdrant.tech/documentation/guides/installation/](https://qdrant.tech/documentation/guides/installation/) y ejecutar:

```bash
./qdrant
```

Qdrant corre por defecto en el puerto **6333** (REST API) y **6334** (gRPC).

---

## 3. Ejecución del Proyecto

### 3.1 Backend API (`clientes/backend`)

API pública que sirve las búsquedas de transcripciones (tanto por regex como semánticas).

```bash
cd clientes/backend
npm install
npm start
```

El servidor corre en el puerto **3000**.

Opcionalmente, se puede crear un archivo `.env` en este directorio con las siguientes variables:

```env
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=tu_api_key_si_es_necesaria
```

### 3.2 Dashboard Backend (`dashboard/backend`)

API privada del panel de administración y servidor WebSocket para procesamiento de clips.

```bash
cd dashboard/backend
npm install
npm start
```

El servidor corre en el puerto **15000**.

### 3.3 Frontend Público (`clientes/frontend`)

Aplicación Vue.js para el buscador público de transcripciones.

```bash
cd clientes/frontend
npm install
npm run dev      # Desarrollo (puerto 5173)
npm run build    # Producción
```

### 3.4 Dashboard Frontend (`dashboard/frontend`)

Aplicación Vue.js para el panel de administración.

```bash
cd dashboard/frontend
npm install
npm run dev      # Desarrollo (puerto 5173)
npm run build    # Producción
```

### 3.5 Migración de Embeddings a Qdrant

Para poblar Qdrant con las transcripciones existentes en MongoDB, ejecutar el script de migración:

```bash
cd scripts/migrationVectorialDB
pip install -r requirements.txt   # o instalar dependencias manualmente
python migrate.py
```

Este script:
1. Lee todas las transcripciones desde MongoDB
2. Genera embeddings usando LMStudio (modelo `text-embedding-all-minilm-l6-v2-embedding`)
3. Inserta los vectores en la colección `transcriptions` de Qdrant

---

## 4. API Endpoints

### 4.1 API Pública (`clientes/backend` — Puerto 3000)

| Método | Ruta | Descripción | Parámetros |
|--------|------|-------------|------------|
| GET | `/` | Health check | - |
| GET | `/search/:query` | Búsqueda por regex en transcripciones | `query` (URL param), Headers: `streamer`, `date_gte`, `date_lte` |
| GET | `/streamers` | Listar todos los streamers | - |
| GET | `/stats` | Estadísticas de transcripciones por streamer | - |
| GET | `/changelogs` | Obtener changelogs | - |
| GET | `/vectorsearch/:query` | Búsqueda semántica por embeddings | `query` (URL param) |
| GET | `/loaderio-*` | Verificación Loader.io | - |

---

#### `GET /`

Health check. Devuelve un mensaje de confirmación.

**URL:** `http://localhost:3000/`

**Respuesta:**
```
Hello World!
```

---

#### `GET /search/:query`

Búsqueda por expresión regular en las transcripciones almacenadas en MongoDB. Permite filtrar por streamer y rango de fechas.

**URL:** `http://localhost:3000/search/termino`

**Headers (opcionales):**

| Header | Descripción |
|--------|-------------|
| `streamer` | Filtrar por nombre de streamer |
| `date_gte` | Fecha desde (formato: `YYYY-MM-DD`) |
| `date_lte` | Fecha hasta (formato: `YYYY-MM-DD`) |

**Respuesta:**
```json
[
  {
    "_id": "123456789",
    "streamer": "nombre_streamer",
    "transcription": "texto de la transcripción",
    "date": "2024-01-15T12:30:00Z"
  }
]
```

---

#### `GET /streamers`

Devuelve una lista con los IDs de todos los streamers registrados.

**URL:** `http://localhost:3000/streamers`

**Respuesta:**
```json
["streamer1", "streamer2", "streamer3"]
```

---

#### `GET /stats`

Obtiene estadísticas de cada streamer (información almacenada en la colección `streamers` de MongoDB).

**URL:** `http://localhost:3000/stats`

**Respuesta:**
```json
[
  {
    "_id": "streamer1",
    "twitch_id": 12345678,
    "pfp": "https://url-de-imagen",
    "totalClips": 500,
    "transcribed": 350
  }
]
```

---

#### `GET /changelogs`

Devuelve todos los registros de cambios, ordenados del más reciente al más antiguo.

**URL:** `http://localhost:3000/changelogs`

**Respuesta:**
```json
[
  {
    "_id": "...",
    "title": "Nueva funcionalidad",
    "description": "Descripción del cambio",
    "date": "2024-01-15"
  }
]
```

---

#### `GET /vectorsearch/:query`

Búsqueda semántica utilizando embeddings vectoriales almacenados en Qdrant.

**URL:** `http://localhost:3000/vectorsearch/consulta`

**Respuesta:**
```json
[
  {
    "id": "uuid",
    "score": 0.89,
    "payload": {
      "twitch_id": "123456789",
      "texto": "transcripción del clip",
      "streamer": "nombre_streamer",
      "date": "2024-01-15T12:30:00Z"
    }
  }
]
```

---

#### `GET /loaderio-*`

Endpoint de verificación para Loader.io. La ruta exacta depende del token de verificación configurado.

**URL:** `http://localhost:3000/loaderio-9d3537e9de0853804eb05680dd50ab68`

**Respuesta:**
```
loaderio-9d3537e9de0853804eb05680dd50ab68
```

---

### 4.2 API Dashboard (`dashboard/backend` — Puerto 15000)

| Método | Ruta | Auth | Descripción |
|--------|------|------|-------------|
| GET | `/` | No | Health check |
| GET | `/updateStats` | **JWT** | Actualizar estadísticas de streamers |
| POST | `/start` | **JWT** | Iniciar servidor WebSocket para procesar clips |
| POST | `/stop` | **JWT** | Detener servidor WebSocket |
| POST | `/restart` | **JWT** | Reiniciar servidor WebSocket |
| POST | `/getId` | No | Obtener el ID numérico de Twitch de un streamer |
| GET | `/getstreamers` | No | Obtener todos los streamers con su información |
| POST | `/login` | No | Autenticación de usuario |
| POST | `/addStreamer` | **JWT** | Agregar un nuevo streamer |
| POST | `/checkClips` | **JWT** | Escanear clips de un streamer |
| GET | `/wsstatus` | No | Estado del servidor WebSocket |
| POST | `/verify-token` | No | Verificar validez de un JWT |
| POST | `/addChangeLog` | **JWT** | Agregar un changelog |
| POST | `/deleteChangelog` | **JWT** | Eliminar un changelog |

---

#### `GET /`

Health check del dashboard.

**URL:** `http://localhost:15000/`

**Respuesta:**
```
Dashboard api, what are u doing here?
```

---

#### `GET /updateStats` (Requiere JWT)

Recalcula y actualiza las estadísticas de todos los streamers (`totalClips` y `transcribed`) en la colección `streamers`.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |

**URL:** `http://localhost:15000/updateStats`

**Respuesta:**
```
Data Updated!
```

---

#### `POST /start` (Requiere JWT)

Inicia el servidor WebSocket (puerto 8080) que distribuye clips a los workers para su transcripción.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |
| `id` | Nombre del streamer |
| `month` | `true` para escanear todo el mes actual |

**URL:** `http://localhost:15000/start`

**Respuesta:**
```json
{ "status": "Servidor WebSocket iniciado" }
```

---

#### `POST /stop` (Requiere JWT)

Detiene el servidor WebSocket de transcripción.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |

**URL:** `http://localhost:15000/stop`

**Respuesta:**
```json
{ "status": "Servidor WebSocket detenido" }
```

---

#### `POST /restart` (Requiere JWT)

Reinicia el servidor WebSocket de transcripción.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |
| `id` | Nombre del streamer |

**URL:** `http://localhost:15000/restart`

**Respuesta:**
```json
{ "status": "Servidor WebSocket reiniciado" }
```

---

#### `POST /getId`

Obtiene el ID numérico de Twitch de un streamer almacenado en la base de datos.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `streamer` | Nombre del streamer |

**URL:** `http://localhost:15000/getId`

**Respuesta:**
```
12345678
```

---

#### `GET /getstreamers`

Obtiene la lista completa de streamers con toda su información almacenada.

**URL:** `http://localhost:15000/getstreamers`

**Respuesta:**
```json
[
  {
    "_id": "streamer1",
    "twitch_id": 12345678,
    "last_checked": "2024-01-15T00:00:00Z",
    "last_trans": "2024-01-15T00:00:00Z",
    "pfp": "https://url-de-imagen",
    "totalClips": 500,
    "transcribed": 350
  }
]
```

---

#### `POST /login`

Autentica un usuario y devuelve un token JWT válido por 6 horas.

**Body (JSON):**

```json
{
  "user": "nombre_usuario",
  "password": "contraseña"
}
```

**URL:** `http://localhost:15000/login`

**Respuesta (éxito):**
```
<token_jwt>
```

**Respuesta (error - 401):**
```
Usuario o contraseña incorrectos
```

---

#### `POST /addStreamer` (Requiere JWT)

Agrega un nuevo streamer a la base de datos.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |
| `name` | Nombre del streamer |
| `id` | ID numérico de Twitch |
| `pfp` | URL de la foto de perfil |

**URL:** `http://localhost:15000/addStreamer`

**Respuesta (éxito):**
```
streamer creado
```

**Respuesta (error):**
```
Ha ocurrido un error
```

---

#### `POST /checkClips` (Requiere JWT)

Encola un escaneo de clips para un streamer específico. El escaneo se ejecuta de forma asíncrona.

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |
| `streamer` | Nombre del streamer |

**URL:** `http://localhost:15000/checkClips`

**Respuesta:**
```
revisando clips de X streamer
```

---

#### `GET /wsstatus`

Devuelve el estado actual del servidor WebSocket de transcripción.

**URL:** `http://localhost:15000/wsstatus`

**Respuesta (activo):**
```json
{ "status": 1 }
```

**Respuesta (inactivo):**
```json
{ "status": 0 }
```

---

#### `POST /verify-token`

Verifica si un token JWT es válido.

**Body (JSON):**

```json
{
  "token": "<token_jwt>"
}
```

**URL:** `http://localhost:15000/verify-token`

**Respuesta (válido):**
```json
{ "valid": true }
```

**Respuesta (inválido):**
```json
{ "valid": false }
```

---

#### `POST /addChangeLog` (Requiere JWT)

Agrega una nueva entrada de changelog.

**Body (JSON):**

```json
{
  "data": {
    "title": "Título del cambio",
    "description": "Descripción detallada",
    "date": "2024-01-15"
  }
}
```

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |

**URL:** `http://localhost:15000/addChangeLog`

**Respuesta (éxito):**
```
Changelogs subido
```

---

#### `POST /deleteChangelog` (Requiere JWT)

Elimina un changelog por su ID.

**Body (JSON):**

```json
{
  "id": "id_del_changelog"
}
```

**Headers:**

| Header | Descripción |
|--------|-------------|
| `Authorization` | `Bearer <token_jwt>` |

**URL:** `http://localhost:15000/deleteChangelog`

**Respuesta (éxito):**
```
Changelog borrado
```

---

## 5. Arquitectura

```
                    +------------------+
                    |   LMStudio        |
                    |  (Embeddings)     |
                    |  localhost:1234   |
                    +--------+---------+
                             |
                             v
+----------+       +------------------+       +-----------+
| Usuario  | ----> |  Vercel          | ----> | API       |
| Browser  |       |  (Frontend Vue)  |       | Backend   |
|          |       |                  |       | :3000     |
+----------+       +------------------+       +-----+-----+
                                                     |
                                          +----------+----------+
                                          |                     |
                                    +-----v-----+         +-----v-----+
                                    |  MongoDB   |         |  Qdrant   |
                                    | (Datos)    |         | (Vectores)|
                                    +------------+         +-----------+

+------------------+       +---------------------+
| Dashboard        | ----> | Dashboard Backend   |
| Frontend (Vue)   |       | :15000 (REST API)   |
|                  |       | :8080 (WebSocket)    |
+------------------+       +---------+-----------+
                                      |
                            +---------v-----------+
                            | Python Workers       |
                            | (Whisper/Descarga)   |
                            +---------------------+
```

**Componentes principales:**

1. **Frontend Público** — Aplicación Vue.js desplegada en Vercel. Los usuarios realizan búsquedas de transcripciones a través de esta interfaz.

2. **API Backend** (`clientes/backend`) — API REST en Express que corre en el puerto 3000. Maneja búsquedas por regex (MongoDB) y búsquedas semánticas (Qdrant + LMStudio).

3. **Dashboard Backend** (`dashboard/backend`) — API REST en Express que corre en el puerto 15000. Gestiona streamers, autenticación y changelogs. También aloja un servidor WebSocket en el puerto 8080 para coordinar la transcripción de clips.

4. **Dashboard Frontend** — Aplicación Vue.js para la administración del sistema.

5. **Workers Python** — Scripts en Python que se conectan vía WebSocket al dashboard para recibir clips, descargarlos y transcribirlos usando Whisper de OpenAI.

6. **LMStudio** — Servidor local de embeddings que corre en el puerto 1234. Convierte texto en vectores de 384 dimensiones.

7. **MongoDB** — Base de datos principal que almacena clips, streamers, transcripciones, usuarios y changelogs.

8. **Qdrant** — Base de datos vectorial que almacena los embeddings de las transcripciones para búsquedas semánticas.

---

## 6. Colecciones de MongoDB

| Colección | Descripción |
|-----------|-------------|
| `clips` | Metadatos de los clips de Twitch descargados (ID, streamer, fecha, estado de transcripción) |
| `streamers` | Información de los streamers registrados (ID de Twitch, foto de perfil, estadísticas) |
| `transcriptions` | Transcripciones de los clips generadas por Whisper |
| `users` | Usuarios del dashboard (credenciales hasheadas con bcrypt) |
| `changelogs` | Registro de cambios y novedades del sistema |

---

## 7. Variables de Entorno

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `QDRANT_URL` | URL de conexión a Qdrant | `http://localhost:6333` |
| `QDRANT_API_KEY` | API key opcional para Qdrant | - |

**Nota importante:** Las credenciales de MongoDB se encuentran actualmente hardcodeadas en los archivos `clientes/backend/services/dbmanager.js` y `dashboard/backend/services/dbmanager.js`. No existe una variable de entorno para configurarlas. Se recomienda migrar a variables de entorno para evitar exponer credenciales en el código fuente.
