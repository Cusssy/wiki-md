# Guía de Estilo — Portfolio Vue.js

> Referencia rápida para mantener consistencia visual al crear nuevas páginas (ProjectView, About, etc.)

---

## 1. Filosofía del diseño actual

Tu portfolio tiene un estilo **minimalista, oscuro y centrado en texto**. No busca ser llamativo, sino transmitir profesionalismo y claridad. Piensa en él como una mezcla entre **terminal de desarrollador** y **portfolio clásico**.

Principios clave:

- **Menos es más**: sin decoraciones innecesarias, sin gradientes, sin sombras excesivas
- **Texto como protagonista**: la jerarquía tipográfica comunica la estructura
- **Modo oscuro por defecto**: fondo oscuro, texto claro, acentos sutiles
- **Consistencia antes que creatividad**: reutiliza patrones existentes en lugar de inventar nuevos

---

## 2. Tipografía

### Fuente principal: Poppins

Se importa desde Google Fonts con **todos los pesos** (100–900), pero en la práctica solo usas algunos:

```css
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@100;200;300;400;500;600;700;800;900&display=swap');
```

### Pesos recomendados y cuándo usarlos

| Peso | Uso | Ejemplo |
|------|-----|---------|
| 300 | Texto secundario, roles | `.role` — "Full Stack Developer" |
| 400 | Cuerpo de texto normal | Párrafos, descripciones |
| 500 | Subtítulos, h2 | `.sub-titulo`, títulos de sección |
| 600 | Título principal, h1 | Nombre del portfolio |
| 700+ | Evitar (demasiado pesado para este estilo) | — |

### Patrones tipográficos existentes

```css
/* Subtítulo de sección — uppercase, espaciado, muted */
.sub-titulo {
  font-size: 0.68rem;
  font-weight: 500;
  letter-spacing: 0.2em;
  text-transform: uppercase;
  color: var(--muted);
}

/* Rol / posición — ligero, secundario */
.role {
  font-size: 0.9rem;
  font-weight: 300;
  color: var(--muted);
}

/* Título principal */
h1 {
  font-family: "Poppins", sans-serif;
  font-weight: 600;
}

/* Títulos de sección */
h2 {
  font-family: "Poppins", sans-serif;
  font-weight: 500;
}
```

### Regla de oro

> **Nunca uses otra fuente que no sea Poppins.** Merriweather está importada pero no se usa. Si en el futuro quieres contrastar, úsala solo para bloques de texto largo, nunca para UI.

---

## 3. Sistema de colores

Todos los colores viven como **variables CSS** en `:root`. Esto permite cambiar el tema completo modificando un solo lugar.

### Variables (modo oscuro)

```css
:root {
  --text: #eeeeee;       /* Texto principal — blanco suave */
  --light_text: #a0a0a0; /* Texto secundario — gris medio */
  --muted: #707070;      /* Labels, subtítulos — gris apagado */
  --light: #444444;      /* Dividers, bordes secundarios */
  --border: #333333;     /* Bordes principales */
  --mega_light: #1a1a1a; /* Fondos de tags, cards */
}
```

### Jerarquía visual de los colores

```
#eeeeee  ← Lo más importante (títulos, texto principal)
#a0a0a0  ← Importante pero secundario (descripciones, links hover)
#707070  ← Contexto (labels, subtítulos, metadata)
#444444  ← Separación (líneas divisorias)
#333333  ← Estructura (bordes de componentes)
#1a1a1a  ← Fondos (casi negro, para cards/tags)
```

### Cómo usarlas consistentemente

```css
/* ✅ BIEN — siempre usa variables */
color: var(--text);
border: 1px solid var(--border);
background: var(--mega_light);

/* ❌ MAL — nunca hardcodees colores */
color: #eeeeee;
border: 1px solid #333333;
background: #1a1a1a;
```

### Modo claro (light mode)

El sitio detecta automáticamente la preferencia del sistema:

```css
@media (prefers-color-scheme: light) {
  :root {
    /* Aquí se redefinen las mismas variables con colores claros */
  }
}
```

> **Tip**: Al crear un nuevo componente, no pienses en colores absolutos. Piensa en **qué rol cumple** el color y usa la variable correspondiente.

---

## 4. Espaciado y layout

### Estructura global

```css
#app {
  max-width: 1280px;
  margin: 0 auto;       /* Centrado horizontal */
  display: flex;
  flex-direction: column;
  padding: 2rem;
}
```

### Patrón de espaciado entre secciones

```css
/* Separación entre bloques principales */
margin-top: 5rem;
```

Este valor de `5rem` es tu **ritmo vertical**. Úsalo consistentemente entre secciones grandes.

### Spaciado interno de componentes

| Elemento | Padding típico |
|----------|---------------|
| Tags | `0.25rem 0.5rem` |
| Cards/contenedores | `1rem` a `2rem` |
| Secciones | `2rem` (del #app) |

### Flexbox para listas de items

```css
/* Contenedor de proyectos, skills, etc. */
.container {
  display: flex;
  flex-wrap: wrap;    /* Permite que los items bajen de línea */
  gap: 1rem;          /* Espacio uniforme entre items */
}
```

> **`gap` es tu amigo**: reemplaza la necesidad de usar `margin-right` o `margin-bottom` en los hijos.

### Responsive básico

```css
@media (max-width: 768px) {
  #app {
    padding: 1rem;
  }

  /* Ajusta tamaños de fuente si es necesario */
  h1 {
    font-size: 1.5rem;
  }
}
```

---

## 5. Componentes reutilizables

Estos son los patrones que ya existen y debes replicar en nuevas páginas.

### Links

```css
a {
  color: var(--text);
  text-decoration: none;
  border-bottom: 1px solid var(--border);
  transition: all 0.3s;
}

a:hover {
  margin-left: 0.25rem;
  color: var(--light_text);
}
```

**Comportamiento**: línea inferior sutil + desplazamiento a la izquierda en hover.

### Tags / Badges

```css
.tag {
  display: inline-block;
  padding: 0.25rem 0.5rem;
  background: var(--mega_light);
  border: 1px solid var(--border);
  border-radius: 0.25rem;
  font-size: 0.875rem;
  color: var(--light_text);
}
```

**Uso**: tecnologías, categorías, labels de proyectos.

### Botones (si los necesitas en el futuro)

Sigue la misma lógica que los tags pero con interacción de link:

```css
.btn {
  display: inline-block;
  padding: 0.5rem 1rem;
  background: transparent;
  border: 1px solid var(--border);
  border-radius: 0.25rem;
  color: var(--text);
  font-family: "Poppins", sans-serif;
  font-size: 0.875rem;
  cursor: pointer;
  transition: all 0.3s;
}

.btn:hover {
  margin-left: 0.25rem;
  color: var(--light_text);
  border-color: var(--light);
}
```

### Checklist al crear un componente nuevo

- [ ] ¿Usa `var(--color)` en lugar de colores hardcodeados?
- [ ] ¿Usa `rem` en lugar de `px` para tipografía y spacing?
- [ ] ¿La fuente es Poppins?
- [ ] ¿Tiene `transition` si es interactivo?
- [ ] ¿Respeta el `max-width: 1280px` si es un layout?

---

## 6. Funciones y técnicas CSS útiles

### `rem` vs `px`

| Unidad | Cuándo usar | Ejemplo |
|--------|-------------|---------|
| `rem` | Tipografía, padding, margin, gap | `font-size: 0.875rem` |
| `px` | Bordes finos, bordes de 1px | `border: 1px solid` |

```css
/* ✅ BIEN */
font-size: 0.68rem;
padding: 0.25rem 0.5rem;
margin-top: 5rem;
border: 1px solid var(--border);  /* px solo para bordes de 1px */

/* ❌ MAL */
font-size: 11px;
padding: 4px 8px;
margin-top: 80px;
```

> **Por qué `rem`**: se escala con la configuración del navegador. Si un usuario cambia el tamaño base de fuente, todo tu sitio se adapta.

### `var()` — Variables CSS

```css
/* Definir */
:root {
  --border: #333333;
}

/* Usar */
border: 1px solid var(--border);
```

### `calc()` — Cálculos dinámicos

Útil cuando necesitas combinar unidades:

```css
/* Ancho total menos padding */
width: calc(100% - 4rem);

/* Espaciado proporcional */
margin-top: calc(5rem + 1vh);
```

### `gap` — Espaciado en flex/grid

```css
/* Reemplaza márgenes individuales */
.projects-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;  /* Espacio horizontal Y vertical entre items */
}

/* No necesitas esto: */
.project-card {
  margin-right: 1rem;     /* ❌ innecesario con gap */
  margin-bottom: 1rem;    /* ❌ innecesario con gap */
}
```

### `flex-wrap` — Items que bajan de línea

```css
/* Sin flex-wrap: los items se salen del contenedor */
/* Con flex-wrap: los items bajan cuando no caben */
.skills {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}
```

### `@media` — Responsive

```css
/* Mobile first: estilos base para móvil */
h1 {
  font-size: 1.25rem;
}

/* Tablet y desktop */
@media (min-width: 768px) {
  h1 {
    font-size: 2rem;
  }
}

/* O al revés: desktop first */
h1 {
  font-size: 2rem;
}

@media (max-width: 767px) {
  h1 {
    font-size: 1.25rem;
  }
}
```

### `transition` — Animaciones sutiles

```css
/* Tu patrón actual */
a {
  transition: all 0.3s;
}

/* Más específico (mejor rendimiento) */
a {
  transition: color 0.3s, margin-left 0.3s;
}
```

---

## 7. Tips para nuevas páginas (ProjectView, etc.)

### Al crear una nueva vista, empieza con esta estructura:

```vue
<template>
  <section class="page-section">
    <p class="sub-titulo">Proyecto</p>
    <h1>Nombre del Proyecto</h1>
    <p class="role">Descripción corta o tech stack</p>

    <div class="content" style="margin-top: 5rem;">
      <!-- Contenido principal -->
    </div>

    <div class="tags" style="margin-top: 5rem;">
      <span class="tag">Vue.js</span>
      <span class="tag">Node.js</span>
    </div>
  </section>
</template>

<style scoped>
.page-section {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;
}

/* Reutiliza las clases globales: .sub-titulo, .role, .tag */
/* No las redefinas en scoped */
</style>
```

### Reglas para mantener consistencia

1. **Usa las clases globales** (`.sub-titulo`, `.role`, `.tag`) en lugar de crear nuevas
2. **Respeta la jerarquía**: `sub-titulo` → `h1`/`h2` → `role` → contenido
3. **Mantén el ritmo vertical**: `margin-top: 5rem` entre secciones
4. **No introduzcas nuevos colores**: si necesitas un color que no existe, defínelo como variable en `:root`
5. **Los proyectos se muestran igual que en HomeView**: misma estructura de card, mismos tags, mismos links

### Patrón para lista de proyectos

```vue
<div class="projects-list">
  <article v-for="project in projects" :key="project.id" class="project-card">
    <h2>{{ project.name }}</h2>
    <p class="role">{{ project.description }}</p>
    <div class="project-tags">
      <span class="tag" v-for="tech in project.tech" :key="tech">{{ tech }}</span>
    </div>
    <a :href="project.url">Ver proyecto →</a>
  </article>
</div>

<style scoped>
.projects-list {
  display: flex;
  flex-direction: column;
  gap: 3rem;
}

.project-card {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.project-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}
</style>
```

---

## 8. Errores comunes a evitar

| Error | Por qué está mal | Solución |
|-------|-----------------|----------|
| **Mezclar `px` con `rem`** | Rompe la escalabilidad | Usa `rem` para todo excepto bordes de 1px |
| **Hardcodear colores** | Imposible cambiar tema, inconsistencia | Siempre usa `var(--nombre)` |
| **Crear clases duplicadas** | `.titulo` vs `.title` vs `.heading` | Reutiliza `.sub-titulo`, `h1`, `h2` |
| **Usar `!important`** | Hace el CSS imposible de mantener | Especificidad correcta > !important |
| **Márgenes inconsistentes** | Un sitio con `20px`, otro con `5rem` | Usa `5rem` entre secciones siempre |
| **Olvidar `flex-wrap`** | Los tags se salen de la pantalla en móvil | Siempre `flex-wrap: wrap` en listas |
| **No usar `gap`** | Márgenes desiguales entre items | `gap` reemplaza margin en flex/grid |
| **Añadir sombras o gradientes** | Rompe la estética minimalista | Manténlo plano y limpio |
| **Importar fuentes nuevas** | Carga lenta, inconsistencia visual | Solo Poppins (Merriweather no se usa) |
| **Ignorar `prefers-color-scheme`** | Se ve mal en modo claro | Prueba tu componente en ambos modos |

### Antes de hacer commit, pregúntate:

- [ ] ¿Mi nueva página se ve como si siempre hubiera existido en el portfolio?
- [ ] ¿Usé las mismas clases y variables que HomeView?
- [ ] ¿Se ve bien en móvil? (revisa con DevTools)
- [ ] ¿Se ve bien en modo claro? (`prefers-color-scheme: light`)
- [ ] ¿Hay algún color hardcodeado que deba ser una variable?

---

## Resumen rápido

```
FUENTE: Poppins (300, 400, 500, 600)
COLORES: Solo variables CSS (--text, --light_text, --muted, --border, --mega_light)
LAYOUT: max-width 1280px, centrado, flex column, padding 2rem
ESPACIADO: 5rem entre secciones, gap para items hermanos
INTERACTIVOS: border-bottom + hover con desplazamiento 0.25rem
UNIDADES: rem para todo, px solo para bordes de 1px
FILOSOFÍA: minimalista, oscuro, texto primero
```
