# CLAUDE.md — Padel App: Pozos & Retos

Documento de contexto para continuar el desarrollo de esta aplicación con Claude.

---

## Qué es la app

Aplicación web para registrar torneos de pádel ("pozos") y partidos de reto. Funciona como una SPA (Single Page Application) en un solo archivo HTML. Los datos se sincronizan entre dispositivos usando GitHub Gist como base de datos, con un Cloudflare Worker como proxy que mantiene el token de GitHub oculto.

---

## Arquitectura general

```
Browser (pozos.html)
    ↓ fetch
Cloudflare Worker  ←  GITHUB_TOKEN (secret de Cloudflare)
    ↓ PATCH/GET
GitHub Gist API  →  pozos.json (datos del usuario)
```

- **Frontend:** un solo archivo `pozos.html`. Sin dependencias npm, sin bundler. HTML + CSS + JS vanilla inline.
- **Backend:** `worker.js` desplegado en Cloudflare Workers. Solo hace de proxy — recibe peticiones de la app, agrega el token de GitHub, y reenvía a la Gist API.
- **Base de datos:** un Gist de GitHub por usuario, con un archivo `pozos.json`.
- **Deploy del frontend:** Cloudflare Pages o GitHub Pages (drag and drop del `pozos.html` renombrado a `index.html`).

---

## Archivos del proyecto

| Archivo | Descripción |
|---|---|
| `pozos.html` | La app completa. Todo el HTML, CSS y JS en un solo archivo. |
| `worker.js` | El Cloudflare Worker proxy. Se sube al dashboard de Cloudflare. |
| `CLAUDE.md` | Este archivo. |

---

## URLs de producción

- **App:** la URL que el usuario tenga en Cloudflare Pages o GitHub Pages
- **Worker:** `https://padel.fedexlang.workers.dev`
  - `GET /gist/:gistId` → lee datos del usuario
  - `POST /gist/:gistId` → guarda datos del usuario

---

## Formato del JSON en el Gist

```json
{
  "nombre": "Juan",
  "pozos": [ ...array de pozos... ],
  "retos": [ ...array de retos... ]
}
```

### Estructura de un pozo

```json
{
  "id": "abc123",
  "fecha": "2026-04-15",
  "categoria": "5ta",
  "club": "Costa Rica Country Club",
  "tipo": "pareja",
  "companero": "Eduardo",
  "participantes": 16,
  "posicion": 3,
  "victorias": 4,
  "empates": 1,
  "derrotas": 2,
  "puntaje": 12,
  "calorias": 450,
  "costo": 25.00,
  "premio": 80.00,
  "notas": "Buena jornada",
  "creado": "2026-04-15T10:00:00.000Z"
}
```

### Estructura de un reto

```json
{
  "id": "def456",
  "tipo": "reto",
  "fecha": "2026-04-20",
  "club": "Costa Rica Country Club",
  "companero": "Eduardo",
  "rival1": "Carlos",
  "rival2": "Pedro",
  "sets": [
    { "mine": 6, "rival": 4 },
    { "mine": 3, "rival": 6 },
    { "mine": 7, "rival": 5 }
  ],
  "calorias": 380,
  "precio": 15.00,
  "notas": null,
  "creado": "2026-04-20T15:00:00.000Z"
}
```

El resultado de un reto (V/E/D) **se calcula en tiempo real** contando cuántos sets ganó cada equipo — no se guarda en el JSON.

---

## Estado global en JS (variables principales)

```js
let pozos = [];         // array de pozos
let retos = [];         // array de retos
let nombre = '';        // nombre del usuario (guardado en el Gist)
let editingId = null;   // id del pozo que se está editando (null = nuevo)
let editingRetoId = null;
let activeTab = 'pozos'; // 'pozos' | 'retos'
let settings = null;    // { gistId } — guardado en localStorage
let syncing = false;
let lastSyncedAt = null;
let lastError = null;
let searchQuery = '';
let sortBy = { field: 'fecha', dir: 'desc' };
let chartMode = 'absolute'; // 'absolute' | 'percentual'
```

---

## Funciones clave del JS

### Sincronización con el Worker

```js
workerFetch(gistId)              // GET → devuelve { pozos, retos, nombre }
workerUpdate(gistId, pArr, rArr, nom) // POST → guarda todo
syncDown()                       // baja datos del Gist y rerenderiza
syncUp()                         // sube estado actual al Gist
```

Cualquier cambio en los datos (agregar/editar/borrar pozo o reto) debe llamar `syncUp()` al final.

### Datos

```js
migrate(p)          // migra un pozo de formato viejo a nuevo
migrateReto(r)      // normaliza campos de un reto
calcRetoResultado(sets)  // calcula 'V'|'E'|'D'|null desde los sets
pozoWinRate(p)      // win rate de un pozo como % (0-100 | null)
pozoPercentualPos(p) // posición relativa: 100% = 1er lugar, 0% = último
```

### Búsqueda

```js
getSearchTerms()    // parsea searchQuery en grupos OR/AND
matchesSearch(p)    // true si el pozo matchea la query actual
getFilteredPozos()  // devuelve pozos filtrados por searchQuery
```

Sintaxis de búsqueda:
- `+` = OR: `enero + febrero` → pozos de enero o febrero
- `&` = AND: `country club & eduardo` → pozos de Country Club con Eduardo
- Se pueden combinar: `country club & eduardo + la cima & pedro`

### Render

```js
render()            // llama a renderStats + renderChart + renderList + renderRetosStats + renderRetos
renderStats()       // usa getFilteredPozos() — reacciona al searchQuery
renderChart()       // gráfico SVG de posiciones a lo largo del tiempo
renderList()        // historial de pozos filtrado y ordenado
renderRetosStats()  // dashboard de stats de retos
renderRetos()       // lista de retos ordenada por fecha
```

---

## Almacenamiento local (localStorage)

La app usa dos keys en localStorage como caché offline:

| Key | Contenido |
|---|---|
| `pozos_settings_v1` | `{ gistId }` — configuración del dispositivo |
| `pozos_cache_v1` | `{ pozos: [...], retos: [...] }` — última snapshot conocida |

Al cargar la app: muestra el caché inmediatamente y luego sincroniza desde el Gist en segundo plano.

---

## Diseño visual

### Paleta de colores (CSS variables)

```css
--bg: #0a100c           /* fondo general */
--surface: #131b16      /* cards y panels */
--surface-2: #1c2620    /* inputs y elementos secundarios */
--surface-3: #243029    /* hover states */
--court: #2d8659        /* verde oscuro (tab activo) */
--court-bright: #4ade80 /* verde brillante (acento principal) */
--ink: #f0f4ed          /* texto principal */
--ink-muted: #8a9690    /* texto secundario */
--ink-dim: #5a655e      /* texto muy apagado */
--gold: #fbbf24         /* premios, pareja en chart */
--crimson: #ef4444      /* derrota, valores negativos */
--amber: #f59e0b        /* empate, warnings */
```

### Tipografía

- **Bebas Neue** — headings, números grandes, botones FAB
- **Manrope** — cuerpo de texto, labels, botones normales
- **JetBrains Mono** — chips, datos técnicos, fechas, IDs

### Colores en el gráfico de posiciones

- Punto dorado `#fbbf24` → torneo en pareja
- Punto morado `#a78bfa` → torneo individual

---

## Flujo de setup para un usuario nuevo

1. El administrador crea un Gist en `gist.github.com` con un archivo `pozos.json` con contenido `[]`
2. Le pasa el Gist ID al usuario (ej: `4b3a2c1d...`)
3. El usuario abre la app → pantalla de setup → pega el Gist ID → "Conectar"
4. El Worker valida que el Gist existe y devuelve los datos
5. La configuración queda guardada en localStorage del dispositivo

En ajustes (ícono ⚙️) el usuario puede:
- Cambiar su nombre (se guarda en el Gist)
- Cambiar el Gist ID
- Exportar JSON como backup
- Desconectarse (borra localStorage, datos del Gist intactos)

---

## Cloudflare Worker — detalles de configuración

El `worker.js` usa ES Modules (`export default { async fetch() {} }`).

Variables de entorno requeridas en Cloudflare:

| Variable | Tipo | Valor |
|---|---|---|
| `GITHUB_TOKEN` | **Secret** | Personal Access Token de GitHub con scope `gist` |
| `ALLOWED_ORIGIN` | Variable normal | URL de la app (ej: `https://pozos.pages.dev`). Opcional — si no se define acepta cualquier origen |

Endpoints:

```
GET  /gist/:gistId   → lee pozos.json del Gist, devuelve { data: { pozos, retos, nombre } }
POST /gist/:gistId   → recibe { data: { pozos, retos, nombre } }, actualiza el Gist
```

El Worker acepta tanto el formato nuevo `{ pozos, retos, nombre }` como el formato viejo `[...]` (array puro) por compatibilidad con datos anteriores.

---

## Cómo agregar nuevos campos a un pozo o reto

1. **Formulario HTML** — agregar el `<input>` dentro del modal correspondiente (`#modalBg` para pozos, `#retoModalBg` para retos)
2. **`migrate()` o `migrateReto()`** — agregar el campo con su valor default para datos existentes que no lo tengan
3. **Submit del form** — incluir el campo en el objeto `data` que se construye
4. **`openModal()` o `openRetoModal()`** — popularlo al abrir en modo edición
5. **`renderList()` o `renderRetos()`** — mostrarlo en la card (chip o detalle expandido)
6. **`pozoSearchText()`** — incluirlo si debe ser buscable

---

## Cómo funciona la búsqueda

La función `pozoSearchText(p)` concatena todos los campos textuales de un pozo en un string en minúsculas. Luego `matchesSearch(p)` evalúa la query contra ese string.

La query se parsea así:
1. Split por `+` → grupos OR
2. Cada grupo se splitea por `&` → términos AND
3. Un pozo matchea si ALGÚN grupo OR matchea
4. Un grupo matchea si TODOS sus términos AND aparecen en el texto

Para agregar un campo al buscador, agregarlo a `pozoSearchText`:

```js
function pozoSearchText(p) {
  const parts = [
    p.categoria || '', p.club || '', p.companero || '',
    p.notas || '', p.tipo || '', p.fecha || '',
    String(p.posicion || ''), String(p.puntaje ?? ''),
    p.nuevocampo || '',  // ← agregar acá
  ];
  // ...
}
```

---

## Cómo funciona el gráfico

SVG generado inline en `renderChart()`. No usa librerías externas.

- Eje X: fechas de los pozos ordenadas cronológicamente
- Eje Y: posición (modo absoluto: 1 arriba, último abajo) o posición relativa (modo porcentual: 100% = 1er lugar)
- Puntos coloreados por tipo: dorado = pareja, morado = individual
- Toggle "Absoluta / % Relativa" persistido en variable `chartMode`
- Respeta el `searchQuery` activo — solo muestra los pozos que matchean

Fórmula de posición relativa:
```
(participantes - posicion) / (participantes - 1) * 100
```
Ejemplo: 3/12 = 81.8%, 3/48 = 95.7%

---

## Notas de implementación importantes

- La app usa `'use strict'` y ES2020+. No hay transpilación.
- El `$` es un alias de `document.querySelector`, no jQuery.
- El `uid()` genera IDs únicos combinando timestamp base36 + random. No usa crypto.
- Los modales se manejan con la clase CSS `.open` en `.modal-bg`.
- El click handler de reto cards usa `capture: true` (`addEventListener('click', fn, true)`) para evitar conflictos con el handler de pozos cards.
- `saveCache(pozos, retos)` siempre recibe ambos arrays. Si se llama con uno solo rompe la caché.
- Los datos del Gist se migran automáticamente en `workerFetch` — si el JSON es un array puro (formato viejo), se convierte a `{ pozos: data, retos: [], nombre: '' }`.

---

## Ideas para próximas funciones (backlog)

- **Rendimiento por compañero** — win rate y posición promedio jugando con cada persona
- **Rendimiento por club** — en qué cancha se juega mejor
- **Racha actual** — cuántos pozos seguidos con posición top 3
- **Share card** — imagen para compartir por WhatsApp con el resultado del pozo
- **PWA** — instalable en el home screen del celular
- **Múltiples usuarios** — actualmente la arquitectura asume 1 usuario por Gist. Para múltiples usuarios habría que agregar un mini backend propio (ya tienen el Worker como punto de partida) o usar Cloudflare KV.
- **Partidos individuales dentro del reto** — desglose de cada set con marcador por juego
