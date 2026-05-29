# 🎓 Guía de Estudio — GraphQL con Strawberry + FastAPI + Firestore

## Proyecto: AstroHunters 🚀

> Guía completa para el examen. Explicaciones en **español**, con fragmentos reales del código del proyecto.
> Pensada para alguien que sabe Python pero está aprendiendo GraphQL.

---

## 📚 TABLA DE CONTENIDOS

1. [Teoría Básica de GraphQL](#1-teoría-básica-de-graphql)
2. [Strawberry en Este Proyecto](#2-strawberry-en-este-proyecto)
3. [Union Types y Manejo de Errores](#3-union-types-y-manejo-de-errores)
4. [Autenticación y Autorización](#4-autenticación-y-autorización)
5. [Data Layer (Firestore)](#5-data-layer-firestore)
6. [DataLoader y N+1](#6-dataloader-y-n1)
7. [LazyType](#7-lazytype)
8. [Preguntas Típicas de Examen](#8-preguntas-típicas-de-examen-con-respuestas)
9. [Glosario Rápido](#9-glosario-rápido)

---

## 1. TEORÍA BÁSICA DE GRAPHQL

### ¿Qué es GraphQL?

GraphQL es un **lenguaje de consulta** (query language) para APIs. Fue creado por Facebook en 2012 y publicado en 2015. A diferencia de REST, donde el servidor decide qué datos devolver, **en GraphQL es el cliente quien decide exactamente qué campos quiere recibir**.

### GraphQL vs REST

| REST | GraphQL |
|------|---------|
| Múltiples endpoints (`/users`, `/posts`, `/comments`) | Un **único endpoint** (`/graphql`) |
| El servidor devuelve datos fijos (over-fetching o under-fetching) | El cliente pide exactamente lo que necesita |
| Cada endpoint devuelve una estructura predefinida | La respuesta tiene la misma forma que la query |
| Para datos relacionados, necesitas varias llamadas | Una sola query puede traer datos de múltiples recursos |

**Ejemplo práctico:** Imagina que quieres el perfil de un jugador Y su inventario. Con REST harías 2 llamadas:
- `GET /jugadors/abc123`
- `GET /jugadors/abc123/inventari`

Con GraphQL haces **1 sola query**:
```graphql
query {
  perfilJugador(uid: "abc123") {
    nickname
    nivell
    diners
  }
  inventariJugador(uid: "abc123") {
    nomItem
    raresa
  }
}
```

### Los 3 Pilares de GraphQL

#### 1. Schema (Esquema)
Es el **contrato** entre cliente y servidor. Define todos los tipos de datos disponibles y qué operaciones se pueden hacer. En Strawberry, el schema se construye a partir de las clases Python que decoras.

#### 2. Queries (Consultas)
Son las operaciones de **lectura**. Equivalen a un `GET` en REST. No modifican datos.
```graphql
query {
  perfilJugador(uid: "abc123") {
    nickname
  }
}
```

#### 3. Mutations (Mutaciones)
Son las operaciones de **escritura** (crear, modificar, eliminar). Equivalen a `POST`, `PUT`, `DELETE` en REST.
```graphql
mutation {
  crearPartida {
    ... on Partida {
      id
      mapa
    }
  }
}
```

> **Subscriptions:** GraphQL también tiene subscriptions (para datos en tiempo real vía WebSockets), pero **este proyecto no las usa**. Si te preguntan en el examen, existen pero no están implementadas aquí.

### Strawberry: Code-First Approach

Strawberry es una librería de GraphQL para Python. Usa un enfoque **"code-first"**: escribes código Python normal con decoradores, y Strawberry **genera automáticamente el schema de GraphQL** a partir de tu código.

Los decoradores clave que necesitas conocer:

#### `@strawberry.type` — Define un tipo de objeto GraphQL
```python
@strawberry.type
class Jugador:
    uid: str           # Campo escalar (string)
    nickname: str      # Campo escalar (string)
    nivell: int        # Campo escalar (entero)
    banejat: bool      # Campo escalar (booleano)
    diners: int = 0    # Campo con valor por defecto
```

Esto genera automáticamente este tipo en el schema GraphQL:
```graphql
type Jugador {
  uid: String!
  nickname: String!
  nivell: Int!
  banejat: Boolean!
  diners: Int!
}
```

#### `@strawberry.input` — Define un tipo de entrada (input) para mutaciones
Se usa cuando necesitas pasar múltiples parámetros a una mutación:
```python
@strawberry.input
class RegistrarJugadorInput:
    nickname: str
```

#### `@strawberry.field` — Define un campo de query
```python
@strawberry.field
def perfil_jugador(self, uid: str) -> Optional[Jugador]:
    """El cliente puede consultar este campo con: perfilJugador(uid: "abc123")"""
    ...
```

#### `@strawberry.mutation` — Define una mutación
```python
@strawberry.mutation
def crear_partida(self, info: strawberry.Info) -> Union[Partida, ErrorSensePartidaActiva, ...]:
    """El cliente puede ejecutar: mutation { crearPartida { ... } }"""
    ...
```

### Conversión automática de snake_case a camelCase

Un detalle **muy importante** de Strawberry: convierte automáticamente los nombres de campos Python (snake_case) a GraphQL (camelCase).

| Python (snake_case) | GraphQL (camelCase) |
|---|---|
| `perfil_jugador` | `perfilJugador` |
| `crear_partida` | `crearPartida` |
| `sala_actual` | `salaActual` |
| `jugador_id` | `jugadorId` |
| `baixes_acumulades` | `baixesAcumulades` |
| `taula_classificacio` | `taulaClassificacio` |
| `registrar_jugador` | `registrarJugador` |
| `donar_monedes` | `donarMonedes` |

Esto es automático. El cliente SIEMPRE usa camelCase, el backend SIEMPRE usa snake_case. No los mezcles.

---

## 2. STRAWBERRY EN ESTE PROYECTO

### Cómo se construye el Schema (schema.py)

El proyecto sigue un patrón de arquitectura llamado **DDD (Domain-Driven Design)** con dos dominios separados:

- `app/jugadors/` — Todo lo relacionado con perfiles de jugador
- `app/partides/` — Todo lo relacionado con partidas, salas, puntuaciones

Cada dominio tiene 3 archivos:
- `types.py` — Tipos GraphQL (objetos, inputs, errores)
- `queries.py` — Consultas (lectura)
- `mutations.py` — Mutaciones (escritura)

El **schema central** (`app/schema.py`) une todo usando **herencia múltiple** (patrón Mixin):

```python
# app/schema.py
import strawberry
from app.jugadors.queries import JugadorsQuery
from app.jugadors.mutations import JugadorsMutation
from app.partides.queries import PartidesQuery
from app.partides.mutations import PartidesMutation

@strawberry.type
class Query(JugadorsQuery, PartidesQuery):
    """Hereda TODOS los campos de ambos dominios"""
    pass

@strawberry.type
class Mutation(JugadorsMutation, PartidesMutation):
    """Hereda TODAS las mutaciones de ambos dominios"""
    pass

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

**¿Qué significa esto?** Que la clase `Query` hereda todos los `@strawberry.field` de `JugadorsQuery` Y `PartidesQuery`. El schema final incluye:
- `perfilJugador`, `inventariJugador` (de jugadors)
- `partidaActiva`, `salesPartida`, `taulaClassificacio` (de partides)

### El GraphQLRouter en main.py

En FastAPI, se monta el endpoint GraphQL así:

```python
# main.py
from fastapi import FastAPI, Request
from strawberry.fastapi import GraphQLRouter
from app.auth import get_auth_context
from app.schema import schema

app = FastAPI()

async def get_context(request: Request) -> dict:
    """Esta función se ejecuta en CADA petición GraphQL.
    Extrae el JWT del header Authorization y lo verifica.
    El resultado se inyecta en info.context."""
    auth = await get_auth_context(request)
    return {"auth": auth, "request": request}

graphql_app = GraphQLRouter(schema, context_getter=get_context)
app.include_router(graphql_app, prefix="/graphql")
```

**Punto clave:** El `context_getter` se ejecuta **antes de cada resolver**. Inyecta `auth` y `request` en `info.context`. Todos los resolvers acceden a estos datos para saber quién está haciendo la petición.

### El objeto `info.context`

Dentro de cualquier resolver (query o mutation), `info.context` es un diccionario que contiene:

| Clave | Tipo | Contenido |
|-------|------|-----------|
| `"auth"` | `AuthContext \| None` | UID, email y rol del usuario autenticado. `None` si no hay token |
| `"request"` | `Request` | El objeto Request de FastAPI (rara vez se usa directamente) |

Ejemplo de uso en un resolver:
```python
@strawberry.field
def partida_activa(self, info: strawberry.Info) -> Optional[Partida]:
    auth: AuthContext = info.context.get("auth")
    if auth is None:
        return None  # No autenticado, no hay partida que mostrar
    # ... buscar partida por auth.uid
```

---

## 3. UNION TYPES Y MANEJO DE ERRORES

### El patrón: ¿Por qué Union Types en vez de excepciones?

En una API REST tradicional, cuando algo falla, lanzas una excepción HTTP (404, 401, 500). En GraphQL, **no se lanzan excepciones** para errores de negocio. En su lugar, las mutaciones devuelven un **Union Type**: o bien el resultado exitoso, o bien uno de varios tipos de error.

**¿Por qué es mejor?** Porque el cliente recibe errores **tipados** (con campos específicos como `missatge`) que puede manejar con `... on` en GraphQL. No recibe un error genérico 500 que no sabe de dónde viene.

### Cómo funciona

Una mutación declara su retorno como `Union[TipoExito, Error1, Error2, ...]`:

```python
@strawberry.mutation
def registrar_jugador(
    self, info: strawberry.Info, input: RegistrarJugadorInput
) -> Union[Jugador, ErrorAutenticacio, ErrorJugadorJaExisteix, ErrorJugadorBanejat]:
    auth = info.context.get("auth")
    if auth is None:
        return ErrorAutenticacio(missatge="Token JWT no vàlid")  # ← RAMA DE ERROR

    doc_ref = db.collection("jugadors").document(auth.uid)
    if doc_ref.get().exists:
        return ErrorJugadorJaExisteix(missatge="El jugador ja està registrat")  # ← RAMA DE ERROR

    # ... si todo va bien:
    return Jugador(uid=auth.uid, nickname=input.nickname, nivell=1, banejat=False, diners=0)  # ← RAMA ÉXITO
```

### Todos los tipos de error del proyecto

| Tipo de Error | Archivo | Significado |
|---|---|---|
| `ErrorCredencials` | `jugadors/types.py:29` | Email/contraseña incorrectos o error al crear cuenta |
| `ErrorAutenticacio` | `jugadors/mutations.py:31` | Token JWT no proporcionado o inválido |
| `ErrorJugadorBanejat` | `jugadors/types.py:49` | El jugador está baneado (nickname ofensivo) |
| `ErrorJugadorNoTrobat` | `jugadors/types.py:44` | El UID del jugador no existe en Firestore |
| `ErrorJugadorJaExisteix` | `jugadors/mutations.py:36` | Ya existe un perfil de jugador con ese UID |
| `ErrorItemNoDisponible` | `jugadors/types.py:34` | El item no está en la sala actual o no es una tienda |
| `ErrorDinersInsufficients` | `jugadors/types.py:39` | No tienes suficientes monedas para comprar |
| `ErrorSensePartidaActiva` | `partides/types.py:74` | No tienes partida en curso o no hay sesión |
| `ErrorPartidaJaActiva` | `partides/types.py:79` | Ya tienes una partida en curso |
| `ErrorMazmorraCompletada` | `partides/types.py:84` | Ya has llegado al boss |

### Cómo el cliente discrimina con `... on`

En GraphQL, el cliente usa **inline fragments** (`... on TipoError`) para discriminar qué tipo recibió:

```graphql
mutation {
  registrarJugador(input: { nickname: "Hero" }) {
    __typename              # ← Siempre inclúyelo para debug
    ... on Jugador {
      uid
      nickname
      nivell
    }
    ... on ErrorAutenticacio {
      missatge
    }
    ... on ErrorJugadorJaExisteix {
      missatge
    }
    ... on ErrorJugadorBanejat {
      missatge
    }
  }
}
```

**Explicación:** `__typename` es un campo mágico de GraphQL que devuelve el nombre del tipo concreto (p.ej. `"Jugador"` o `"ErrorAutenticacio"`). Los fragmentos `... on X` solo se ejecutan si el resultado es de tipo `X`.

**En código JavaScript:**
```javascript
const result = await graphqlClient.mutate({ ... });
if (result.registrarJugador.__typename === "Jugador") {
  console.log("Éxito:", result.registrarJugador.nickname);
} else {
  console.log("Error:", result.registrarJugador.missatge);
}
```

---

## 4. AUTENTICACIÓN Y AUTORIZACIÓN

### Flujo completo

```
1. REGISTRO/LOGIN
   Cliente → mutation { registrar(email, password, nickname) }
   Backend → Firebase Auth (signUp/signIn) → devuelve token JWT
   Cliente recibe → AuthPayload { token, jugador }

2. PETICIONES AUTENTICADAS
   Cliente → POST /graphql
             Header: Authorization: Bearer <token_jwt>
   
3. VERIFICACIÓN (CADA PETICIÓN)
   main.py → get_context()
           → get_auth_context(request)
           → firebase_auth.verify_id_token(token)
           → AuthContext(uid, email, rol)
           → inyectado en info.context["auth"]

4. RESOLVER
   mutation → auth = info.context.get("auth")
            → if auth is None: return ErrorAutenticacio(...)
            → usar auth.uid para operaciones
```

### get_auth_context() en detalle

```python
# app/auth.py
from firebase_admin import auth as firebase_auth

class AuthContext:
    def __init__(self, uid: str, email: Optional[str] = None, rol: str = "jugador"):
        self.uid = uid
        self.email = email
        self.rol = rol

async def get_auth_context(request: Request) -> Optional[AuthContext]:
    auth_header = request.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        return None  # ← No hay token → petición anónima

    token = auth_header[len("Bearer "):]
    try:
        decoded = firebase_auth.verify_id_token(token)  # ← Firebase verifica el JWT
        uid = decoded["uid"]
        email = decoded.get("email", "")
        # DETERMINACIÓN DE ROL por dominio de email
        rol = "admin" if email.endswith("@astrohunters.com") else "jugador"
        return AuthContext(uid=uid, email=email, rol=rol)
    except Exception:
        return None  # ← Token inválido → petición anónima
```

### ¿Por qué `registrar_jugador` usa el UID del JWT y no uno pasado por parámetro?

**Respuesta de seguridad:** Para evitar que un usuario malicioso pueda crear un perfil con el UID de otro jugador.

```python
@strawberry.mutation
def registrar_jugador(
    self, info: strawberry.Info, input: RegistrarJugadorInput
) -> Union[Jugador, ErrorAutenticacio, ...]:
    auth: AuthContext = info.context.get("auth")
    # ✅ El UID viene del token JWT VERIFICADO, no del input del cliente
    doc_ref = db.collection("jugadors").document(auth.uid)
    # ❌ Si usáramos input.uid, un atacante podría suplantar a cualquiera
```

**Dato importante:** El `RegistrarJugadorInput` solo tiene el campo `nickname`. El UID **no se pasa nunca** por parámetro, siempre se extrae del token JWT verificado.

### ¿Qué queries/mutations requieren auth y cuáles son públicas?

| Operación | ¿Requiere auth? | ¿Qué pasa sin auth? |
|---|---|---|
| `perfilJugador(uid)` | ❌ Pública | Funciona normal |
| `inventariJugador(uid)` | ❌ Pública | Funciona normal |
| `taulaClassificacio` | ❌ Pública | Funciona normal |
| `login` / `registrar` | ❌ Públicas | Son justamente para obtener el token |
| `partidaActiva` | ✅ Requiere auth | Retorna `None` |
| `salesPartida` | ✅ Requiere auth | Retorna `None` |
| `registrarJugador` | ✅ Requiere auth | Retorna `ErrorAutenticacio` |
| `crearPartida` | ✅ Requiere auth | Retorna `ErrorSensePartidaActiva` |
| `crearSala` | ✅ Requiere auth | Retorna `ErrorSensePartidaActiva` |
| `finalitzarPartida` | ✅ Requiere auth | Retorna `ErrorSensePartidaActiva` |
| `donarMonedes` | ✅ Requiere auth | Retorna `ErrorAutenticacio` |
| `afegirDiners` | ✅ Requiere auth | Retorna `ErrorAutenticacio` |
| `atorgarItem` | ✅ Requiere auth | Retorna `ErrorAutenticacio` |

### Patrón de verificación de auth en resolvers

Todas las mutaciones que requieren auth siguen el mismo patrón:

```python
@strawberry.mutation
def alguna_mutacion(self, info: strawberry.Info, ...) -> Union[...]:
    # PASO 1: Extraer auth del contexto
    auth: AuthContext = info.context.get("auth")

    # PASO 2: Verificar que existe (token válido)
    if auth is None:
        return ErrorAutenticacio(missatge="Token JWT no vàlid o no proporcionat")

    # PASO 3: Usar auth.uid para las operaciones
    doc_ref = db.collection("jugadors").document(auth.uid)
    ...
```
﻿
## 5. DATA LAYER (FIRESTORE)

### Sin ORM — Acceso directo a Firestore

Este proyecto **no usa SQL, no usa SQLAlchemy, no usa ORM**. Todo es acceso directo a Firestore (base de datos NoSQL de Google Cloud).

La inicialización es simple:
```python
# app/firebase_conf.py
import firebase_admin
from firebase_admin import credentials, firestore

cred = credentials.Certificate("credentials.json")
firebase_admin.initialize_app(cred)
db = firestore.client()  # ← Este es el objeto que se usa en TODOS los archivos
```

### Colecciones y subcolecciones

```
Firestore
├── jugadors/                          ← Colección principal
│   ├── {uid}/                         ← Documento (ID = UID de Firebase Auth)
│   │   ├── nickname: "Hero"
│   │   ├── nivell: 5
│   │   ├── banejat: false
│   │   ├── diners: 250
│   │   └── inventari/                 ← Subcolección
│   │       ├── {autoID}/
│   │       │   ├── nom_item: "Poció de Vida"
│   │       │   └── raresa: "Comú"
│   │       └── ...
│   └── ...
├── partides/                          ← Colección principal
│   ├── {autoID}/
│   │   ├── jugador_id: "abc123"
│   │   ├── mapa: "Mazmorra Profunda"
│   │   ├── estat: "En curs"
│   │   ├── sala_actual: 3
│   │   ├── boss_posicio: 10
│   │   ├── botiga_posicio: 6
│   │   ├── baixes_acumulades: 15
│   │   └── sales/                     ← Subcolección
│   │       ├── {autoID}/
│   │       │   ├── index: 1
│   │       │   ├── tipus: 3
│   │       │   ├── entitats: [1, 2]
│   │       │   ├── completada: false
│   │       │   ├── door_location: "up"
│   │       │   └── items: [...]       ← Solo si tipus==6 (botiga)
│   │       └── ...
│   └── ...
└── puntuacions/                       ← Colección principal
    ├── {autoID}/
    │   ├── jugador_id: "abc123"
    │   ├── puntuacio: 850
    │   ├── baixes: 15
    │   ├── partida_id: "xyz"
    │   └── data: Timestamp
    └── ...
```

### Operaciones CRUD con Firestore

#### Lectura de un documento
```python
# Obtener un jugador por UID
doc = db.collection("jugadors").document(uid).get()
if doc.exists:
    data = doc.to_dict()
    nickname = data["nickname"]
```

#### Escritura (crear o sobrescribir)
```python
# Crear/sobrescribir un documento
db.collection("jugadors").document(uid).set({
    "nickname": "Hero",
    "nivell": 1,
    "banejat": False,
    "diners": 0,
})
```

#### Actualización parcial
```python
# Solo modificar campos específicos
db.collection("jugadors").document(uid).update({"diners": 500})
```

#### Streaming (leer múltiples documentos de una consulta)
```python
# Obtener todos los documentos de una subcolección
docs = db.collection("jugadors").document(uid).collection("inventari").stream()
for doc in docs:
    data = doc.to_dict()
    print(data["nom_item"])
```

#### Añadir documento con ID automático
```python
# Firestore genera un ID único
doc_ref = db.collection("puntuacions").add({
    "jugador_id": auth.uid,
    "puntuacio": 850,
    "baixes": 15,
    "data": SERVER_TIMESTAMP,
})
```

### Consultas con filtros (FieldFilter)

Para hacer WHERE en Firestore, se usa `FieldFilter`:

```python
from google.cloud.firestore_v1.base_query import FieldFilter

# Buscar la partida activa del jugador
partides = list(db.collection("partides")
    .where(filter=FieldFilter("jugador_id", "==", auth.uid))    # WHERE jugador_id = ?
    .where(filter=FieldFilter("estat", "==", "En curs"))        # AND estat = 'En curs'
    .stream())
```

### Ordenación y paginación

```python
# Ranking con paginación
docs = db.collection("puntuacions") \
    .order_by("puntuacio", direction="DESCENDING") \  # ORDER BY puntuacio DESC
    .offset(offset) \                                   # OFFSET
    .limit(limit) \                                     # LIMIT
    .stream()
```

### SERVER_TIMESTAMP

Para guardar la fecha/hora del servidor (no la del cliente):
```python
from google.cloud.firestore_v1 import SERVER_TIMESTAMP

partida_data = {
    "data_creacio": SERVER_TIMESTAMP,  # ← Firebase pone la hora del servidor
    ...
}
```

---

## 6. DATALOADER Y N+1

### ¿Qué es el problema N+1?

Es uno de los problemas de rendimiento más comunes en GraphQL. Ocurre cuando tienes una consulta que devuelve N objetos, y para cada uno necesitas hacer otra consulta a la base de datos.

**Ejemplo concreto en AstroHunters:**

El ranking (`taulaClassificacio`) devuelve 50 puntuaciones. Cada `Puntuacio` tiene un campo `jugador` que devuelve el `Jugador` (con su `nickname`):

```graphql
query {
  taulaClassificacio(limit: 50) {
    puntuacio
    jugador {        # ← Para cada puntuación, necesitamos el jugador
      nickname
    }
  }
}
```

**Sin DataLoader (problema N+1):**
1. 1 consulta para obtener 50 puntuaciones → **1 petición a Firestore**
2. Para cada una de las 50 puntuaciones, resolver `jugador` hace `db.collection("jugadors").document(uid).get()` → **50 peticiones individuales a Firestore**
3. **Total: 51 peticiones** 😱

**Con DataLoader:**
1. 1 consulta para las 50 puntuaciones → **1 petición**
2. DataLoader agrupa los 50 UIDs y hace **1 sola petición batch** (`db.get_all()`) → **1 petición**
3. **Total: 2 peticiones** 🚀

### Cómo funciona en este proyecto

```python
# app/loaders.py
from strawberry.dataloader import DataLoader
from app.firebase_conf import db

def _batch_load_jugadors(ids: list[str]) -> list[dict | None]:
    """Carrega múltiples jugadors en UNA sola petición a Firestore."""
    if not ids:
        return []

    # Creamos referencias para cada UID
    referencies = [db.collection("jugadors").document(uid) for uid in ids]

    # get_all() = UNA sola llamada batch a Firestore (BatchGetDocuments)
    documents = db.get_all(referencies)

    # Indexamos por ID para mantener el orden de retorno
    resultat = {}
    for doc in documents:
        if doc.exists:
            resultat[doc.id] = doc.to_dict()
        else:
            resultat[doc.id] = None

    # Devolvemos en el MISMO orden que los IDs de entrada
    return [resultat.get(uid) for uid in ids]

# El DataLoader se instancia UNA vez como singleton
jugador_loader = DataLoader(load_fn=_batch_load_jugadors)
```

### Cómo se usa en Puntuacio.jugador

```python
# app/partides/types.py (líneas 42-70)
@strawberry.type
class Puntuacio:
    id: str
    jugador_id: str
    puntuacio: int
    baixes: int
    data: str

    @strawberry.field
    async def jugador(self) -> JugadorRef | None:
        """Resuelve el nickname usando el DataLoader (batch).
        
        Cuando el cliente pide 50 puntuaciones con nickname,
        el DataLoader hace 1 petición a Firestore en lugar de 50.
        """
        from app.jugadors.types import Jugador
        from app.loaders import jugador_loader

        # load() NO ejecuta la consulta inmediatamente.
        # Acumula los IDs y los ejecuta en batch cuando
        # el event loop tiene un respiro.
        data = await jugador_loader.load(self.jugador_id)
        if data is None:
            return None

        return Jugador(
            uid=self.jugador_id,
            nickname=data["nickname"],
            nivell=data["nivell"],
            banejat=data.get("banejat", False),
            diners=data.get("diners", 0),
        )
```

### El `load_fn` en detalle

La función que pasas a `DataLoader(load_fn=...)` debe cumplir este contrato:

- **Entrada:** `list[Key]` — Una lista de claves a cargar (en este caso, UIDs de jugador)
- **Salida:** `list[Value | None]` — Una lista de resultados en el **mismo orden** que las claves de entrada
- **Comportamiento:** Carga TODAS las claves en una sola operación batch

El DataLoader internamente:
1. Acumula llamadas individuales a `.load(key)`
2. Deduplica claves repetidas (si dos puntuaciones son del mismo jugador, solo lo carga una vez)
3. Ejecuta `load_fn` una sola vez con todas las claves
4. Distribuye los resultados a cada `.load()` pendiente

---

## 7. LAZYTYPE

### ¿Por qué se necesita?

En una arquitectura con múltiples dominios, es común que un dominio necesite referenciar tipos de otro dominio. Cuando dos módulos se importan mutuamente, se produce una **dependencia circular** (circular import) que Python no puede resolver:

```
partides/types.py → necesita importar Jugador de jugadors/types.py
jugadors/types.py → (potencialmente) necesita importar algo de partides/types.py
```

Strawberry resuelve esto con `LazyType`, que permite referenciar un tipo por su **nombre** y **módulo** como strings, en lugar de importarlo directamente.

### Cómo funciona en este proyecto

```python
# app/partides/types.py (línea 4-8)
from strawberry.types.lazy_type import LazyType

# Referencia "perezosa" a Jugador — NO es un import real
# Solo guarda el nombre del tipo y su módulo como strings
JugadorRef = LazyType(type_name="Jugador", module="app.jugadors.types")

@strawberry.type
class Puntuacio:
    id: str
    jugador_id: str
    puntuacio: int
    baixes: int
    data: str

    @strawberry.field
    async def jugador(self) -> JugadorRef | None:  # ← Usa la referencia lazy
        """Resol el nickname del jugador usant el DataLoader (batch)."""
        from app.jugadors.types import Jugador  # ← Import REAL, pero DENTRO de la función
        from app.loaders import jugador_loader

        data = await jugador_loader.load(self.jugador_id)
        if data is None:
            return None

        return Jugador(uid=self.jugador_id, nickname=data["nickname"], ...)
```

**Explicación de las dos técnicas usadas:**

1. **LazyType en la anotación de tipo** (`-> JugadorRef | None`): Le dice a Strawberry "este campo devolverá un tipo llamado `Jugador` del módulo `app.jugadors.types`". Strawberry resuelve el tipo cuando construye el schema, no en tiempo de importación del módulo.

2. **Import dentro de la función** (`from app.jugadors.types import Jugador`): El import real de `Jugador` se hace dentro de la función, no a nivel de módulo. Esto evita el circular import porque el código no se ejecuta hasta que se llama al resolver.

### Dónde se usa en el proyecto

Solo en un sitio: `app/partides/types.py`, línea 8. El tipo `Puntuacio` (del dominio `partides`) necesita referenciar al tipo `Jugador` (del dominio `jugadors`).

---

﻿
## 8. PREGUNTAS TÍPICAS DE EXAMEN (CON RESPUESTAS)

### Pregunta 1 — Teoría
**"¿Qué diferencia hay entre Query y Mutation en GraphQL?"**

**Respuesta:** 
- **Query** es para operaciones de **lectura** (equivalente a GET en REST). No modifica datos en el servidor. Ejemplo: `perfilJugador`, `taulaClassificacio`.
- **Mutation** es para operaciones de **escritura** (equivalente a POST/PUT/DELETE en REST). Modifica datos. Ejemplo: `crearPartida`, `donarMonedes`.
- En Strawberry, las queries se declaran con `@strawberry.field` dentro de una clase que hereda de `Query`, y las mutaciones con `@strawberry.mutation` dentro de una clase que hereda de `Mutation`.

---

### Pregunta 2 — Teoría
**"¿Qué es el problema N+1 y cómo lo resuelve este proyecto?"**

**Respuesta:**
El problema N+1 ocurre cuando haces 1 consulta para obtener N objetos, y luego para cada objeto haces otra consulta individual. En AstroHunters, `taulaClassificacio` devuelve hasta 50 `Puntuacio`. Si cada una tuviera que resolver `jugador.nickname` con una consulta individual a Firestore, serían **51 consultas totales** (1 + 50).

El proyecto lo resuelve con un **DataLoader** (`app/loaders.py`). La función `_batch_load_jugadors` usa `db.get_all()` para cargar todos los jugadores en **1 sola consulta batch**. El DataLoader automáticamente agrupa (batchea) y deduplica las peticiones. Resultado: solo **2 consultas** totales.

---

### Pregunta 3 — Lectura de código
**"¿Qué devuelve esta query y qué operaciones de Firestore se ejecutan?"**

```graphql
query {
  perfilJugador(uid: "abc123") {
    nickname
    nivell
    diners
  }
}
```

**Respuesta:**
Devuelve un objeto `Jugador` con los campos `nickname`, `nivell` y `diners` del jugador con UID `"abc123"`.

**Operaciones Firestore:** Exactamente 1 → `db.collection("jugadors").document("abc123").get()`

**Código que se ejecuta:**
```python
@strawberry.field
def perfil_jugador(self, uid: str) -> Optional[Jugador]:
    doc = db.collection("jugadors").document(uid).get()
    if not doc.exists:
        return None
    data = doc.to_dict()
    return Jugador(uid=uid, nickname=data["nickname"], nivell=data["nivell"],
                   banejat=data.get("banejat", False), diners=data.get("diners", 0))
```

Si el jugador no existe, devuelve `null` (el equivalente GraphQL de `None` en Python).

---

### Pregunta 4 — Lectura de código
**"Dada esta query, ¿cuántas peticiones se hacen a Firestore?"**

```graphql
query {
  taulaClassificacio(limit: 10) {
    puntuacio
    baixes
    jugador {
      nickname
    }
  }
}
```

**Respuesta: 2 peticiones.**

1. **1 petición** para la query de puntuaciones: `db.collection("puntuacions").order_by("puntuacio", direction="DESCENDING").limit(10).stream()`
2. **1 petición batch** del DataLoader: `db.get_all([ref1, ref2, ..., ref10])` para cargar los 10 jugadores de una vez.

Gracias al DataLoader, no son 11 peticiones (1 + 10 individuales), sino solo 2.

---

### Pregunta 5 — Escenario
**"Un jugador quiere ver el ranking completo. ¿Qué query debe ejecutar y qué sucede en el backend paso a paso?"**

**Respuesta:**
El jugador debe ejecutar:
```graphql
query {
  taulaClassificacio(limit: 10, offset: 0) {
    puntuacio
    baixes
    jugador {
      nickname
    }
  }
}
```

**Qué sucede en el backend paso a paso:**
1. `PartidesQuery.taula_classificacio()` consulta la colección `puntuacions` con ordenación descendente por puntuación y paginación.
2. Por cada puntuación, se crea un objeto `Puntuacio` con `jugador_id` pero sin resolver aún el campo `jugador`.
3. Cuando el cliente pide `jugador { nickname }`, se dispara el resolver `Puntuacio.jugador()` para CADA puntuación.
4. Cada llamada a `jugador_loader.load(uid)` NO ejecuta la consulta — solo encola el UID.
5. El DataLoader detecta que tiene N UIDs pendientes y llama a `_batch_load_jugadors(ids)` UNA SOLA VEZ.
6. `_batch_load_jugadors` ejecuta `db.get_all(referencies)` — 1 sola petición batch.
7. Los resultados se distribuyen a cada `Puntuacio.jugador` que los estaba esperando.
8. La query es **pública** (no requiere autenticación).

---

### Pregunta 6 — Debugging
**"¿Por qué esta mutación devuelve ErrorAutenticacio si el usuario acaba de hacer login?"**

```graphql
mutation {
  registrarJugador(input: { nickname: "Hero" }) {
    ... on ErrorAutenticacio {
      missatge
    }
  }
}
```

**Respuesta:**
Lo más probable es que el cliente **no esté enviando el token JWT** en el header `Authorization`. Después del login, el cliente recibe un `AuthPayload` con un `token`. Ese token debe enviarse en todas las peticiones posteriores como header HTTP:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6...
```

El flujo correcto es:
1. Ejecutar `login(email, password)` o `registrar(email, password, nickname)` → obtener `token` del `AuthPayload`
2. Guardar el token en el cliente (localStorage, variable, etc.)
3. Enviarlo en el header `Authorization: Bearer <token>` en **cada** petición posterior

Si el header no está presente, está mal formado, o el token ha caducado, `get_auth_context()` retorna `None`, y `registrarJugador` devuelve `ErrorAutenticacio`.

---

### Pregunta 7 — Diseño
**"¿Cómo añadirías un nuevo campo `vides` (vidas, tipo Int, valor por defecto 3) al tipo Jugador? Enumera todos los archivos que necesitas modificar."**

**Respuesta:**
Hay que modificar **múltiples archivos** porque el tipo Jugador se construye en varios sitios:

**1. `app/jugadors/types.py` — Añadir el campo al tipo:**
```python
@strawberry.type
class Jugador:
    uid: str
    nickname: str
    nivell: int
    banejat: bool
    diners: int = 0
    vides: int = 3  # ← Nuevo campo
```

**2. `app/jugadors/queries.py` — Incluir el campo al construir Jugador desde Firestore:**
```python
return Jugador(
    uid=uid,
    nickname=data["nickname"],
    nivell=data["nivell"],
    banejat=data.get("banejat", False),
    diners=data.get("diners", 0),
    vides=data.get("vides", 3),  # ← Nuevo
)
```

**3. `app/auth_service.py` — Añadir al crear un jugador nuevo:**
```python
db.collection("jugadors").document(uid).set({
    "nickname": nickname,
    "nivell": 1,
    "banejat": False,
    "diners": 0,
    "vides": 3,  # ← Nuevo
})
```

**4. `app/jugadors/mutations.py` — Actualizar la función helper `_a_jugador()`**
**5. `app/partides/types.py` — Actualizar `Puntuacio.jugador`**
**6. Las mutaciones `registrar` y `registrarJugador`** — También construyen Jugador manualmente.

---

### Pregunta 8 — Seguridad
**"¿Por qué `registrar_jugador` usa el UID del token JWT y no uno pasado por parámetro en el input?"**

**Respuesta:**
Por seguridad. Si el UID se pasara como parámetro en el input:

```python
# ❌ INSEGURO - El cliente podría enviar cualquier UID
@strawberry.input
class RegistrarJugadorInput:
    uid: str       # ← Esto NO existe en el código real
    nickname: str
```

Un atacante podría crear un perfil con el UID de otro jugador y suplantarlo completamente. En su lugar, el código real extrae el UID del token JWT ya verificado por Firebase:

```python
# ✅ SEGURO - El UID viene del token JWT verificado
auth: AuthContext = info.context.get("auth")
doc_ref = db.collection("jugadors").document(auth.uid)  # UID del token, no del input
```

El `RegistrarJugadorInput` solo tiene `nickname`. El UID es **implícito** (viene del token verificado), no **explícito** (no se pasa en el body de la petición).

---

﻿
### Pregunta 9 — Diseño
**"¿Cómo funciona el sistema anti-ofensivo de nicknames? Explica el flujo completo."**

**Respuesta:**
Hay una **lista negra** (blacklist) definida en `app/jugadors/mutations.py` (línea 22):

```python
LLA_NEGRA = {"israel", "hitler", "putin", "nazi", "terrorista"}
```

La función `_nickname_es_ofensiu()` comprueba si el nickname está en la lista:
```python
def _nickname_es_ofensiu(nickname: str) -> bool:
    return nickname.strip().lower() in LLA_NEGRA
```

**Flujo cuando un jugador se registra con un nickname ofensivo:**

1. **En `registrar` (registro con email/password):**
   - Se crea la cuenta en Firebase Auth normalmente
   - Se crea el perfil en Firestore con `banejat: True`
   - La mutación devuelve `ErrorCredencials` con mensaje "Aquest nickname no esta permes"

2. **En `registrarJugador` (registro post-login):**
   - Se comprueba el nickname contra la lista negra
   - Si es ofensivo, se crea el perfil con `banejat: True`
   - La mutación devuelve `ErrorJugadorBanejat`

3. **Bloqueo en otras mutaciones:**
   - `crearPartida` y `finalitzarPartida` llaman a `_jugador_esta_banejat(auth.uid)` que consulta Firestore
   - Si `banejat == True`, devuelven `ErrorJugadorBanejat` y bloquean la operación

El baneo es **automático e inmediato** al momento del registro.

---

### Pregunta 10 — Código
**"¿Qué hace esta mutación y qué condiciones pueden hacer que falle?"**

```graphql
mutation {
  atorgarItem(salaId: "xyz", nomItem: "Poció de Vida") {
    ... on ItemInventari { nomItem raresa }
    ... on ErrorAutenticacio { missatge }
    ... on ErrorItemNoDisponible { missatge }
    ... on ErrorDinersInsufficients { missatge }
  }
}
```

**Respuesta:**
Esta mutación (`atorgar_item` en `app/jugadors/mutations.py:191`) permite comprar un item de la botiga dentro de una mazmorra.

**Flujo de la mutación:**
1. Verifica autenticación (JWT)
2. Busca la partida activa del jugador (`estat == "En curs"`)
3. Verifica que la sala especificada existe y es de tipo botiga (`tipus == 6`)
4. Busca el item por nombre dentro de los items de la sala
5. Comprueba que el jugador tiene suficientes monedas
6. Descuenta el precio de las monedas del jugador
7. Añade el item a la subcolección `jugadors/{uid}/inventari/`

**Condiciones de fallo:**

| Condición | Error retornado |
|---|---|
| No hay token JWT o es inválido | `ErrorAutenticacio` |
| El jugador no tiene partida activa | `ErrorItemNoDisponible` ("No tens cap partida activa") |
| La sala especificada no existe | `ErrorItemNoDisponible` ("Sala no trobada") |
| La sala no es una botiga (`tipus != 6`) | `ErrorItemNoDisponible` ("Aquesta sala no és una botiga") |
| El item no está en esa botiga | `ErrorItemNoDisponible` ("L'item no està disponible en aquesta botiga") |
| El jugador no tiene suficientes monedas | `ErrorDinersInsufficients` ("Diners insuficients. Tens X, necessites Y") |

**Si todo va bien:** Se retorna `ItemInventari` con `id`, `nomItem` y `raresa`.

---

### Pregunta 11 — Arquitectura
**"Explica cómo se ensambla el schema de GraphQL usando herencia múltiple y por qué se hace así."**

**Respuesta:**
El schema se construye en `app/schema.py` usando el patrón **Mixin** con herencia múltiple:

```python
@strawberry.type
class Query(JugadorsQuery, PartidesQuery):
    pass

@strawberry.type
class Mutation(JugadorsMutation, PartidesMutation):
    pass

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

**¿Por qué se hace así?** Permite separar el código en **dominios independientes** (DDD - Domain-Driven Design). Cada dominio (`jugadors`, `partides`) tiene sus propios types, queries y mutations en archivos separados. La herencia múltiple los "fusiona" en un solo schema sin que los dominios tengan que conocerse entre sí directamente.

**Ventajas concretas:**
- Cada dominio se desarrolla de forma aislada
- Si añades un nuevo dominio (ej: `clans/`), solo creas su carpeta con `types.py`, `queries.py`, `mutations.py` y lo añades a la herencia
- El código está organizado y es fácil de navegar
- Los tests pueden probar cada dominio por separado

---

### Pregunta 12 — Depuración
**"Un jugador ejecuta `crearPartida` y recibe `ErrorPartidaJaActiva`. ¿Qué significa exactamente y qué debería hacer?"**

**Respuesta:**
Significa que el jugador **ya tiene una partida en curso** (`estat == "En curs"`). El código en `crear_partida` comprueba esto:

```python
partida = _trobar_partida_activa(auth.uid)
if partida:
    return ErrorPartidaJaActiva(
        missatge="Ja tens una partida en curs. Finalitza-la abans de crear-ne una altra"
    )
```

La función helper `_trobar_partida_activa` busca en Firestore:
```python
def _trobar_partida_activa(uid: str):
    partides = list(db.collection("partides")
        .where(filter=FieldFilter("jugador_id", "==", uid))
        .where(filter=FieldFilter("estat", "==", "En curs"))
        .stream())
    return partides[0] if partides else None
```

**Lo que debe hacer el jugador:** Ejecutar `finalitzarPartida` para cerrar la partida actual, y luego `crearPartida` para empezar una nueva.

---

### Pregunta 13 — Cálculo y lógica
**"¿Cómo se calcula la puntuación al finalizar una partida? Explica la fórmula y dónde se guarda."**

**Respuesta:**
La puntuación se calcula en `finalitzar_partida` (`app/partides/mutations.py:264`):

```python
baixes_totals = partida_data.get("baixes_acumulades", 0)
sala_final = partida_data["sala_actual"]
puntuacio_total = baixes_totals * 10 + sala_final * 50
```

**Fórmula:** `puntuación = (enemigos derrotados × 10) + (sala alcanzada × 50)`

- Cada enemigo derrotado = **10 puntos**
- Cada sala superada = **50 puntos**

**¿Cómo se acumulan las bajas?** Los enemigos se acumulan automáticamente al avanzar de sala en `crear_sala` (`app/partides/mutations.py:219`):
```python
baixes_acumulades = partida_data.get("baixes_acumulades", 0)
enemics_derrotats = _comptar_baixes_sala(partida_id, sala_actual)

if enemics_derrotats > 0:
    baixes_acumulades += enemics_derrotats
    partida.reference.update({"baixes_acumulades": baixes_acumulades})
```

**Ejemplo:** Si un jugador llega a la sala 8 y ha derrotado 25 enemigos:
`puntuación = 25 × 10 + 8 × 50 = 250 + 400 = 650 puntos`

El resultado se guarda en la colección `puntuacions`:
```python
db.collection("puntuacions").add({
    "jugador_id": auth.uid,
    "puntuacio": puntuacio_total,
    "baixes": baixes_totals,
    "partida_id": partida_id,
    "data": SERVER_TIMESTAMP,
})
```

---

### Pregunta 14 — Profundización
**"¿Qué es `info` en `def crear_partida(self, info: strawberry.Info)` y qué información contiene?"**

**Respuesta:**
`info` es un objeto de tipo `strawberry.Info` que Strawberry inyecta automáticamente en cada resolver. Contiene **metadatos de la petición GraphQL en curso**.

En este proyecto, lo más importante es `info.context`, un diccionario que contiene:

```python
info.context = {
    "auth": AuthContext(uid="abc123", email="user@test.com", rol="jugador"),
    "request": <Request object de FastAPI>
}
```

También da acceso a:
- `info.field_name` — Nombre del campo que se está resolviendo
- `info.path` — Camino en la query (útil para tracing y debugging)
- `info.schema` — El schema completo de GraphQL (rara vez se usa directamente)
- `info.variable_values` — Variables pasadas en la query GraphQL

**Dato importante para el examen:** El parámetro `info` es especial porque **no se declara en el schema GraphQL**, el cliente no lo pasa explícitamente. Strawberry lo inyecta automáticamente porque detecta el tipo `strawberry.Info` en la firma de la función. Es la "puerta de entrada" al contexto de la petición.

---

### Pregunta 15 — Comparación
**"¿En qué se diferencia `registrar` de `registrarJugador`? ¿Cuándo usarías cada una?"**

**Respuesta:**

| Característica | `registrar` | `registrarJugador` |
|---|---|---|
| **Archivo** | `jugadors/mutations.py:71` | `jugadors/mutations.py:107` |
| **Parámetros** | `email, password, nickname` | `input: RegistrarJugadorInput` (solo `nickname`) |
| **¿Requiere auth previa?** | ❌ No (crea cuenta nueva) | ✅ Sí (necesita token JWT válido) |
| **¿Crea cuenta Firebase?** | ✅ Sí (llama a `sign_up`) | ❌ No (la cuenta ya existe) |
| **¿Crea perfil en Firestore?** | ✅ Sí | ✅ Sí |
| **Retorno en éxito** | `AuthPayload { token, jugador }` | `Jugador` (sin token) |
| **¿Cuándo usarla?** | Registro inicial: email + password + nickname | Usuario con cuenta Firebase pero sin perfil en Firestore |

**Flujo típico:**
1. Usuario nuevo → `registrar(email, password, nickname)` → obtiene token + perfil creado
2. Usuario con cuenta Firebase pero sin perfil → `login` → obtiene token → `registrarJugador(input: {nickname})` → obtiene perfil

---

﻿
## 9. GLOSARIO RÁPIDO

### Tipos GraphQL (Object Types)

| Tipo | Archivo | Línea | Campos principales |
|------|---------|-------|--------------------|
| `Jugador` | `app/jugadors/types.py` | 7 | `uid`, `nickname`, `nivell`, `banejat`, `diners` |
| `ItemInventari` | `app/jugadors/types.py` | 16 | `id`, `nomItem`, `raresa` |
| `AuthPayload` | `app/jugadors/types.py` | 23 | `token`, `jugador` |
| `Partida` | `app/partides/types.py` | 30 | `id`, `jugadorId`, `mapa`, `estat`, `dataCreacio`, `bossPosicio`, `botigaPosicio`, `salaActual`, `baixesAcumulades` |
| `Sala` | `app/partides/types.py` | 19 | `id`, `index`, `tipus`, `entitats`, `completada`, `doorLocation`, `items` |
| `ItemBotiga` | `app/partides/types.py` | 12 | `nom`, `preu`, `descripcio` |
| `Puntuacio` | `app/partides/types.py` | 43 | `id`, `jugadorId`, `puntuacio`, `baixes`, `data`, `jugador` (DataLoader + LazyType) |

### Input Types

| Tipo | Archivo | Línea | Campos |
|------|---------|-------|--------|
| `RegistrarJugadorInput` | `app/jugadors/types.py` | 54 | `nickname` |

### Tipos de Error (Union Types)

| Tipo de Error | Archivo | Línea |
|---|---|---|
| `ErrorCredencials` | `app/jugadors/types.py` | 29 |
| `ErrorAutenticacio` | `app/jugadors/mutations.py` | 31 |
| `ErrorJugadorBanejat` | `app/jugadors/types.py` | 49 |
| `ErrorJugadorNoTrobat` | `app/jugadors/types.py` | 44 |
| `ErrorJugadorJaExisteix` | `app/jugadors/mutations.py` | 36 |
| `ErrorItemNoDisponible` | `app/jugadors/types.py` | 34 |
| `ErrorDinersInsufficients` | `app/jugadors/types.py` | 39 |
| `ErrorSensePartidaActiva` | `app/partides/types.py` | 74 |
| `ErrorPartidaJaActiva` | `app/partides/types.py` | 79 |
| `ErrorMazmorraCompletada` | `app/partides/types.py` | 84 |

### Queries

| Query GraphQL | Función Python | Archivo | Línea | ¿Pública? |
|---------------|----------------|---------|-------|-----------|
| `perfilJugador(uid)` | `perfil_jugador` | `app/jugadors/queries.py` | 12 | ✅ Sí |
| `inventariJugador(uid)` | `inventari_jugador` | `app/jugadors/queries.py` | 26 | ✅ Sí |
| `partidaActiva` | `partida_activa` | `app/partides/queries.py` | 14 | ❌ Auth |
| `salesPartida` | `sales_partida` | `app/partides/queries.py` | 42 | ❌ Auth |
| `taulaClassificacio(limit, offset)` | `taula_classificacio` | `app/partides/queries.py` | 77 | ✅ Sí |

### Mutations

| Mutation GraphQL | Función Python | Archivo | Línea | Auth | Retorna (Union) |
|------------------|----------------|---------|-------|------|-----------------|
| `login(email, password)` | `login` | `jugadors/mutations.py` | 53 | ❌ | `AuthPayload \| ErrorCredencials` |
| `registrar(email, password, nickname)` | `registrar` | `jugadors/mutations.py` | 71 | ❌ | `AuthPayload \| ErrorCredencials` |
| `registrarJugador(input)` | `registrar_jugador` | `jugadors/mutations.py` | 107 | ✅ | `Jugador \| ErrorAutenticacio \| ErrorJugadorJaExisteix \| ErrorJugadorBanejat` |
| `afegirDiners(quantitat)` | `afegir_diners` | `jugadors/mutations.py` | 136 | ✅ | `Jugador \| ErrorAutenticacio` |
| `donarMonedes(jugadorId, quantitat)` | `donar_monedes` | `jugadors/mutations.py` | 157 | ✅ | `Jugador \| ErrorAutenticacio \| ErrorJugadorNoTrobat` |
| `atorgarItem(salaId, nomItem)` | `atorgar_item` | `jugadors/mutations.py` | 191 | ✅ | `ItemInventari \| ErrorAutenticacio \| ErrorItemNoDisponible \| ErrorDinersInsufficients` |
| `crearPartida` | `crear_partida` | `partides/mutations.py` | 134 | ✅ | `Partida \| ErrorSensePartidaActiva \| ErrorPartidaJaActiva \| ErrorJugadorBanejat` |
| `crearSala` | `crear_sala` | `partides/mutations.py` | 188 | ✅ | `Sala \| ErrorSensePartidaActiva \| ErrorMazmorraCompletada` |
| `finalitzarPartida` | `finalitzar_partida` | `partides/mutations.py` | 264 | ✅ | `Partida \| ErrorSensePartidaActiva \| ErrorJugadorBanejat` |

### Archivos Clave del Proyecto

| Archivo | Propósito |
|---------|-----------|
| `main.py` | Arranque FastAPI + GraphQLRouter con `context_getter` |
| `app/schema.py` | Ensamblaje del schema (herencia múltiple Mixin) |
| `app/auth.py` | `AuthContext` + `get_auth_context()` — extracción y verificación JWT |
| `app/auth_service.py` | `sign_up` / `sign_in` contra Firebase Identity REST API |
| `app/firebase_conf.py` | Inicialización Firebase Admin SDK (`db = firestore.client()`) |
| `app/loaders.py` | DataLoader para batch de jugadores (resuelve N+1) |
| `app/cataleg_botiga.py` | Catálogo base de la tienda (items con precio variable) |
| `app/jugadors/types.py` | Tipos: `Jugador`, `ItemInventari`, `AuthPayload`, `RegistrarJugadorInput`, errores |
| `app/jugadors/queries.py` | Queries: `perfil_jugador`, `inventari_jugador` |
| `app/jugadors/mutations.py` | Mutaciones de jugador + sistema anti-ofensivo |
| `app/partides/types.py` | Tipos: `Partida`, `Sala`, `ItemBotiga`, `Puntuacio` (con LazyType + DataLoader), errores |
| `app/partides/queries.py` | Queries: `partida_activa`, `sales_partida`, `taula_classificacio` |
| `app/partides/mutations.py` | Mutaciones de juego: `crear_partida`, `crear_sala`, `finalitzar_partida` + lógica de puntuación |

---

## 📌 RESUMEN FINAL PARA EL EXAMEN

### Las 12 cosas que SÍ o SÍ debes saber:

1. **GraphQL = un solo endpoint (`/graphql`)** donde el cliente pide exactamente lo que necesita. No hay over-fetching ni under-fetching.

2. **Strawberry es code-first**: decoras clases Python con `@strawberry.type`, `@strawberry.field`, `@strawberry.mutation`, `@strawberry.input` y él genera el schema automáticamente.

3. **snake_case → camelCase automático**: escribes `sala_actual` en Python, el cliente usa `salaActual` en GraphQL. NUNCA los mezcles.

4. **Schema por herencia múltiple (Mixins)**: `class Query(JugadorsQuery, PartidesQuery)` fusiona ambos dominios en un solo schema.

5. **Errores con Union Types, NO excepciones**: las mutaciones devuelven `Union[Éxito, Error1, Error2]` y el cliente discrimina con `... on TipoError { missatge }`.

6. **Auth por JWT de Firebase**: `get_auth_context()` extrae el token del header `Authorization: Bearer <token>`, lo verifica con `firebase_auth.verify_id_token()`, y lo inyecta en `info.context["auth"]` como `AuthContext(uid, email, rol)`.

7. **El UID del JWT, nunca del input**: `registrarJugador` usa `auth.uid`, nunca un UID pasado por el cliente. El `RegistrarJugadorInput` solo tiene `nickname`. Esto es por seguridad.

8. **Firestore directo, sin ORM**: `db.collection("x").document(id).get()`, `.set()`, `.update()`, `.stream()`, `FieldFilter` para WHERE. No hay SQL ni SQLAlchemy.

9. **DataLoader resuelve N+1**: `_batch_load_jugadors` usa `db.get_all()` para cargar N jugadores en 1 sola petición batch en lugar de N peticiones individuales. Se usa en `Puntuacio.jugador`.

10. **LazyType evita imports circulares**: `JugadorRef = LazyType("Jugador", "app.jugadors.types")` permite que `partides/types.py` referencie a `Jugador` sin crear una dependencia circular.

11. **Sistema anti-ofensivo**: lista negra `LLA_NEGRA` de nicknames → auto-ban con `banejat: True` al registrarse. Las mutaciones `crearPartida` y `finalitzarPartida` bloquean a jugadores baneados.

12. **Puntuación**: `baixes × 10 + sala_actual × 50`, calculada en `finalitzar_partida`, guardada en colección `puntuacions`, consultable vía `taulaClassificacio`.

---

> _Esta guía fue creada con cariño por tu profesor universitario, para que mi abuela española también pueda entender GraphQL. ¡Mucha suerte en el examen! 🍀_
