# Guía: Plantillas Dinámicas en Vue.js

> Una guía paso a paso para reemplazar páginas individuales de proyectos por una sola plantilla que cambia según el proyecto.

---

## Índice

1. [Qué es una plantilla dinámica](#1-qué-es-una-plantilla-dinámica)
2. [Cómo funciona el router dinámico](#2-cómo-funciona-el-router-dinámico)
3. [De dónde viene la información](#3-de-dónde-viene-la-información)
4. [Cómo sabe Vue dónde poner cada dato](#4-cómo-sabe-vue-dónde-poner-cada-dato)
5. [Estructura paso a paso](#5-estructura-paso-a-paso)
6. [Cómo Vue sabe qué proyecto mostrar](#6-cómo-vue-sabe-qué-proyecto-mostrar)

---

## 1. Qué es una plantilla dinámica

Imagina que tienes dos páginas para tus proyectos:

- `WordFinderView.vue` → se ve en `/wordfinder`
- `ShortlessView.vue` → se ve en `/shortless`

Cada una es un archivo separado. Si mañana añades un tercer proyecto, tienes que crear otro archivo. Y otro. Y otro...

Una **plantilla dinámica** es un **único archivo Vue** que sirve para **todos** los proyectos. La página es la misma, pero el contenido cambia según qué proyecto se pida. Es como un formulario en blanco que se rellena con datos diferentes cada vez.

```
ANTES (una página por proyecto):
/wordfinder  →  WordFinderView.vue
/shortless   →  ShortlessView.vue
/otro        →  OtroView.vue   (tendrías que crearla)

DESPUÉS (una plantilla para todos):
/proyecto/wordfinder  →  ProjectView.vue  (con datos de WordFinder)
/proyecto/shortless   →  ProjectView.vue  (con datos de Shortless)
/proyecto/otro        →  ProjectView.vue  (¡sin crear nada nuevo!)
```

---

## 2. Cómo funciona el router dinámico

En Vue Router, puedes definir una ruta con un **parámetro dinámico** usando dos puntos `:` delante del nombre.

```js
// En src/router/index.js

{ path: '/proyecto/:slug', component: ProjectView }
```

### ¿Qué es `:slug`?

El `:slug` es un **hueco variable**. Significa: "cualquier cosa que el usuario escriba aquí, guárdalo con el nombre `slug`".

| URL que visita el usuario | Valor de `slug` |
|---|---|
| `/proyecto/wordfinder` | `"wordfinder"` |
| `/proyecto/shortless` | `"shortless"` |
| `/proyecto/mi-proyecto-genial` | `"mi-proyecto-genial"` |

Dentro de tu componente `ProjectView.vue`, puedes leer ese valor así:

```vue
<script setup>
import { useRoute } from 'vue-router'

const route = useRoute()
const slug = route.params.slug  // aquí llega "wordfinder", "shortless", etc.
</script>
```

> **Nota:** `slug` es solo un nombre. Podría llamarse `:id`, `:nombre`, `:loquesea`. Lo importante es que coincida cuando lo lees con `route.params`.

---

## 3. De dónde viene la información

La plantilla necesita saber qué datos mostrar para cada proyecto. Una forma sencilla es guardar toda la información en un **archivo JSON**.

### ¿Qué es JSON?

JSON es un formato de texto para guardar datos estructurados. Piensa en él como una **ficha** para cada proyecto:

```json
{
  "wordfinder": {
    "titulo": "WordFinder",
    "descripcion": "Busca palabras en clips de Twitch",
    "url": "https://wordfinder.cusssy.com"
  },
  "shortless": {
    "titulo": "Shortless.link",
    "descripcion": "Acortador de enlaces",
    "url": "https://shortless.link"
  }
}
```

Cada proyecto tiene una **clave** (`wordfinder`, `shortless`) que coincide con el `slug` de la URL. Así es como se conectan las piezas:

```
URL: /proyecto/wordfinder
              ↓
       route.params.slug = "wordfinder"
              ↓
    projectsData["wordfinder"]
              ↓
     → { titulo: "WordFinder", descripcion: "..." }
```

### ¿Dónde guardar el JSON?

Una buena ubicación es `src/data/projects.json`. Al estar dentro de `src/`, Vite lo puede importar directamente desde tus componentes.

---

## 4. Cómo sabe Vue dónde poner cada dato

Esta es la parte más importante. Veamos el flujo completo con un ejemplo concreto.

### La analogía del restaurante

Piensa en esto como un restaurante:

1. **El JSON** es el **almacén** donde están guardados todos los ingredientes (los datos).
2. **El `<script setup>`** es el **cocinero** que va al almacén, recoge los ingredientes que necesita y los prepara.
3. **El `<template>`** es el **plato servido** donde el cliente ve el resultado final.

### El flujo paso a paso

```
┌─────────────────────────────────────────────────────────────┐
│  projects.json (el almacén)                                  │
│  ┌───────────────────────────────────────────┐              │
│  │ "wordfinder": {                           │              │
│  │   "titulo": "WordFinder",                 │              │
│  │   "descripcion": "Busca palabras..."      │              │
│  │   "url": "https://wordfinder.cusssy.com"  │              │
│  │ }                                         │              │
│  └───────────────────────────────────────────┘              │
└────────────────────────────┬────────────────────────────────┘
                             │  import projectsData
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  <script setup> (el cocinero)                                │
│  ┌───────────────────────────────────────────┐              │
│  │ import projectsData from '@/data/projects.json' │        │
│  │ import { useRoute } from 'vue-router'     │              │
│  │                                           │              │
│  │ const route = useRoute()                  │              │
│  │ const slug = route.params.slug  // "wordfinder"          │
│  │ const proyecto = projectsData[slug]       │              │
│  │ // → proyecto ahora contiene:             │              │
│  │ //   { titulo: "WordFinder", ... }        │              │
│  └───────────────────────────────────────────┘              │
└────────────────────────────┬────────────────────────────────┘
                             │  proyecto (reactivo)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  <template> (el plato servido)                               │
│  ┌───────────────────────────────────────────┐              │
│  │ <h1>{{ proyecto.titulo }}</h1>            │              │
│  │   → Vue reemplaza con: "WordFinder"       │              │
│  │                                           │              │
│  │ <p>{{ proyecto.descripcion }}</p>         │              │
│  │   → Vue reemplaza con: "Busca palabras..."│              │
│  │                                           │              │
│  │ <a :href="proyecto.url">Visitar</a>       │              │
│  │   → Vue reemplaza con:                    │              │
│  │     href="https://wordfinder.cusssy.com"  │              │
│  └───────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

### ¿Qué son esas llaves `{{ }}`?

Las **llaves dobles** `{{ }}` le dicen a Vue: "aquí dentro hay una expresión de JavaScript, calcula su valor y pon el resultado aquí".

```vue
<!-- Esto en tu template -->
<h1>{{ proyecto.titulo }}</h1>

<!-- Vue lo convierte en esto en el navegador -->
<h1>WordFinder</h1>
```

Es como un **formulario con campos en blanco**: `{{ proyecto.titulo }}` es el campo en blanco, y Vue lo rellena con el valor que viene del script.

### La notación de punto `.`

Cuando escribes `proyecto.titulo`, estás diciendo: "del objeto `proyecto`, dame la propiedad `titulo`". Es como abrir un cajón etiquetado:

```
proyecto → es el mueble completo (el objeto)
proyecto.titulo → es el cajón que dice "titulo" (la propiedad)
```

---

## 5. Estructura paso a paso

### Step 1: Crear `projects.json` con datos de ejemplo

Crea un archivo en `src/data/projects.json` con esta estructura:

```json
{
  "wordfinder": {
    "titulo": "WordFinder",
    "descripcion": "Busca palabras en clips de Twitch!",
    "url": "https://wordfinder.cusssy.com",
    "tecnologias": ["Vue.js", "Node.js", "Twitch API"]
  },
  "shortless": {
    "titulo": "Shortless.link",
    "descripcion": "Acortador de enlaces",
    "url": "https://shortless.link",
    "tecnologias": ["Node.js", "MongoDB"]
  }
}
```

> Cada clave (`wordfinder`, `shortless`) será el `slug` que se usa en la URL.

### Step 2: Crear `ProjectView.vue` como plantilla

Crea un archivo en `src/views/ProjectView.vue`. La estructura será similar a tus vistas actuales, pero con lógica para leer el `slug` y buscar los datos:

```vue
<script setup>
// Aquí necesitas:
// 1. Importar useRoute de vue-router
// 2. Importar el JSON de datos
// 3. Leer el slug de la URL
// 4. Buscar el proyecto correspondiente en el JSON
// 5. (Opcional) Manejar el caso de que el slug no exista
</script>

<template>
  <main>
    <!-- Muestra el título del proyecto -->
    <!-- Muestra la descripción -->
    <!-- Muestra las tecnologías usando TagComponent -->
    <!-- Incluye un enlace para visitar el proyecto -->
  </main>
</template>
```

> Usa el componente `<TagComponent>` que ya tienes para mostrar las tecnologías. Puedes pasárselo con `v-for` para crear una etiqueta por cada tecnología.

### Step 3: Actualizar `router/index.js` con la ruta dinámica

Modifica `src/router/index.js` para añadir la ruta dinámica:

```js
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'
import ProjectView from '../views/ProjectView.vue'  // ← tu nueva plantilla

const routes = [
  { path: '/', component: HomeView },
  { path: '/proyecto/:slug', component: ProjectView },  // ← ruta dinámica
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
})

export default router
```

> Ya no necesitas importar `WordFinderView`, `ShortlessView`, etc. La única vista que maneja todos los proyectos es `ProjectView`.

### Step 4: Actualizar los links en `ProyectosComponentes.vue`

Actualmente usas `<a href="...">` que apuntan a URLs externas. Para navegar dentro de tu app, usa `<router-link>`:

```vue
<script setup>
// Ya no necesitas nada especial aquí
</script>

<template>
  <div id="proyectos-component">
    <p class="sub-titulo">PROYECTOS</p>
    <div id="proyectos">
      <ul>
        <li>
          <!-- Cambia <a href="..."> por <router-link to="..."> -->
          <!-- El "to" apunta a la ruta dinámica con el slug -->
          <router-link to="/proyecto/wordfinder">
            <div class="project-info">
              <p class="title">WordFinder</p>
              <p class="role">Busca palabras en clips de twitch!</p>
            </div>
            <span aria-hidden="true">&gt;</span>
          </router-link>
        </li>
        <li>
          <router-link to="/proyecto/shortless">
            <div class="project-info">
              <p class="title">Shortless.link</p>
              <p class="role">Acortador de enlaces</p>
            </div>
            <span aria-hidden="true">&gt;</span>
          </router-link>
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
/* Tus estilos se mantienen igual, solo cambia "a" por "router-link" si es necesario */
/* router-link se renderiza como <a> en el navegador, así que los estilos deberían funcionar */
</style>
```

> `<router-link>` es un componente de Vue Router que genera un `<a>` automáticamente, pero en lugar de recargar la página, navega internamente usando JavaScript. Por eso tus estilos de `#proyectos a` seguirán funcionando.

---

## 6. Cómo Vue sabe qué proyecto mostrar

Este es el momento en que todas las piezas encajan. Veamos qué pasa cuando un usuario hace clic en "WordFinder":

```
1. El usuario hace clic en el link en ProyectosComponentes
   └→ <router-link to="/proyecto/wordfinder">

2. Vue Router lee la URL y busca una ruta que coincida
   └→ Encuentra: { path: '/proyecto/:slug', component: ProjectView }
   └→ Extrae: params.slug = "wordfinder"

3. Vue crea una instancia de ProjectView y le pasa la ruta

4. Dentro de ProjectView:
   └→ const route = useRoute()
   └→ const slug = route.params.slug      → "wordfinder"
   └→ const proyecto = projectsData[slug] → { titulo: "WordFinder", ... }

5. El template se renderiza con esos datos:
   └→ {{ proyecto.titulo }}       → "WordFinder"
   └→ {{ proyecto.descripcion }}  → "Busca palabras en clips de Twitch!"
```

### Resumen visual del flujo completo

```
┌──────────────────┐
│  Clic en link    │
│  "WordFinder"    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐     ┌──────────────────┐
│  URL cambia a:   │────▶│  Vue Router      │
│  /proyecto/      │     │  extrae slug     │
│  wordfinder      │     │  = "wordfinder"  │
└──────────────────┘     └────────┬─────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  ProjectView.vue         │
                    │  lee route.params.slug   │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │  projects.json           │
                    │  projectsData["wordfinder"]│
                    │  → datos del proyecto    │
                    └────────────┬─────────────┘
                                 │
                                 ▼
                    ┌──────────────────────────┐
                    │  Template se renderiza   │
                    │  {{ proyecto.titulo }}   │
                    │  → "WordFinder"          │
                    └──────────────────────────┘
```

### Cuando el usuario hace clic en otro proyecto

El proceso es **exactamente el mismo**, solo cambia el `slug`:

- Clic en Shortless → URL: `/proyecto/shortless` → slug: `"shortless"` → `projectsData["shortless"]` → datos de Shortless

La plantilla (`ProjectView.vue`) **no cambia**. Solo cambian los datos que lee.

---

## Resumen rápido

| Concepto | Qué es | Ejemplo en tu proyecto |
|---|---|---|
| `:slug` en la ruta | Un hueco variable en la URL | `/proyecto/wordfinder` → slug = `"wordfinder"` |
| `route.params.slug` | Cómo leer ese valor desde el componente | `useRoute().params.slug` |
| JSON | Donde guardas los datos de cada proyecto | `src/data/projects.json` |
| `{{ }}` | Sintaxis de Vue para mostrar datos en el template | `{{ proyecto.titulo }}` |
| `<router-link>` | Link interno que no recarga la página | `<router-link to="/proyecto/wordfinder">` |

Con esto tienes todo lo necesario para construir tu plantilla dinámica. ¡Ánimo!
