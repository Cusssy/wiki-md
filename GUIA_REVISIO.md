# 🚀 AstroHunters Backend — Guia de Revisió

## Estructura del Projecte (DDD)

```
backend/
├── main.py                      # FastAPI + Strawberry GraphQL router
├── credentials.json             # Firebase Admin SDK (NO COMMITEJAR)
├── pyproject.toml               # Dependències (FastAPI, Strawberry, Firebase)
├── app/
│   ├── schema.py                # Ensamblador central (herència múltiple Mixins)
│   ├── firebase_conf.py         # Inicialització Firebase Admin
│   ├── auth.py                  # Extracció JWT → AuthContext (uid, email, rol)
│   ├── auth_service.py          # Sign up / sign in contra Firebase Identity
│   ├── cataleg_botiga.py        # Catàleg de la botiga (items base)
│   ├── loaders.py               # DataLoader per batch de jugadors (N+1)
│   ├── jugadors/                # Domini: Perfils i inventari
│   │   ├── types.py             # Tipus: Jugador, ItemInventari, errors...
│   │   ├── queries.py           # Queries: perfil_jugador, inventari_jugador
│   │   └── mutations.py         # Mutacions: registrar, login, donar_monedes...
│   └── partides/                # Domini: Partides, sales, puntuacions
│       ├── types.py             # Tipus: Partida, Sala, Puntuacio, errors...
│       ├── queries.py           # Queries: partida_activa, taula_classificacio
│       └── mutations.py         # Mutacions: crear_partida, finalitzar_partida...
```

---

## Model de Dades Firestore

### Col·lecció: `jugadors` (doc ID = uid del Firebase Auth)
| Camp | Tipus | Descripció |
|---|---|---|
| nickname | String | Nom del jugador |
| nivell | Int | Nivell (comença a 1) |
| banejat | Bool | Jugador banejat? |
| diners | Int | Monedes del jugador |

#### Subcol·lecció: `jugadors/{uid}/inventari`
| Camp | Tipus | Descripció |
|---|---|---|
| nom_item | String | Ex: "Pistola Làser" |
| raresa | String | "Comú", "Èpic", "Llegendari" |

### Col·lecció: `partides` (auto-ID)
| Camp | Tipus | Descripció |
|---|---|---|
| jugador_id | String | UID del jugador |
| mapa | String | Ex: "Mazmorra Profunda" |
| estat | String | "En curs" / "Finalitzada" |
| data_creacio | Timestamp | |
| sala_actual | Int | Sala on es troba |
| boss_posicio | Int | Posició del boss |
| botiga_posicio | Int | Posició de la botiga |
| baixes_acumulades | Int | Enemics derrotats totals |
| cofre_generat | Bool | Si ja s'ha generat cofre |

#### Subcol·lecció: `partides/{id}/sales`
| Camp | Tipus | Descripció |
|---|---|---|
| index | Int | Número seqüencial |
| tipus | Int | 1-4 (normal), 6 (botiga), 10 (boss) |
| entitats | [Int] | Enemics presents a la sala |
| completada | Bool | |
| door_location | String | "up", "down", "left", "right" |
| items | [dict] | Items a la botiga (opcional) |

### Col·lecció: `puntuacions` (auto-ID)
| Camp | Tipus | Descripció |
|---|---|---|
| jugador_id | String | UID del jugador |
| puntuacio | Int | `baixes * 10 + sala_actual * 50` |
| baixes | Int | Enemics derrotats |
| partida_id | String | Referencia a la partida |
| data | Timestamp | |

---

## Esquema GraphQL

### Queries

```graphql
# Perfil complet del jugador
query {
  perfilJugador(uid: "abc123") {
    nickname
    nivell
    banejat
    diners
  }
}

# Inventari del jugador
query {
  inventariJugador(uid: "abc123") {
    nomItem
    raresa
  }
}

# Partida activa del jugador autenticat
query {
  partidaActiva {
    id
    estado
    salaActual
    baixesAcumulades
  }
}

# Sales de la partida activa
query {
  salesPartida {
    index
    tipus
    entitats
    completada
  }
}

# Ranking global amb paginacio (DataLoader per evitar N+1)
query {
  taulaClassificacio(limit: 10, offset: 0) {
    jugadorId
    puntuacio
    baixes
    jugador {       # LazyType! Relacio N:1 a Jugador
      nickname
    }
  }
}
```

### Mutacions

```graphql
# Registrar (crea compte Firebase + perfil)
mutation {
  registrar(email: "user@test.com", password: "123456", nickname: "Player1") {
    ... on AuthPayload { token jugador { nickname } }
    ... on ErrorCredencials { missatge }
  }
}

# Login (retorna token JWT)
mutation {
  login(email: "user@test.com", password: "123456") {
    ... on AuthPayload { token jugador { uid nickname diners } }
  }
}

# Registrar jugador (extreu UID del JWT, no es passa per parametre)
mutation {
  registrarJugador(input: { nickname: "Hero" }) {
    ... on Jugador { uid nickname nivell }
    ... on ErrorJugadorBanejat { missatge }
  }
}

# Donar monedes a un altre jugador
mutation {
  donarMonedes(jugadorId: "abc123", quantitat: 100) {
    ... on Jugador { diners }
    ... on ErrorJugadorNoTrobat { missatge }
  }
}

# Crear nova partida
mutation {
  crearPartida {
    ... on Partida { id mapa salaActual baixesAcumulades }
    ... on ErrorPartidaJaActiva { missatge }
    ... on ErrorJugadorBanejat { missatge }
  }
}

# Avançar a la següent sala
mutation {
  crearSala {
    ... on Sala { index tipus entitats }
    ... on ErrorMazmorraCompletada { missatge }
  }
}

# Finalitzar partida (calcula puntuacio i la guarda)
mutation {
  finalitzarPartida {
    ... on Partida { id estat salaActual baixesAcumulades }
    ... on ErrorJugadorBanejat { missatge }
  }
}

# Comprar item a la botiga
mutation {
  atorgarItem(salaId: "abc123", nomItem: "Poció de Vida") {
    ... on ItemInventari { nomItem raresa }
    ... on ErrorDinersInsufficients { missatge }
  }
}
```

---

## Punts Clau Per La Revisió

### 1. Arquitectura DDD (Domain-Driven Design)

- Dos dominis: `app/jugadors/` i `app/partides/`
- Cada domini amb `types.py`, `queries.py`, `mutations.py`
- `app/schema.py` fusiona tot amb **herència múltiple (Mixins)**:
  ```python
  class Query(JugadorsQuery, PartidesQuery): pass
  class Mutation(JugadorsMutation, PartidesMutation): pass
  ```

### 2. LazyType (dependències entre dominis)

Fitxer: `app/partides/types.py:8`

```python
from strawberry.types.lazy_type import LazyType

JugadorRef = LazyType(type_name="Jugador", module="app.jugadors.types")

@strawberry.type
class Puntuacio:
    @strawberry.field
    async def jugador(self) -> JugadorRef | None:
        ...
```

`Puntuacio.jugador` referencia `Jugador` del domini `jugadors` sense crear un import circular. Strawberry resol el tipus en temps de construcció de l'esquema.

### 3. DataLoader (evitar el problema N+1)

Fitxer: `app/loaders.py`

Quan el client demana 50 puntuacions amb el `nickname` de cada jugador:
- ❌ **Sense DataLoader**: 50 peticions individuals a Firestore
- ✅ **Amb DataLoader**: 1 sola petició batch (`db.get_all()`)

```python
def _batch_load_jugadors(ids: list[str]) -> list[dict | None]:
    referencies = [db.collection("jugadors").document(uid) for uid in ids]
    documents = db.get_all(referencies)  # UNA sola crida
    ...
```

### 4. Sistema Anti-Trampes (Trigger d'ofensivitat + Union Types)

Fitxer: `app/jugadors/mutations.py`

- **Llista negra** de nicknames ofensius
- Si un jugador es registra amb un nom prohibit → `banejat: true`
- `registrar_jugador` retorna `ErrorJugadorBanejat` (Union Type)
- `crear_partida` i `finalitzar_partida` comproven `banejat` i el retornen

```graphql
mutation {
  registrarJugador(input: { nickname: "Israel" }) {
    ... on ErrorJugadorBanejat { missatge }
  }
}
```

El client rep un error **tipat**, no una excepció genèrica.

### 5. Puntuació i Leaderboard

Fitxer: `app/partides/mutations.py` (finalitzar_partida)

- **Fórmula**: `puntuacio = baixes*10 + sala_actual*50`
- Cada cop que avances de sala, els enemics de la sala anterior s'acumulen a `baixes_acumulades`
- En finalitzar la partida, es guarda un document a la col·lecció `puntuacions`
- `taula_classificacio` retorna el ranking global amb paginació

### 6. Seguretat JWT

Fitxer: `app/auth.py`

- Token JWT de Firebase verificat a cada petició
- `registrar_jugador` extreu l'UID del token, no del body
- `AuthContext` inclou `uid`, `email` i `rol` (admin si email acaba en `@astrohunters.com`)

---

## Glossari Ràpid

| Què busques? | Fitxer |
|---|---|
| `Jugador` type | `app/jugadors/types.py:6` |
| `Partida` type | `app/partides/types.py:29` |
| `Puntuacio` type (amb LazyType) | `app/partides/types.py:42` |
| `taula_classificacio` query | `app/partides/queries.py:78` |
| `finalitzar_partida` (guarda puntuacio) | `app/partides/mutations.py:261` |
| `crear_sala` (acumula baixes) | `app/partides/mutations.py:185` |
| `donar_monedes` | `app/jugadors/mutations.py:142` |
| `registrar_jugador` (anti-ofensiu) | `app/jugadors/mutations.py:101` |
| Llista negra nicknames | `app/jugadors/mutations.py:22` |
| DataLoader batch jugadors | `app/loaders.py:7` |
| Autenticació JWT | `app/auth.py:11` |
| Schema (fusió Mixins) | `app/schema.py:10` |
| `ErrorJugadorBanejat` | `app/jugadors/types.py:48` |
| `ErrorJugadorNoTrobat` | `app/jugadors/types.py:44` |
| FastAPI main | `main.py:1` |
