# Chuleta SimpleWiki — Documentación Completa

> Proyecto: Servidor web minimalista para visualizar documentación Markdown desde un repositorio GitHub.
> Stack: Express (backend) + Vue 3 (frontend) + Markdown rendering.

---

## Índice

1. [Arquitectura General](#1-arquitectura-general)
2. [Backend — Express + Node.js](#2-backend--express--nodejs)
   - [2.1 Estructura del proyecto](#21-estructura-del-proyecto)
   - [2.2 Express setup (index.ts)](#22-express-setup-indexts)
   - [2.3 Servicio de sincronización (sync.service.ts)](#23-servicio-de-sincronización-syncservicets)
   - [2.4 Servicio de Markdown (markdown.service.ts)](#24-servicio-de-markdown-markdownservicets)
   - [2.5 Rutas de Markdown (markdown.routes.ts)](#25-rutas-de-markdown-markdownroutests)
   - [2.6 package.json — scripts y dependencias](#26-packagejson--scripts-y-dependencias)
   - [2.7 tsconfig.json — configuración TypeScript](#27-tsconfigjson--configuración-typescript)
   - [2.8 Variables de entorno (.env)](#28-variables-de-entorno-env)
3. [Frontend — Comunicación entre Componentes](#3-frontend--comunicación-entre-componentes)
   - [3.1 Props (padre → hijo)](#31-props-padre--hijo)
   - [3.2 Emits (hijo → padre)](#32-emits-hijo--padre)
   - [3.3 Flujo completo en SimpleWiki](#33-flujo-completo-en-simplewiki)
   - [3.4 watch: reaccionar a cambios en Props](#34-watch-reaccionar-a-cambios-en-props)
   - [3.5 Sintaxis de templates explicada](#35-sintaxis-de-templates-explicada)
4. [Frontend — Renderizado de Markdown](#4-frontend--renderizado-de-markdown)
   - [4.1 marked — conversor de MD a HTML](#41-marked--conversor-de-md-a-html)
   - [4.2 v-html — directiva de Vue para HTML crudo](#42-v-html--directiva-de-vue-para-html-crudo)
   - [4.3 highlight.js — coloreado de sintaxis](#43-highlightjs--coloreado-de-sintaxis)
   - [4.4 marked-highlight — puente entre marked y highlight.js](#44-marked-highlight--puente-entre-marked-y-highlightjs)
   - [4.5 github-markdown-css — estilo visual](#45-github-markdown-css--estilo-visual)
   - [4.6 Pipeline completo de renderizado](#46-pipeline-completo-de-renderizado)
5. [Conceptos Clave Explicados](#5-conceptos-clave-explicados)
   - [5.1 TypeScript básico](#51-typescript-básico)
   - [5.2 Módulos nativos de Node](#52-módulos-nativos-de-node)
   - [5.3 setTimeout recursivo para polling](#53-settimeout-recursivo-para-polling)
   - [5.4 Manejo de errores con instanceof](#54-manejo-de-errores-con-instanceof)
   - [5.5 Seguridad: path traversal](#55-seguridad-path-traversal)

---

## 1. Arquitectura General

```
[Usuario/Navegador]
       │
       ▼  (HTTP)
┌──────────────────────┐
│    Vue 3 Frontend     │
│   (localhost:5173)    │
│                       │
│  ┌─────────────────┐  │
│  │   DocList.vue    │  │  ← Lista de documentos (sidebar)
│  │   DocViewer.vue  │  │  ← Contenido renderizado
│  │   App.vue        │  │  ← Orquestador (estado compartido)
│  └─────────────────┘  │
└──────────┬───────────┘
           │  fetch() a :3000/api/docs
           ▼  (CORS)
┌──────────────────────┐
│   Express Backend    │
│   (localhost:3000)   │
│                       │
│  ┌─────────────────┐  │
│  │  markdown/       │  │  ← Rutas + lógica de documentos
│  │  sync/           │  │  ← Sincronización con GitHub
│  │  index.ts        │  │  ← Punto de entrada
│  └─────────────────┘  │
└──────────────────────┘
           │
           ▼  (git clone/pull)
   ┌───────────────┐
   │   docs/        │  ← Repositorio clonado (archivos .md)
   └───────────────┘
```

---

## 2. Backend — Express + Node.js

### 2.1 Estructura del proyecto

```
backend/
├── .env                    # Variables de entorno
├── index.ts                # Punto de entrada — configura Express
├── package.json            # Dependencias y scripts
├── tsconfig.json           # Configuración de TypeScript
├── docs/                   # Carpeta donde se clona el wiki (git)
├── markdown/
│   ├── markdown.routes.ts   # Rutas HTTP para documentos
│   └── markdown.service.ts  # Lógica de negocio: leer documentos
├── sync/
│   └── sync.service.ts      # Sincronización con Git
└── node_modules/            # Dependencias instaladas
```

**Organización modular por funcionalidad** — cada dominio (markdown, sync) agrupa todo lo relacionado.

### 2.2 Express setup (`index.ts`)

**Archivo:** `backend/index.ts`

```typescript
import 'dotenv/config'
import express from 'express';
import cors from "cors";

import markdownRouter from './markdown/markdown.routes.js'
import { startSync } from './sync/sync.service.js'

const app = express()
const port = process.env.PORT ?? 3000

app.use(cors())
app.use(express.json())

startSync()

app.use('/api/docs', markdownRouter);

app.get('/', (req, res) => {
    res.send("simplewiki backend")
})

app.listen(port, '0.0.0.0',() => {
    console.log(`SimpleWiki backend running in http://localhost:${port}`)
})
```

| Línea | Qué hace | Por qué |
|---|---|---|
| `import 'dotenv/config'` | Carga variables de `.env` en `process.env` | Así usamos `process.env.PORT`, etc. |
| `app.use(cors())` | Permite peticiones desde otros dominios | El frontend está en otro puerto (5173) |
| `app.use(express.json())` | Parsea el body de peticiones JSON | Middleware necesario para recibir JSON |
| `startSync()` | Inicia el primer clone/pull del repo | El wiki debe estar disponible al arrancar |
| `app.use('/api/docs', markdownRouter)` | Monta rutas bajo `/api/docs` | Todas las rutas del router empiezan con ese prefijo |
| `app.listen(port, '0.0.0.0', ...)` | Escucha en todas las interfaces de red | Permite acceder desde otros dispositivos en la red |

> ⚠️ En los imports se usa extensión `.js` aunque los archivos fuente sean `.ts`. Con `"module": "NodeNext"`, TypeScript espera extensiones `.js` para que el código compilado funcione directamente.

### 2.3 Servicio de sincronización (`sync.service.ts`)

**Archivo:** `backend/sync/sync.service.ts`

```typescript
import { execSync } from 'child_process';
import fs from 'fs';

const url = process.env.GITHUB_URL ?? ''
const interval = parseInt(process.env.SYNC_INTERVAL ?? '60000')

function sync() {
    try {
        const exists = fs.existsSync('docs')
        if (exists) {
            execSync('git -C docs pull', { stdio: 'inherit' })
        } else {
            execSync(`git clone --depth 1 ${url} docs`, { stdio: 'inherit' })
        }
    } catch (err) {
        if (err instanceof Error) {
            console.error(`Sync error: ${err.message}`)
        }
    }
}

export function startSync() {
    if (!url) {
        console.error('GITHUB_URL not set')
        return
    }
    sync()
    setTimeout(() => startSync(), interval)
}
```

#### `execSync` (de `child_process`)

Ejecuta comandos del sistema operativo de forma **síncrona** (bloquea la ejecución hasta que el comando termina).

- `{ stdio: 'inherit' }` — la salida del comando se muestra directamente en la consola del backend.
- `git clone --depth 1 <url> <nombre>` — clona solo el último commit (superficial) en una carpeta con nombre específico.
- `git -C <carpeta> pull` — ejecuta `git pull` dentro de esa carpeta sin hacer `cd` previo.

#### `fs.existsSync('docs')`

Comprueba si la carpeta existe. Si existe → `git pull`. Si no → `git clone`.

#### Bucle infinito con `setTimeout`

```typescript
export function startSync() {
    sync()                                          // ejecuta ahora
    setTimeout(() => startSync(), interval)          // se programa a sí misma
}
```

**¿Por qué `setTimeout` en vez de `setInterval`?**

- `setInterval` ejecuta cada N ms **sin importar si la ejecución anterior terminó**. Si sync tarda más que el intervalo, se acumulan llamadas.
- `setTimeout` **espera a que termine** y luego programa la siguiente. Nunca hay dos sincronizaciones en paralelo.

### 2.4 Servicio de Markdown (`markdown.service.ts`)

**Archivo:** `backend/markdown/markdown.service.ts`

```typescript
import fs from 'fs';

type DocMeta = {
    file_name: string
    path: string
}

export function listDocs(): DocMeta[] {
    const files = fs.readdirSync('docs/').filter(f => f.endsWith('.md'))

    var formated_files: DocMeta[] = []

    files.forEach(file => {
        const doc = {
            file_name: file.split(".md")[0].replaceAll('_', ' '), 
            path: file
        }
        formated_files.push(doc)
    });

    return formated_files;
}

export function getDoc(path: string): string | null {
    if (path.includes('..')) return null

    try {
        return fs.readFileSync('docs/' + path, 'utf-8')
    } catch (err) {
        if (err instanceof Error && 'code' in err && (err as any).code === 'ENOENT') {
            try {
                return fs.readFileSync('docs/' + path + '.md', 'utf-8')
            } catch {
                return null
            }
        }
        return null
    }
}
```

#### El tipo `DocMeta`

```typescript
type DocMeta = {
    file_name: string
    path: string
}
```

Un **type alias** de TypeScript que define la forma de los metadatos de un documento:
- `file_name`: nombre legible para mostrar (sin `.md`, guiones bajos como espacios).
- `path`: nombre real del archivo (con extensión) para recuperarlo después.

#### `listDocs()` — Listar documentos

1. **`fs.readdirSync('docs/')`** — Lee el contenido de la carpeta `docs/` de forma síncrona y devuelve un array con los nombres.
2. **`.filter(f => f.endsWith('.md'))`** — Se queda solo con los archivos `.md`.
3. **`.split(".md")[0]`** — Quita la extensión `.md`.
4. **`.replaceAll('_', ' ')`** — Convierte guiones bajos en espacios para nombres más amigables.

#### `getDoc(path)` — Leer un documento

**Seguridad: control de path traversal**

```typescript
if (path.includes('..')) return null
```

**Path traversal** es un ataque donde alguien escribe `../../etc/passwd` para salirse de la carpeta permitida y leer archivos del sistema. Esta línea previene ese ataque: si la ruta contiene `..` (subir al directorio padre), devuelve `null`.

**`ENOENT`** — Error NO ENTry (el archivo no existe). Si no encuentra el archivo, **intenta añadir `.md`** automáticamente y lo vuelve a leer. Esto permite pedir documentos con o sin extensión.

### 2.5 Rutas de Markdown (`markdown.routes.ts`)

**Archivo:** `backend/markdown/markdown.routes.ts`

```typescript
import { Router, Request, Response } from 'express';

import { listDocs, getDoc } from './markdown.service.js'

const markdownRouter: Router = Router();
export default markdownRouter;

markdownRouter.get('/', (req: Request, res: Response) => {
    try {
        const docs = listDocs()
        res.json({ docs })
    } catch (err) {
        res.status(500).json({ error: 'Failed to list documents' })
    }
});

markdownRouter.get('/:name', (req: Request, res: Response) => {
    const name = req.params.name;
    const doc = getDoc(name)
    if (doc === null) {
        res.status(404).json({ error: 'Document not found' })
        return
    }
    res.json({ content: doc })
});
```

**`Router`** — Crea un mini-enrutador independiente. Montado en `index.ts` con `app.use('/api/docs', markdownRouter)`:
- `'/'` del router → `GET /api/docs`
- `'/:name'` del router → `GET /api/docs/:name`

**Endpoint `GET /api/docs`** — Lista documentos. Devuelve `{ docs: [...] }`.

**Endpoint `GET /api/docs/:name`** — Obtiene un documento.
- `req.params.name` captura el parámetro dinámico de la URL.
- **`return` después de `res.status(404).json(...)`** — Sin el `return`, la ejecución continuaría e intentaría enviar otra respuesta, lo que lanza error.

### 2.6 `package.json` — scripts y dependencias

**Archivo:** `backend/package.json`

```json
{
  "name": "backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "cors": "^2.8.6",
    "dotenv": "^17.4.2",
    "express": "^5.2.1"
  },
  "devDependencies": {
    "@types/express": "^5.0.6",
    "@types/node": "^25.9.0",
    "tsx": "^4.22.3",
    "typescript": "^6.0.3"
  }
}
```

#### `"type": "module"`

Activa módulos ES (`import`/`export`) en lugar de CommonJS (`require`/`module.exports`).

#### Scripts

| Script | Comando | Uso |
|---|---|---|
| `dev` | `tsx watch index.ts` | Desarrollo con **recarga automática** (tsx ejecuta TS directamente) |
| `build` | `tsc` | Compila TS a JS usando `tsconfig.json` → carpeta `dist/` |
| `start` | `node dist/index.js` | Producción: ejecuta el JS compilado |

#### Dependencias

| Paquete | Propósito |
|---|---|
| `express` | Framework web HTTP |
| `cors` | Middleware CORS (permite peticiones cross-origin) |
| `dotenv` | Carga variables de `.env` en `process.env` |
| `tsx` | Ejecuta TypeScript directamente sin compilar (modo watch incluido) |
| `typescript` | Compilador TS (`tsc`) |
| `@types/express` | Tipos TypeScript para Express |
| `@types/node` | Tipos TypeScript para Node.js (fs, child_process, process...) |

### 2.7 `tsconfig.json` — configuración TypeScript

**Archivo:** `backend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["./**/*.ts", "index.ts"],
  "exclude": ["node_modules", "dist"]
}
```

| Opción | Valor | Significado |
|---|---|---|
| `target` | `ES2022` | Compila a JS compatible con ES2022 (async/await, for...of, etc.) |
| `module` | `NodeNext` | Usa módulos ES nativos. Con `"type": "module"`, genera imports ES |
| `moduleResolution` | `NodeNext` | Busca en `node_modules`, respeta extensiones `.js` en imports |
| `outDir` | `./dist` | Carpeta donde se guarda el JS compilado |
| `rootDir` | `./` | Carpeta raíz del código fuente (preserva estructura al compilar) |
| `strict` | `true` | Activa todas las comprobaciones estrictas de TypeScript |
| `esModuleInterop` | `true` | Permite `import express from 'express'` en módulos CommonJS |
| `skipLibCheck` | `true` | Omite chequeo de tipos en dependencias (acelera compilación) |

### 2.8 Variables de entorno (`.env`)

```env
PORT = "3000"
GITHUB_URL = "https://github.com/tu-user/tu-repo"
```

Se cargan con `import 'dotenv/config'` y se accede con `process.env.VARIABLE ?? 'default'`.

El operador `??` (nullish coalescing) da un valor por defecto solo si la variable es `null` o `undefined`.

---

## 3. Frontend — Comunicación entre Componentes

### 3.1 Props (padre → hijo)

Un **prop** es un dato que un componente **padre** le pasa a un **componente hijo**. El flujo es siempre **unidireccional**: del padre hacia el hijo.

#### Declarar Props en el hijo (`defineProps`)

```typescript
// DocViewer.vue — el hijo declara qué props espera recibir
const props = defineProps({ doc: String })
// Ahora puedes usar props.doc
```

`defineProps` es una **macro del compilador** de Vue (no necesita importarse). Recibe un objeto donde las claves son nombres de props y los valores son los tipos esperados.

#### Pasar Props desde el padre (`:propName`)

```vue
<!-- App.vue — el padre pasa el valor -->
<DocViewer :doc="selectedDoc" />
```

El `:` es abreviatura de `v-bind:` y significa "esto es una expresión de JavaScript, no un string literal".

**Diferencia clave:**
- `doc="selectedDoc"` → pasa el **string literal** `"selectedDoc"`
- `:doc="selectedDoc"` → pasa el **valor de la variable** `selectedDoc`

### 3.2 Emits (hijo → padre)

Un **emit** es un mecanismo mediante el cual un componente hijo **lanza un evento** hacia arriba, al componente padre.

#### Declarar Emits en el hijo (`defineEmits`)

```typescript
// DocList.vue
const emit = defineEmits(['select-doc'])
```

Declara que el componente puede emitir un evento llamado `select-doc`.

#### Emitir el evento

```vue
<!-- DocList.vue — en el template -->
<li @click="emit('select-doc', doc.path)">
```

- `emit('select-doc', doc.path)` — primer argumento: nombre del evento; segundo: datos a enviar.
- Los nombres de eventos siguen **kebab-case** (con guiones).

#### Escuchar el evento en el padre

```vue
<!-- App.vue -->
<Doclist @select-doc="handleSelectDoc" />
```

```typescript
// App.vue
function handleSelectDoc(path: any) {
  selectedDoc.value = path
}
```

El dato que el hijo emite (`doc.path`) aparece automáticamente como primer parámetro de la función manejadora.

### 3.3 Flujo completo en SimpleWiki

```
DocList hace clic
  → @click="emit('select-doc', doc.path)"
  → evento 'select-doc' viaja hacia arriba con doc.path
       ↓
App.vue escucha
  → @select-doc="handleSelectDoc"
  → handleSelectDoc(path) { selectedDoc.value = path }
  → selectedDoc cambia reactivamente
       ↓
Vue detecta el cambio y re-renderiza
  → <DocViewer :doc="selectedDoc" />
  → el prop doc ahora tiene el nuevo valor
       ↓
DocViewer recibe el cambio
  → const props = defineProps({ doc: String })
  → watch(() => props.doc, async (newPath) => { ... })
  → fetch() al backend, parsea con marked, renderiza con v-html
       ↓
¡Usuario ve el documento!
```

### 3.4 `watch`: reaccionar a cambios en Props

```typescript
// DocViewer.vue
import { watch, ref } from 'vue'

const props = defineProps({ doc: String })
const content = ref<any>('')

watch(() => props.doc, async (newPath) => {
    if (!newPath) return
    const res = await fetch('http://localhost:3000/api/docs/' + newPath)
    const data = await res.json()
    content.value = marked.parse(data.content)
})
```

**Sintaxis:**

```typescript
watch(
  () => props.doc,          // 1. FUENTE: función que devuelve el valor a observar
  async (newPath) => { ... } // 2. CALLBACK: qué hacer cuando cambie
)
```

| Parámetro | Significado |
|---|---|
| `() => props.doc` | **Getter function** — Se usa función flecha en vez de pasar `props.doc` directamente porque Vue necesita rastrear la reactividad correctamente en propiedades de un objeto `props`. |
| `newPath` | Primer argumento del callback — contiene el **nuevo valor** de `props.doc`. También puedes recibir el valor anterior: `(newPath, oldPath)`. |
| `if (!newPath) return` | **Guard clause** — Si el path está vacío (al inicio de la app), sale sin hacer nada y evita un fetch innecesario. |

**¿Por qué `watch` y no `onMounted`?**

| | `onMounted` | `watch` |
|---|---|---|
| Se ejecuta | Una sola vez, al montar el componente | Cada vez que el valor observado cambia |
| Sirve para | Cosas que pasan una vez al inicio | Reaccionar a cambios en el tiempo |
| ¿Funciona aquí? | ❌ No. Si cambia `props.doc`, no se ejecuta otra vez | ✅ Sí. Reacciona a cada cambio |

### 3.5 Sintaxis de templates explicada

| Sintaxis | Significado | Ejemplo |
|---|---|---|
| `@click` | Abreviatura de `v-on:click`. Escucha evento nativo de clic | `@click="emit('select-doc', doc.path)"` |
| `@select-doc` | Escucha un **evento personalizado** emitido por el hijo | `@select-doc="handleSelectDoc"` |
| `:doc` | Abreviatura de `v-bind:doc`. Binding dinámico de prop | `:doc="selectedDoc"` |
| `v-for="doc in DocList"` | Itera sobre un array y renderiza el elemento por cada ítem | `<li v-for="doc in DocList">` |
| `{{ doc.file_name }}` | **Interpolación**: muestra el valor como texto | `{{ doc.file_name }}` |
| `v-html="content"` | Inserta HTML real en el DOM (como innerHTML) | `<div v-html="content">` |

**`v-html` vs `{{ }}`:**

```vue
<!-- Si content = "<strong>Hola</strong>" -->
<div v-html="content" />    → Muestra: Hola (en negrita)
<div>{{ content }}</div>    → Muestra: <strong>Hola</strong> (texto literal)
```

---

## 4. Frontend — Renderizado de Markdown

### 4.1 `marked` — conversor de MD a HTML

**Propósito:** Convierte texto Markdown en HTML.

```typescript
import { Marked } from 'marked'   // Versión 18+, usa { Marked } no { marked }

const html = marked.parse('# Título\n\nEsto es **negrita**')
// <h1>Título</h1>\n<p>Esto es <strong>negrita</strong></p>
```

**Entrada:** string en Markdown.
**Salida:** string en HTML.

### 4.2 `v-html` — directiva de Vue para HTML crudo

`v-html` inserta HTML directamente en el DOM, como `innerHTML`. Es necesaria porque `{{ }}` escapa el HTML y lo muestra como texto literal.

```vue
<div class="markdown-body" v-html="content"></div>
```

**⚠️ Seguridad:** `v-html` no sanitiza el contenido. Si el HTML contiene `<script>` o `onerror`, se ejecutará. Solo usarlo con contenido de confianza (como tu propio backend).

### 4.3 `highlight.js` — coloreado de sintaxis

**Propósito:** Da color a los bloques de código según el lenguaje.

```typescript
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'  // tema claro estilo GitHub

const language = hljs.getLanguage(lang) ? lang : 'plaintext'
return hljs.highlight(code, { language }).value
```

Las clases CSS que genera (`hljs-keyword`, `hljs-string`, etc.) necesitan un **tema CSS** para tener color. Temas populares:

| Import | Tema |
|---|---|
| `highlight.js/styles/github.css` | Claro estilo GitHub |
| `highlight.js/styles/github-dark.css` | Oscuro estilo GitHub |
| `highlight.js/styles/atom-one-dark.css` | Oscuro estilo Atom |
| `highlight.js/styles/monokai.css` | Oscuro estilo Monokai |

### 4.4 `marked-highlight` — puente entre marked y highlight.js

**Propósito:** Extiende `marked` para que los bloques de código se coloreen durante la conversión.

Sin `marked-highlight`:

```
```javascript
const x = 42;
````

→ `<pre><code class="language-javascript">const x = 42;</code></pre>` (sin colores)

Con `marked-highlight`:

→ `<pre><code class="hljs language-javascript"><span class="hljs-keyword">const</span> x = <span class="hljs-number">42</span>;</code></pre>` (con colores)

#### Configuración completa

```typescript
import { Marked } from 'marked'
import { markedHighlight } from 'marked-highlight'
import hljs from 'highlight.js'

const marked = new Marked(
    markedHighlight({
        emptyLangClass: 'hljs',
        langPrefix: 'hljs language-',
        highlight(code, lang) {
            const language = hljs.getLanguage(lang) ? lang : 'plaintext'
            return hljs.highlight(code, { language }).value
        }
    })
)

// Ahora marked.parse() genera HTML con código coloreado
```

**Opciones de `markedHighlight`:**

| Opción | Tipo | Default | Descripción |
|---|---|---|---|
| `async` | boolean | `false` | `true` si la función `highlight` devuelve una Promise |
| `langPrefix` | string | `'language-'` | Prefijo para la clase CSS del `<code>` |
| `emptyLangClass` | string | `''` | Clase añadida si el bloque no especifica lenguaje |
| `highlight` | function | — | **Requerida.** Función que transforma código en HTML coloreado |

### 4.5 `github-markdown-css` — estilo visual

**Propósito:** Aplica estilos CSS al HTML renderizado para que se vea como el Markdown de GitHub (tipografía, tablas, listas, blockquotes, etc.).

#### Archivos CSS disponibles

| Archivo | Comportamiento |
|---|---|
| `github-markdown.css` | **Auto**: cambia entre claro/oscuro según `prefers-color-scheme` |
| `github-markdown-light.css` | Solo claro |
| `github-markdown-dark.css` | Solo oscuro |
| `github-markdown-dark-dimmed.css` | Oscuro atenuado |
| `github-markdown-dark-high-contrast.css` | Oscuro alto contraste |
| `github-markdown-dark-colorblind.css` | Oscuro daltonismo |
| `github-markdown-light-colorblind.css` | Claro daltonismo |

#### Uso

```ts
// main.ts — importar el CSS globalmente
import 'github-markdown-css/github-markdown-light.css'
```

```vue
<!-- En el componente: añadir clase markdown-body al contenedor -->
<div class="markdown-body" v-html="content"></div>
```

**Estilos recomendados para el contenedor:**

```css
.markdown-body {
    box-sizing: border-box;
    min-width: 200px;
    max-width: 980px;
    margin: 0 auto;
    padding: 45px;
}

@media (max-width: 767px) {
    .markdown-body {
        padding: 15px;
    }
}
```

### 4.6 Pipeline completo de renderizado

```
Archivo .md en el servidor (docs/)
       │
       ▼
  GET /api/docs/mi-documento
       │
       ▼
  Backend responde:
  { "content": "# Título\n\n```js\nconst a=1;\n```" }
       │
       ▼
  marked.parse(data.content)
       │
       ├── marked-highlight detecta bloque ```js
       ├── llama a hljs.highlight("const a=1;", { language: "javascript" })
       └── devuelve HTML con <span class="hljs-keyword">const</span>...
       │
       ▼
  content.value = "<h1>Título</h1><pre><code class=\"hljs ...\">..."
       │
       ▼
  <div class="markdown-body" v-html="content"></div>
       │
       ├── v-html inyecta el HTML en el DOM
       └── github-markdown-css aplica los estilos visuales
       │
       ▼
  ¡El usuario ve el documento formateado y con código coloreado!
```

#### Código completo de DocViewer.vue

```vue
<script setup lang="ts">
import { watch, ref } from 'vue';
import { Marked } from 'marked'
import { markedHighlight } from 'marked-highlight'
import hljs from 'highlight.js'
import 'highlight.js/styles/github.css'

const props = defineProps({ doc: String })
const content = ref<any>('');

const marked = new Marked(
    markedHighlight({
        emptyLangClass: 'hljs',
        langPrefix: 'hljs language-',
        highlight(code, lang) {
            const language = hljs.getLanguage(lang) ? lang : 'plaintext'
            return hljs.highlight(code, { language }).value
        }   
    })
)

watch(() => props.doc, async (newPath) => {
    if (!newPath) return
    const res = await fetch('http://localhost:3000/api/docs/' + newPath)
    const data = await res.json()
    content.value = marked.parse(data.content)
})
</script>

<template>
    <div class="markdown-body" v-html="content"></div>
</template>
```

---

## 5. Conceptos Clave Explicados

### 5.1 TypeScript básico

#### `type` vs `any`

```typescript
// any — desactiva TypeScript, pierdes toda la ayuda
const x: any[] = []   // ❌ cualquier cosa vale

// type — defines la forma exacta del objeto
type DocMeta = {
  file_name: string
  path: string
}
const docs: DocMeta[] = []   // ✅ solo objetos con file_name y path

// void — la función no devuelve nada
function log(msg: string): void { console.log(msg) }

// string | null — puede devolver string O null
function getDoc(path: string): string | null { ... }
```

#### Type narrowing (estrechamiento de tipos)

```typescript
catch (err) {              // err es "unknown" (no se sabe qué tipo es)
  if (err instanceof Error) {
    err.message             // ✅ dentro del if, err ya es tipo Error
  }
  if (err instanceof Error && 'code' in err) {
    (err as any).code       // accedes a propiedades del error de sistema
  }
}
```

`instanceof` comprueba si el objeto fue creado por una clase concreta. Solo funciona con clases (Error, TypeError, SyntaxError...), no con códigos de error como `ENOENT`.

#### TS + ESM: extensión `.js` en imports

Con `"module": "NodeNext"`, los imports relativos usan extensión `.js` aunque el archivo real sea `.ts`:

```typescript
import { listDocs } from './markdown.service.js'
// El archivo real es markdown.service.ts, pero se escribe .js
```

#### `process.env`

No se importa, es global. Se accede directamente:

```typescript
const port = process.env.PORT ?? '3000'
const url = process.env.GITHUB_URL ?? ''
```

#### `import.meta.url` y `__dirname` (ESM)

```typescript
import { fileURLToPath } from 'node:url'
import { dirname, join } from 'node:path'

const __filename = fileURLToPath(import.meta.url)
const __dirname = dirname(__filename)

const dbPath = join(__dirname, '..', 'data', 'wiki.db')
```

**En Node 22+** existe `import.meta.dirname` directamente:

```typescript
const dbPath = join(import.meta.dirname, '..', 'data', 'wiki.db')
```

### 5.2 Módulos nativos de Node

| Módulo | Función | Para qué |
|---|---|---|
| `node:child_process` | `execSync(command, options)` | Ejecutar comandos de sistema (git) |
| `node:child_process` | `execSync` con `{ stdio: 'inherit' }` | Muestra la salida del comando en consola |
| `node:fs` | `readdirSync(path)` | Lista archivos de un directorio |
| `node:fs` | `readFileSync(path, 'utf-8')` | Lee contenido de un archivo como string |
| `node:fs` | `existsSync(path)` | Comprueba si un archivo/directorio existe |
| `node:path` | `join(...paths)` | Une segmentos de ruta de forma segura |
| `node:url` | `fileURLToPath(url)` | Convierte `file://` URLs a paths del sistema |

### 5.3 `setTimeout` recursivo para polling

```typescript
function sync() {
    // ... hacer cosas (git pull, etc.) ...
    setTimeout(sync, 60000)  // se llama a sí misma dentro de 60s
}
sync()  // arranca la primera vez
```

| | `setInterval` | `setTimeout` recursivo |
|---|---|---|
| Comportamiento | Ejecuta cada N ms exactos, sin esperar | Espera a que termine, luego cuenta N ms |
| Riesgo | Si la función tarda más que el intervalo, se acumulan llamadas | Nunca se acumulan llamadas |
| Uso recomendado | Tareas rápidas y predecibles | Tareas de duración variable (git pull, I/O) |

### 5.4 Manejo de errores con `instanceof`

```typescript
try {
    // código que puede lanzar error
} catch (err) {        // err: unknown
    if (err instanceof Error) {
        console.error(err.message)       // err tratado como Error
    }
}
```

**Para errores de sistema (código `ENOENT`):**

```typescript
} catch (err) {
    if (err instanceof Error && 'code' in err && (err as any).code === 'ENOENT') {
        // archivo no encontrado
    }
}
```

- `err instanceof Error` → confirma que es un objeto Error
- `'code' in err` → verifica que tiene propiedad `code`
- `(err as any).code === 'ENOENT'` → comprueba que es el error específico

**`ENOENT`** = Error NO ENTry (archivo o directorio no existe).

### 5.5 Seguridad: path traversal

```typescript
if (path.includes('..')) return null
```

**Path traversal** — el usuario malicioso escribe `../../etc/passwd` para leer archivos fuera de la carpeta permitida.

`..` en sistemas de archivos significa "subir al directorio padre". Si la ruta contiene `..`, la función devuelve `null` y no intenta leer nada.

---

## Referencias rápidas

### Comandos npm

| Comando | Para qué |
|---|---|
| `npm run dev` | Desarrollo con hot-reload (backend o frontend) |
| `npm run build` | Compila TS a JS (backend) o build de producción (frontend) |
| `npm start` | Ejecuta la app compilada (backend) |
| `npm install <paquete>` | Instala una dependencia |
| `npm install -D <paquete>` | Instala una dependencia de desarrollo |
| `npm uninstall <paquete>` | Elimina una dependencia |

### Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `ERR_UNKNOWN_FILE_EXTENSION .ts` | Ejecutar `.ts` con `node` en vez de `tsx` | Usa `npx tsx archivo.ts` |
| `Cannot find module 'x'` | No está instalado | `npm install x` |
| `esbuild platform mismatch` | `node_modules` mezcla Windows/WSL | `rm -rf node_modules && npm install` desde el SO correcto |
| `marked(): input parameter is object` | Le pasas un objeto a `marked.parse()` | Pasa `data.content`, no `data` entero |
| `Component setup returned promise, no Suspense` | Usas `await` en `<script setup>` sin `onMounted` | Mueve el fetch a `onMounted` |
| `v-for` imprime todo junto | Iteras sobre un objeto en vez de un array | Usa `data = json.docs` no `data = json` |

### WSL / Windows

- Los `node_modules` con binarios nativos (esbuild, bcrypt, better-sqlite3) **NO** se pueden compartir entre Windows y WSL.
- Si instalaste con npm en Windows, ejecuta desde **PowerShell**.
- Si instalaste con npm en WSL, ejecuta desde **WSL**.
- Nunca mezclar — da el error "esbuild for another platform".

---

*Documentación generada para el proyecto SimpleWiki.*
