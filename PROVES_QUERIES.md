# AstroHunters — 4 Consultes de Prova + Bonus

> Document adaptat per a Postman. Totes les queries/mutations inclouen el format
> **GraphQL pur** (per al mode GraphQL) i el **curl equivalent** (per al mode JSON / terminal).

---

## Configuració de Postman

Postman permet enviar GraphQL de **dues maneres**. Tria la que et sigui més còmoda:

### Opció A (recomanada) — Mode GraphQL

1. Al desplegable del **Body**, selecciona **"GraphQL"** (no "raw").
2. Enganxa la query tal qual (el bloc de codi `graphql` que veuràs a cada test).
3. Les variables no calen — tot està escrit directament a la query.
4. Per als endpoints que requereixen JWT, ves a la pestanya **"Headers"** i afegeix:
   ```
   Authorization: Bearer <TOKEN>
   ```

### Opció B — Mode JSON / raw

1. Al desplegable del **Body**, selecciona **"raw"** i al desplegable de la dreta **"JSON"**.
2. Enganxa un objecte JSON amb la clau `"query"`:
   ```json
   {
     "query": "mutation { ... }"
   }
   ```
3. Les cometes dobles **dins** del valor de `"query"` s'han d'escapar amb `\"`.
   > Per estalviar-te feina, directament més avall trobaràs **curl commands** llestos per a
   > enganxar al terminal o a la pestanya de codi de Postman.

---

## Abans de començar

Segueix aquests **5 passos en ordre** per preparar l'entorn de proves.
Cada pas mostra un **curl command** que funciona tant al terminal com a la
pestanya "Code" de Postman (seleccionant "cURL" al desplegable).

> El servidor ha d'estar en marxa (`uv run python main.py` a `backend/`) i
> escoltant a `http://localhost:8000`.

---

### Pas 1 — Registrar-se i obtenir el JWT

Crea un compte d'usuari. El camp `token` del retorn és el teu JWT.

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"mutation { register(email: \"test@astrohunters.com\", password: \"123456\") { ... on AuthPayload { token uid } ... on AuthError { message } } }"}'
```

**Què fer després:** Copia el valor de `token` (sense cometes) i guarda'l.
El necessitaràs als passos 2, 3 i 5.

> **Nota sobre l'email `@astrohunters.com`:** L'server et considera
> **admin** si l'acabament del correu és `@astrohunters.com`. Si vols provar
> el Bonus (`pujarNivell`), has de registrar-te amb un email així.

---

### Pas 2 — Configurar l'Authorization Header

Ara que ja tens el token, **cada curl** dels passos següents porta el header
`Authorization: Bearer <TOKEN>`. Substitueix `<TOKEN>` pel valor que has
obtingut al Pas 1.

> A Postman (mode GraphQL), afegeix-lo a la pestanya Headers com:
> ```
> Authorization: Bearer el_teu_token_aqui
> ```

---

### Pas 3 — Crear el perfil del jugador

Amb el JWT al header, crea el perfil públic. El `nickname` ha de ser únic.

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query":"mutation { registrarJugador(nickname: \"AstroHero\") { uid nickname nivell banejat } }"}'
```

**Guarda el `uid`** que retorna — el faràs servir als tests 1 i 4, i al Pas 5.

---

### Pas 4 — Crear una partida

Crea una partida per poder-hi afegir puntuacions.

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"mutation { crearPartida(mapa: \"Base Lunar\") { id mapa estat dataCreacio } }"}'
```

**Guarda el `id`** que retorna — el necessitaràs als tests 2, 3, 4 i al Pas 5.

---

### Pas 5 — Afegir puntuacions de prova

Substitueix `PARTIDA_ID` i `JUGADOR_UID` amb els valors dels passos anteriors.
Pots executar-ho diverses vegades amb diferents jugadors i punts.

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query":"mutation { registrarPuntuacio(partidaId: \"PARTIDA_ID\", jugadorId: \"JUGADOR_UID\", punts: 1500, baixes: 12) { id punts baixes } }"}'
```

> **Consell:** Canvia `punts`, `baixes` i `jugadorId` per crear múltiples
> puntuacions i veure millor el funcionament de la classificació al Test 3.

---

Un cop completats els 5 passos, ja pots començar amb les consultes de prova.

---

## Test 1 — Query `perfilJugador` amb `inventari` niuat

**Objectiu:** Comprovar que l'esquema resol correctament un tipus amb una
subcol·lecció (`inventari` dins de `Jugador`).

### Mode GraphQL (Postman → Body: GraphQL)

```graphql
query {
  perfilJugador(id: "JUGADOR_UID") {
    uid
    nickname
    nivell
    banejat
    inventari {
      id
      nomItem
      raresa
    }
  }
}
```

### Mode JSON / terminal (Postman → Body: raw / JSON)

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"query { perfilJugador(id: \"JUGADOR_UID\") { uid nickname nivell banejat inventari { id nomItem raresa } } }"}'
```

Substitueix `JUGADOR_UID` pel valor que vas guardar al Pas 3.

**Esperat:** Retorna el perfil del jugador amb el seu inventari. Si no té items,
`inventari` serà un array buit `[]`.

---

## Test 2 — Query `llistarPartides` amb filtre i paginació

**Objectiu:** Comprovar que els filtres per `estat` i la paginació (`limit`,
`offset`) funcionen correctament.

### Mode GraphQL (Postman → Body: GraphQL)

```graphql
query {
  llistarPartides(estat: "En curs", limit: 5, offset: 0) {
    id
    mapa
    estat
    dataCreacio
    puntuacions {
      id
      punts
    }
  }
}
```

### Mode JSON / terminal (Postman → Body: raw / JSON)

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"query { llistarPartides(estat: \"En curs\", limit: 5, offset: 0) { id mapa estat dataCreacio puntuacions { id punts } } }"}'
```

Pots canviar `estat` per altres valors (p. ex. `"Finalitzada"`) o jugar amb
`limit` i `offset` per veure com es comporta la paginació.

**Esperat:** Retorna només les partides amb l'estat indicat, ordenades per data
de creació, amb un màxim de `limit` resultats i saltant-se els primers `offset`
resultats.

---

## Test 3 — Query `taulaClassificacio` amb `jugador { nickname nivell }` (LazyType + DataLoader)

**Objectiu:** Demostrar que `Puntuacio` pot resoldre el `nickname` i `nivell`
del jugador (relació N:1 inversa) usant `strawberry.LazyType` i `DataLoader`
per evitar el problema N+1.

### Mode GraphQL (Postman → Body: GraphQL)

```graphql
query {
  taulaClassificacio(idPartida: "PARTIDA_ID") {
    id
    punts
    baixes
    jugador {
      nickname
      nivell
    }
  }
}
```

### Mode JSON / terminal (Postman → Body: raw / JSON)

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"query { taulaClassificacio(idPartida: \"PARTIDA_ID\") { id punts baixes jugador { nickname nivell } } }"}'
```

Substitueix `PARTIDA_ID` pel valor que vas guardar al Pas 4.

**Esperat:** Retorna les puntuacions ordenades per punts (desc). Cada puntuació
inclou el `nickname` i `nivell` del jugador corresponent, carregat amb un sol
batch request a Firestore gràcies al `DataLoader`.

---

## Test 4 — Mutació `finalitzarPartida` amb union error type (`ErrorPartidaNoTrobada`)

**Objectiu:** Comprovar el funcionament de les **Unions d'error** i les regles
de negoci (només es pot finalitzar una partida si existeix).

> ⚠️ Les unions en GraphQL **requereixen** la sintaxi `... on TypeName` per
> accedir als camps. No hi ha alternativa. Funciona perfectament a Postman
> tant en mode GraphQL com en mode JSON.

### Mode GraphQL (Postman → Body: GraphQL)

```graphql
mutation {
  finalitzarPartida(partidaId: "PARTIDA_ID") {
    ... on Partida {
      id
      mapa
      estat
      dataCreacio
    }
    ... on ErrorPartidaNoTrobada {
      message
    }
  }
}
```

### Mode JSON / terminal (Postman → Body: raw / JSON)

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"mutation { finalitzarPartida(partidaId: \"PARTIDA_ID\") { ... on Partida { id mapa estat dataCreacio } ... on ErrorPartidaNoTrobada { message } } }"}'
```

Substitueix `PARTIDA_ID` pel valor que vas guardar al Pas 4.

**Esperat (cas existència):** Retorna l'objecte `Partida` amb `estat: "Finalitzada"`.
**Esperat (cas error):** Si la partida no existeix (prova-ho amb un ID inventat),
retorna `ErrorPartidaNoTrobada` amb el missatge d'error corresponent.

---

## Bonus — Mutació `pujarNivell` (només admin)

**Objectiu:** Comprovar les Unions d'error amb múltiples tipus
(`ErrorNoAutoritzat`, `ErrorJugadorBanejat`) i la regla de negoci que només
els administradors poden pujar de nivell.

> **Requisit:** Necessites un JWT d'un usuari amb email acabat en
> `@astrohunters.com`. Si al Pas 1 et vas registrar amb
> `test@astrohunters.com`, ja tens permís d'admin.

### Mode GraphQL (Postman → Body: GraphQL)

```graphql
mutation {
  pujarNivell(jugadorId: "JUGADOR_UID") {
    ... on Jugador {
      uid
      nickname
      nivell
    }
    ... on ErrorNoAutoritzat {
      message
    }
    ... on ErrorJugadorBanejat {
      message
    }
  }
}
```

### Mode JSON / terminal (Postman → Body: raw / JSON)

```bash
curl -s http://localhost:8000/graphql -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query":"mutation { pujarNivell(jugadorId: \"JUGADOR_UID\") { ... on Jugador { uid nickname nivell } ... on ErrorNoAutoritzat { message } ... on ErrorJugadorBanejat { message } } }"}'
```

Substitueix `JUGADOR_UID` pel UID del jugador que vols pujar de nivell, i
`TOKEN` pel JWT del Pas 1.

**Esperat:**
- Si ets admin → retorna el `Jugador` amb el `nivell` incrementat en 1.
- Si **no** ets admin → retorna `ErrorNoAutoritzat`.
- Si el jugador està banejat → retorna `ErrorJugadorBanejat`.

---

## Resum ràpid de tipus i unions

| Mutació / Query | Retorna | Inline fragments necessaris? |
|---|---|---|
| `register` / `login` | `AuthResult` (unió: `AuthPayload` \| `AuthError`) | ✅ `... on AuthPayload`, `... on AuthError` |
| `registrarJugador` | `Jugador` (directe) | ❌ No |
| `crearPartida` | `Partida` (directe) | ❌ No |
| `registrarPuntuacio` | `Puntuacio` (directe) | ❌ No |
| `finalitzarPartida` | `FinalitzarPartidaResult` (unió: `Partida` \| `ErrorPartidaNoTrobada`) | ✅ `... on Partida`, `... on ErrorPartidaNoTrobada` |
| `pujarNivell` | `PujarNivellResult` (unió: `Jugador` \| `ErrorNoAutoritzat` \| `ErrorJugadorBanejat`) | ✅ `... on Jugador`, `... on ErrorNoAutoritzat`, `... on ErrorJugadorBanejat` |
| `perfilJugador` | `Jugador \| null` (directe) | ❌ No |
| `llistarPartides` | `[Partida]` (directe) | ❌ No |
| `taulaClassificacio` | `[Puntuacio]` (directe) | ❌ No |

> **Nota:** Les unions (`... on TypeName`) són **l'única manera** d'accedir als
> camps d'un tipus dins d'una unió en GraphQL. Postman les suporta perfectament
> tant en mode "GraphQL" com en mode "raw / JSON".
