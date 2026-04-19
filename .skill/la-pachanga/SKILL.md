---
name: la-pachanga
description: >
  Skill para construir, extender o modificar "La Pachanga", un tracker de fútsal semanal entre amigos.
  Cubre arquitectura completa, modelo de datos, lógica de rotación, cronómetro, IA, apodos, arqueros,
  panel admin, perfiles de jugadores, colores de equipo personalizables, goles en tiempo real con minuto,
  tabla de posiciones con G/E/P, vistas jornada/editjornada y sistema de diseño verde/dorado (v2).
  Usar cuando el usuario pida nuevas funciones, correcciones o una nueva versión del HTML.
---

# La Pachanga — Skill de desarrollo v9

## Archivos activos

| Archivo | Rol |
|---|---|
| `index_local.html` | Versión anterior con lógica avanzada (source of truth lógico) |
| `index_local_v2.html` | **Versión activa** — diseño v2 (Anton/Barlow Condensed, bottom nav, paleta oscura) + toda la lógica de v1 portada |

**Nunca tocar producción/Vercel** — todos los cambios son sobre los archivos `_local`.

---

## Contexto del producto

App web mobile-first (HTML monofichero, sin build, sin servidor) para gestionar una pachanga de fútsal entre amigos. Se comparte por WhatsApp y se agrega a la pantalla de inicio del celu como PWA.

### Formato de juego
- **N equipos de M jugadores** — configurable desde el panel admin (default: 3 equipos × 5 jugadores).
- **Partidos**: duración configurable (default 4 min) o N goles (default 2), lo que ocurra primero.
- **La última**: al llegar a 0 el cronómetro sigue negativo. **Cualquier gol durante tiempo negativo termina el partido automáticamente.**
- **Puntos**: 3 victoria / 1 empate / 0 derrota.
- **Rotación**: gana → se queda (`team1`), pierde → descansa, resting → entra. Empate → sale `team1` (incumbente).
- **Campeón vigente**: el equipo campeón de la última jornada se pre-carga en Equipo 1 del siguiente setup.

---

## Stack técnico

| Elemento | Decisión |
|---|---|
| Framework | **Vanilla JS puro** — sin React, sin build, sin CDN externo |
| Estilo | CSS variables + clases utilitarias inline en `<style>` |
| Persistencia | `localStorage` (offline-first, monousuario) |
| IA | Anthropic API `/v1/messages` directo desde el browser |
| Distribución | Un único `.html` portable, funciona en Safari local sin errores |

**Por qué vanilla JS**: React 18 via CDN falla en Safari cuando el archivo se abre localmente (`Script error`). Se migró a vanilla JS en v6 y se mantuvo.

**Pattern de render**: todas las vistas son funciones que devuelven HTML string → se inyectan en `document.getElementById('app').innerHTML`. Los event listeners se reasignan en `attach()` después de cada render.

---

## localStorage keys (v9 actual)

```js
const SK_H   = 'ph9';   // historial de jornadas (array)
const SK_C   = 'pc9';   // jornada en curso (objeto | null)
const SK_P   = 'pp3';   // lista de jugadores (array de strings)
const SK_K   = 'pak';   // API key de Anthropic (string)
const SK_N   = 'pnk9';  // apodos { [nombreCompleto]: apodo }
const SK_CFG = 'pcfg9'; // configuración admin (objeto)
```

**Regla**: al hacer cambios breaking al esquema, incrementar el sufijo numérico para evitar datos corruptos en browsers de usuarios existentes.

---

## Modelo de datos completo

### Jornada (`S.cur` / item de `S.hist`)
```js
{
  id: string,
  date: string,           // "14/03/2026" (formateado, no ISO)
  theme: string,          // "Gasistas vs Mecánicos vs Plomeros"
  teams: Team[],          // nTeams equipos
  matches: Match[],       // partidos jugados en orden
  goals: { [playerName]: number },  // goles acumulados en la jornada
  events: GoalEvent[],    // log de eventos con minuto (nuevo en v9)
  standings: { [teamId]: Standing },
  status: 'active' | 'done',
  champion: teamId | null,
  mvp: playerName | null,
  nextMatch: NextMatch,
  gk: { [teamId]: playerName }  // arquero activo por equipo (se resetea entre partidos)
}
```

### Team
```js
{ id: string, name: string, players: string[], color?: string }
// color: hex opcional (e.g. '#fb923c') — sobreescribe el color por defecto CHX[i]
```

### GoalEvent (nuevo v9)
```js
{
  type: 'goal',
  player: playerName,
  team: teamId,
  matchIdx: number,     // índice en cur.matches al momento del gol (= matches.length)
  min: number,          // minuto (floor)
  elapsed: number,      // segundos transcurridos
  displayTime: string,  // "2:34" para mostrar
  ts: number            // Date.now()
}
```

### Match
```js
{
  team1: teamId,    // incumbente / izquierda
  team2: teamId,    // visitante / derecha
  resting: teamId,  // descansaba DURANTE este partido
  score1: number,
  score2: number,
  gk: { [teamId]: playerName }  // snapshot del arquero al momento del partido
}
```

### Standing
```js
{ pts, pj, w, d, l, gf, ga }
```

### NextMatch
```js
{ team1: teamId, team2: teamId, resting: teamId }
```

### Config (admin)
```js
{
  pin: string,     // PIN de acceso al admin ('' = sin PIN)
  nTeams: number,  // equipos (2-4, default 3)
  ppt: number,     // jugadores por equipo (3-7, default 5)
  secs: number,    // duración en segundos (default 240)
  goals: number    // goles para ganar (default 2)
}
```

---

## Estado global (`S`)

```js
const S = {
  view: 'home',          // vista activa
  hist: [],              // jornadas cerradas
  cur: null,             // jornada activa
  players: [],           // lista de nombres completos
  nicks: {},             // { [nombreCompleto]: apodo }
  cfg: {},               // config admin
  wiz: null,             // estado del wizard de setup
  score: [0, 0],         // marcador del partido en curso
  secs: 240,             // segundos restantes del timer
  running: false,        // timer corriendo
  rtab: 'hist',          // tab activo en rankings
  selPlayer: null,       // jugador seleccionado para perfil
  eplist: [],            // lista editable en PlayerEditor
  enew: '', eidx: null, eval_: '', enick: '',
  akval: '',
  admUnlocked: false,    // si el admin está desbloqueado esta sesión
  admPin: '',
  selJornada: null,      // índice en S.hist de la jornada abierta en vJornada
  editJ: null            // copia deep de jornada siendo editada en vEditJornada
};
```

### Estado del wizard (`S.wiz`)
```js
{
  step: number,          // paso actual (0=jugadores, 1=info, 2..N+1=equipos)
  date: string,          // ISO date
  theme: string,
  teams: Team[],
  ri: number,            // índice del equipo que descansa
  aiL: boolean,          // cargando IA para temática
  aiNL: number|null,     // índice equipo cargando nombre IA
  availPlayers: string[] // jugadores seleccionados para esta jornada (preselección)
}
```

---

## Lógica de negocio crítica

### Rotación de equipos (`nxM`)
```js
const nxM = ({team1, team2, resting, score1, score2}) => {
  if (score1 > score2) return { team1, team2: resting, resting: team2 };
  if (score2 > score1) return { team1: team2, team2: resting, resting: team1 };
  return { team1: team2, team2: resting, resting: team1 };  // empate: sale team1
};
```

**Error frecuente**: "el que entra" en el popup es el equipo que estaba en `resting` ANTES de llamar a `nxM`. Guardar `tr` antes de commitear.

### Fin automático de partido
```js
function addGoal(ti) {
  const ns = [...S.score]; ns[ti]++;
  // FIX crítico: durante "la última" (secs<0) cualquier gol termina el partido
  const auto = ns[0] >= cfg.goals || ns[1] >= cfg.goals || S.secs < 0;
  if (auto) stopT();
  S.score = ns;
  mGoal(team, goals, player => {
    // registrar gol...
    if (auto) doResult(ns);
    else { /* actualizar DOM parcial */ }
  });
}
```

### Actualización en tiempo real de goles
Después de registrar un gol (sin terminar el partido), actualizar el DOM directamente sin re-render completo:
```js
// Actualizar filas de jugadores en tiempo real
const pr0 = document.getElementById('PR0');
const pr1 = document.getElementById('PR1');
if (pr0) pr0.innerHTML = mkProws(t1, S.cur.goals);
if (pr1) pr1.innerHTML = mkProws(t2, S.cur.goals);
// Actualizar log de goles con minutos
const glive = document.getElementById('glive');
if (glive) glive.innerHTML = mkGoalsLog(S.cur);
```

### Goals log en tiempo real (`mkGoalsLog`)
Muestra los goles del partido actual con minuto, en layout 3 columnas (equipo izq | tiempo | equipo der):
```js
function mkGoalsLog(cur) {
  var curMatchIdx = cur.matches.length;
  var curGoals = (cur.events || []).filter(e => e.type === 'goal' && e.matchIdx === curMatchIdx);
  if (!curGoals.length) return '';
  return curGoals.map(e => {
    var isLeft = cur.nextMatch && e.team === cur.nextMatch.team1;
    var ec = tclr(cur.teams.find(t => t.id === e.team), ...);
    var minStr = e.displayTime || (Math.floor(e.elapsed/60) + ':' + ...);
    return '<div style="display:grid;grid-template-columns:1fr auto 1fr">...' + minStr + '...</div>';
  }).join('');
}
```
El `div#glive` vive dentro del `scoreboard` de `vLive`, arriba de los botones timer.

### Registro de evento de gol
```js
if (!nc.events) nc.events = [];
var _dur = S.cfg.secs || 240;
var _elapsed = S.secs <= 0 ? _dur : Math.max(0, _dur - S.secs);
var _em = Math.floor(_elapsed / 60);
var _es = _elapsed % 60;
var _edt = _em + ':' + (_es < 10 ? '0' : '') + _es;
nc.events.push({
  type: 'goal', player, team: t.id,
  matchIdx: nc.matches.length,  // ANTES de push del match
  min: _em, elapsed: _elapsed, displayTime: _edt, ts: Date.now()
});
```

### undoGoal event-aware
```js
// Busca el último evento de gol del partido actual, decrementa ese jugador específico
// (NO solo decrementa el último gol global)
var curMatchIdx = nc.matches.length;
var lastEvIdx = -1;
(nc.events || []).forEach((e, i) => {
  if (e.type === 'goal' && e.matchIdx === curMatchIdx) lastEvIdx = i;
});
if (lastEvIdx >= 0) {
  var ev = nc.events[lastEvIdx];
  if (nc.goals[ev.player] > 1) nc.goals[ev.player]--;
  else delete nc.goals[ev.player];
  nc.events.splice(lastEvIdx, 1);
}
```

### undoMatch
Revierte el último partido completo: pop de `matches`, recalcula standings, filtra eventos, restaura `nextMatch`:
```js
function undoMatch() {
  var nc = cp(S.cur);
  var lastIdx = nc.matches.length - 1;
  var last = nc.matches[lastIdx];
  nc.matches.pop();
  nc.standings = iSt(nc.teams);
  nc.matches.forEach(m => { nc.standings = apM(nc.standings, m); });
  nc.nextMatch = { team1: last.team1, team2: last.team2, resting: last.resting };
  if (nc.events) {
    nc.events = nc.events.filter(e => e.matchIdx !== lastIdx);
    // recalcular goals desde eventos restantes
    nc.goals = {};
    nc.events.filter(e => e.type === 'goal').forEach(e => {
      nc.goals[e.player] = (nc.goals[e.player] || 0) + 1;
    });
  }
  setCur(nc); S.score = [0, 0]; hmr(); render();
}
```

### Empate técnico (`isTie`)
```js
const isTie = (sorted, st) => {
  const a = st[sorted[0].id] || {pts:0,gf:0,ga:0};
  const b = st[sorted[1].id] || {pts:0,gf:0,ga:0};
  return a.pts === b.pts && (a.gf - a.ga) === (b.gf - b.ga);
};
```

### Cronómetro con cuenta negativa
```js
// fmtT: muestra '+' cuando secs < 0
const fmtT = s => {
  const a = Math.abs(s);
  return (s < 0 ? '+' : '') + Math.floor(a/60) + ':' + pad(a%60);
};
// Color: verde > 60s, dorado > 0, rojo ≤ 0
const tc = S.secs > 60 ? 'var(--green)' : S.secs > 0 ? 'var(--gold)' : 'var(--red)';
// Vibrar al llegar a 0
if (S.secs === 0) vib([100, 50, 100]);
```

---

## Sistema de apodos (`nick`)

```js
// nick() = apodo si existe, sino nombre completo
function nick(p) { return S.nicks[p] || p; }
```

- En **partidos en vivo**, goleadores, modales: se muestra `nick(p)`.
- En **tablas de stats y rankings**: se muestra `nick(p)` con el nombre completo como subtexto si hay apodo.
- Los apodos se editan en la vista Jugadores → campo "🎨 Apodo".
- Se guardan en `SK_N` como `{ [nombreCompleto]: apodo }`.
- Si se cambia el nombre completo de un jugador, el apodo se migra a la nueva key en `saveEdit()`.

```js
// dn() = "Apellido, Nombre" para la grilla de selección del wizard
const dn = p => { const l = ln(p), f = fn(p); return l === p ? p : l + ', ' + f; };
// En la grilla se usa dn(p) (no nick) para identificar correctamente al jugador
```

---

## Sistema de arqueros

- Cada partido tiene un campo `gk: { [teamId]: playerName }` en `S.cur`.
- Se resetea a `{}` al iniciar cada partido nuevo.
- El usuario lo asigna tocando el botón 🧤 bajo el nombre de cada equipo en la vista Live.
- Al commitear el resultado, el `gk` snapshot se guarda en el `Match`.
- Stats de arquero en `computeStats()`:
  ```js
  (j.matches || []).forEach(m => {
    if (!m.gk) return;
    const gk1 = m.gk[m.team1], gk2 = m.gk[m.team2];
    if (gk1) { ps[gk1].gkPj++; ps[gk1].gkGoals += m.score2; }
    if (gk2) { ps[gk2].gkPj++; ps[gk2].gkGoals += m.score1; }
  });
  ```

---

## Panel Admin (vista `admin`)

Protegido por PIN opcional. Se desbloquea con `S.admUnlocked = true` por sesión.

### Configurables
| Campo | Rango | Default |
|---|---|---|
| `nTeams` | 2–4 | 3 |
| `ppt` (jugadores/equipo) | 3–7 | 5 |
| `secs` (duración) | 60–900 (1–15 min) | 240 |
| `goals` (goles p/ganar) | 1–9 | 2 |
| `pin` | string libre | '' |

El wizard de setup lee estos valores en tiempo real. Al cambiar `nTeams`, el wizard genera N equipos.

### Colores de equipo con N equipos
```js
const CHX  = ['#4ade80', '#fbbf24', '#f97316'];  // verde, dorado, naranja (fallback)
const CSEL = ['sg', 'sb', 'so'];                  // clases para chips

// v9: paleta extendida de 8 colores personalizables por equipo
const TEAM_COLORS = [
  {h:'#4ade80', n:'Verde',   d:'#052e16'},
  {h:'#fbbf24', n:'Dorado',  d:'#3d2800'},
  {h:'#fb923c', n:'Naranja', d:'#431407'},
  {h:'#f472b6', n:'Rosa',    d:'#500724'},
  {h:'#38bdf8', n:'Azul',    d:'#0c3040'},
  {h:'#ef4444', n:'Rojo',    d:'#2d0a0a'},
  {h:'#a78bfa', n:'Violeta', d:'#1e0a3d'},
  {h:'#e2e8f0', n:'Blanco',  d:'#1e293b'},
];

// Helpers de color (usan team.color si está seteado, sino CHX[i])
function tclr(team, idx) {
  return (team && team.color) || CHX[idx] || CHX[CHX.length - 1];
}
function tclrD(team, idx) {
  var h = tclr(team, idx);
  var found = TEAM_COLORS.find(c => c.h === h);
  return found ? found.d : '#111';
}
```

El color se persiste en `team.color` dentro de la jornada. Se elige en `vWaiting` con pills de colores (`id="wclr_{teamIdx}_{colorIdx}"`). El attach de waiting actualiza `nc.teams[i].color` y llama `setCur(nc); render()`.

---

## Stats de jugadores (`computeStats`)

```js
function computeStats() {
  const ps = {};
  S.hist.forEach(j => {
    // Inicializa: { goals, mvps, champ, pj, dist: {1,2,3,4,5}, gkPj, gkGoals }
    // dist[n] = jornadas en las que el jugador hizo exactamente n goles (5 = 5+)
    if (j.goals) Object.entries(j.goals).forEach(([p, g]) => {
      ps[p].goals += g;
      const bucket = Math.min(g, 5);
      if (bucket >= 1) ps[p].dist[bucket]++;
    });
    if (j.mvp) ps[j.mvp].mvps++;
    // champ: jugadores del equipo campeón
    if (ch) ch.players.forEach(p => ps[p].champ++);
    // pj: todos los jugadores de todos los equipos
    j.teams.forEach(t => t.players.forEach(p => ps[p].pj++));
    // arquero stats (ver sección arqueros)
  });
  return ps;
}
```

### Perfil de jugador (`mProfile`)
Modal bottom-sheet con:
- PJ, goles totales, promedio goles/jornada
- 🏆 equipo ideal (veces en campeón), ⭐ MVPs
- Distribución de goles: barra horizontal por bucket (1, 2, 3, 4, 5+ goles en una jornada)
- 🧤 Como arquero: partidos atajados, goles recibidos, promedio GC

---

## Vistas (views)

```
home          → pantalla principal
setup         → wizard N+2 pasos
waiting       → previa (equipos, colores, quién descansa)
live          → jornada en vivo
end           → cierre (MVP + campeón)
ranks         → rankings (4 tabs: historial, ideal, goles, perfiles)
jornada       → detalle de jornada histórica (read-only)
editjornada   → edición de jornada histórica
players       → gestión de jugadores + apodos
admin         → panel de configuración (PIN protegido)
apikey        → configurar API key de Anthropic
```

`jornada` y `editjornada` están en `noNav` (no muestran bottom nav).

### Setup wizard (v9)

**Total pasos = 2 + nTeams**:
- **Paso 0**: `availPlayers` — ¿Quiénes juegan hoy? Chips de todos los jugadores. Botones Todos/Ninguno. Mínimo 3 para avanzar.
- **Paso 1**: Info — fecha + temática + botón IA.
- **Pasos 2..N+1**: Equipo i — nombre + jugadores (chips filtrados a `availPlayers`).

`stepNames = ['Jugadores', 'Info', 'Equipo 1', ..., 'Equipo N']`

Progress bar: `pct = (step / totalSteps) * 100`

Grilla de jugadores en pasos de equipo: usa `dn(p)` (Apellido, Nombre) — NO `nick(p)`. Filtra a `w.availPlayers` si hay seleccionados.

### vWaiting (v9 — nueva lógica)

- Muestra equipos con `tclr(t, i)` para el color del borde/nombre.
- Cada tarjeta de equipo tiene fila de **color pills** (8 colores) al pie.
- `attach()`: `on('wclr_{i}_{tci}', ...)` → `nc.teams[i].color = TEAM_COLORS[tci].h; setCur(nc); render()`.
- `wvhm` → `mConfirm` antes de salir (no navegar directo).

### vLive (v9 — nuevas features)

- Header: `←` (lvhm) | título centrado | `⚙️` (lv_gear → `mLiveMenu`)
- `lvhm` → `mConfirm('Salir a inicio', ...)` antes de navegar.
- `div#glive` vive dentro del scoreboard, entre la grilla de scores y los botones timer. Se actualiza sin re-render completo.
- Colores: `c1 = tclr(t1, i1)`, `c2 = tclr(t2, i2)`, `tclr(tr, itr)` para equipo que descansa.
- `stHTML` recibe `cur.champion` como tercer argumento.

### mLiveMenu (v9)
Modal con opciones del partido en vivo:
- **↩ Revertir último partido** → `undoMatch()` (deshabilitado si no hay partidos)
- **🔀 Cambiar próximo partido** → `mNextMatch()`
- **⏱ Resetear reloj** → `rstT()`
- **Cancelar**

### vJornada (v9)
Vista detalle de jornada histórica. Accesible desde el historial en vRanks.
- Header: `←` (jvbk → ranks/hist) | título | `✏️ Editar` (jvedit)
- Cards: campeón, equipos con `mkFormation()`, posiciones, goleadores, partidos
- `S.selJornada` = índice en `S.hist`

### vEditJornada (v9)
Editor de jornada histórica. Usa `S.editJ` (deep copy).
- Editar scores de cada partido (+/-), eliminar partido (recomputa standings)
- Editar goles por jugador (+/-), agregar goleador desde select
- Elegir campeón (botones por equipo + "Sin campeón")
- Elegir MVP (chips de jugadores + "Sin MVP")
- Ver posiciones recalculadas en tiempo real
- `ej_save` → `setHist(nh)`, vuelve a jornada

### Tabla de posiciones `stHTML` (v9)

Columnas: **# | Equipo | G | E | P | PJ | PTS** — siempre visibles, sin expandable.
- G (verde), E (dorado), P (rojo)
- CSS grid: `22px 1fr 26px 26px 26px 26px 34px`
- Firma: `stHTML(teams, standings, championId)`

```js
// Columnas visibles inline en cada fila:
'<div class="st-row">'
  + pos + nombre + (s.w||0) + (s.d||0) + (s.l||0) + s.pj + s.pts
+ '</div>'
```

### Live view — hero card

Cronómetro + marcadores en **una sola tarjeta** al tope, con layout grid 3 columnas:
```
[Equipo1 nombre + score + GK] [Timer + ÚLTIMA] [Equipo2 nombre + score + GK]
[  botón +1 gol E1           ]                 [  botón +1 gol E2           ]
[  lista jugadores E1 (PR0)  ]                 [  lista jugadores E2 (PR1)  ]
[  Descansa: NombreEquipo                                                   ]
[  ▶ Iniciar / ⏸ Pausa     ]  [↺ Reset]
```

IDs relevantes en el DOM live:
- `TD` — timer text
- `TS` — label "ÚLTIMA" (display:flex cuando secs<0)
- `SC0`, `SC1` — scores (se actualizan parcialmente)
- `PR0`, `PR1` — listas de jugadores (se actualizan parcialmente tras gol)
- `TB` — botón timer
- `G0`, `G1` — botones de gol
- `GK0`, `GK1` — botones arquero

### Rankings — 4 tabs

1. **Historial**: jornadas en orden inverso. Muestra primer nombre (`fn(p)`) de los jugadores del campeón para ahorrar espacio.
2. **Equipo Ideal**: tabla con 🏆/⭐/PJ. Nombre de jugador es botón que abre `mProfile`.
3. **Goleadores**: tabla con ⚽/Prom/PJ. Nombre también abre `mProfile`.
4. **Perfiles**: lista de todos los jugadores con mini-stats, botón "Ver" abre `mProfile`.

---

## Modales (bottom-sheet style)

Todos los modales montan desde abajo (`align-items: flex-end`), con pill indicator y scroll interno.

```js
// Patrón base de modal
mr.innerHTML = `<div class="modal-bg" id="mbb">
  <div class="modal">
    <div class="modal-pill"></div>
    <!-- contenido -->
  </div>
</div>`;
// Cerrar tocando el fondo
document.getElementById('mbb').onclick = e => { if (e.target.id === 'mbb') hmr(); };
```

| Modal | Función |
|---|---|
| `mConfirm` | Confirmación genérica (msg, sub, onYes, btnText, btnClass) |
| `mGoal` | Seleccionar goleador. Muestra `nick(p)` + `⚽x N` acumulados |
| `mGK` | Seleccionar arquero. Botón activo si ya está asignado |
| `mResult` | Resultado partido. Grid 2 col: SALE (rojo) / ENTRA (azul) |
| `mMVP` | Seleccionar MVP. Lista con `nick(p)` |
| `mTema` | Opciones de temática generadas por IA |
| `mKey` | Ingresar API key de Anthropic |
| `mProfile` | Perfil completo de jugador (ver sección stats) |
| `mLiveMenu` | Opciones del partido en vivo (undo, next, reset reloj) |
| `mNextMatch` | Override manual del próximo partido |

---

## Integración con Anthropic API

```js
async function callAI(prompt) {
  const key = gk_();
  if (!key) throw new Error('NO_KEY');
  const r = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': key,
      'anthropic-version': '2023-06-01',
      'anthropic-dangerous-allow-browser': 'true'
    },
    body: JSON.stringify({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 800,
      messages: [{ role: 'user', content: prompt }]
    })
  });
  if (!r.ok) throw new Error('API ' + r.status);
  const d = await r.json();
  return (d.content && d.content[0] && d.content[0].text) || '';
  // NO usar d.content?.[0]?.text — optional chaining falla en Safari iOS antiguo
}
```

### Generación de temáticas
- JSON estricto: `[{tema, equipos: [string×3]}]` × 3 opciones.
- Evitar las últimas 8 temáticas usadas.
- Al seleccionar: pre-carga `theme` y nombres de equipos en el wizard.

### Generación de nombre de equipo
- Equipo 0 (defensor): tono de "defiende el título".
- Devuelve solo el nombre, sin comillas ni explicación.
- Solo habilitado si la temática está cargada.

---

## Sistema de diseño

### Paleta (verde/dorado del flyer de La Pachanga)

```css
:root {
  /* Fondos — oscuros verdosos */
  --bg:   #070e07;
  --sur:  #0e1a0e;
  --sur2: #152015;
  --bor:  #1e401e;

  /* Texto */
  --txt:  #e4f0e4;
  --txt2: #7aa87a;
  --txt3: #4a664a;

  /* Colores de equipo */
  --green:    #4ade80;  --gd:      #052e16;  /* E1 */
  --gold:     #fbbf24;  --gold-d:  #3d2800;  /* E2 */
  --orange:   #f97316;  --orange-d:#431407;  /* E3 */

  /* Semánticos */
  --red:    #ef4444;  --rd: #2d0a0a;   /* error, GC, destruir */
  --blue:   #38bdf8;  --bd: #0c3040;   /* "entra al campo" */
  --purple: #a78bfa;                   /* IA */
}
```

### Colores de equipo
```js
const CHX  = ['#4ade80', '#fbbf24', '#f97316'];
const CSEL = ['sg', 'sb', 'so'];
// CSS:
// .pchip.sg { background: #052e16; border-color: var(--green); color: var(--green); }
// .pchip.sb { background: #3d280044; border-color: var(--gold); color: var(--gold); }
// .pchip.so { background: #43140755; border-color: var(--orange); color: var(--orange); }
// .pchip.taken { opacity: .18; pointer-events: none; }
```

### Clases de utilidad

| Clase | Uso |
|---|---|
| `.page` | Wrapper de vista. max-width 440px, padding con safe-area-inset-bottom |
| `.card` | Tarjeta con borde y border-radius 14px |
| `.btn` | Botón full-width con `:active { transform: scale(.97) }` |
| `.btn-green/gold/ghost/red/dis` | Variantes de botón |
| `.btn-end` | Botón "Terminar Pachanga" (rojo oscuro, borde rojo) |
| `.btn-ai` | Botón IA (fondo púrpura oscuro) |
| `.btn-sm` | Botón pequeño inline |
| `.goal-btn` | Botón grande de gol con `:active { transform: scale(.93) }` |
| `.hero-card` | Tarjeta del partido en vivo |
| `.gk-btn` | Botón de arquero (`.active` = azul) |
| `.pchip` | Chip de jugador en wizard |
| `.prow` | Fila jugador en live (nombre + ⚽xN) |
| `.prog-bar/.prog-fill` | Barra de progreso del wizard |
| `.modal-bg/.modal/.modal-pill` | Sistema de modales bottom-sheet |
| `.sbb` | Botón ← de retorno (min 44×44px) |
| `.timer` | Cronómetro (68px, tabular-nums) |
| `.score-num` | Marcador (56px, tabular-nums) |
| `.cfg-row` | Fila de configuración en admin |
| `.num-ctrl` | Control numérico +/- para admin |
| `.profile-stat` | Fila de estadística en perfil |
| `.dist-bar` | Barra de distribución de goles |
| `.tcolor-pill` | Pill circular de color de equipo (24×24px, `.sel` = borde blanco + scale 1.25) |
| `.tcolor-row` | Fila de pills de color (flex, gap) |
| `.formation` | Grid de formación de equipo en vJornada |
| `.f-row` | Fila dentro de formation (flex wrap) |
| `.f-player` | Chip de jugador en formation |

### Diseño v2 (index_local_v2)
- **Fuentes**: Anton (números grandes, títulos) + Barlow Condensed (labels, tablas)
- **Bottom nav** (`#bnav`): 4 tabs — Home, Live, Rankings, Jugadores. Oculto en `noNav`.
- **Paleta**: fondo muy oscuro (`#070e07`), acentos verde/dorado, inputs con `border-radius` redondeado.

### UX mobile
- `-webkit-tap-highlight-color: transparent` en `*`
- `font-size: 16px` en inputs (evita zoom de Safari)
- `safe-area-inset-bottom` en padding del `.page` y modales
- `overscroll-behavior: none` en body
- `window.scrollTo(0, 0)` en cada navegación y cambio de paso del wizard
- Vibración: `navigator.vibrate(ms)` — 40ms al marcar gol, 20ms al seleccionar jugador, [100,50,100] al llegar a 0 el timer
- Meta tags PWA: `apple-mobile-web-app-capable`, `apple-mobile-web-app-status-bar-style`, `theme-color`

---

## Patrones de código

### Render + attach
```js
function render() {
  const html = { home: vHome, setup: vSetup, ... }[S.view];
  document.getElementById('app').innerHTML = html ? html() : '';
  attach();
}
function attach() {
  // reasignar todos los event listeners según S.view
}
```

### Helper `on`
```js
function on(id, fn) { const el = g(id); if (el) el.onclick = fn; }
```

### Deep copy
```js
const cp = x => JSON.parse(JSON.stringify(x));
```

### Ordenar por apellido
```js
const sbl = arr => [...arr].sort((a, b) => {
  const c = ln(a).localeCompare(ln(b), 'es', { sensitivity: 'base' });
  return c !== 0 ? c : fn(a).localeCompare(fn(b), 'es', { sensitivity: 'base' });
});
```

### Sin optional chaining (`?.`)
**Nunca usar `?.`** — falla en versiones antiguas de Safari iOS. Usar en su lugar:
```js
// MAL:  d.content?.[0]?.text
// BIEN: d.content && d.content[0] && d.content[0].text
// MAL:  j.teams?.find(...)
// BIEN: j.teams && j.teams.find(...)
```

### Goles mostrados
```js
// En lugar de '⚽'.repeat(n) usar:
g > 0 ? `⚽x${g}` : ''
```

---

## Jugadores por defecto (19)

```
Agustín Begué, Bruno Pertile, Facundo Fernandez, Federico Fiori,
Gaspar Botti, Gaspar del Río, Ignacio Mazzi, Manuel Brussino,
Mateo Bonacalza, Mateo Caño, Matías Fernandez, Nahuel Torres Arata,
Nicolas Moguilevsky, Ramiro Hoqui, Tobías Botti, Tomás Dominguez,
Tomás Ingaramo, Tomás Rendo, Toti Manza
```

Los acentos deben estar como UTF-8 directo en el HTML. **No usar escapes `\xNN` de Python** al escribir el archivo — generan `\xedn` literal en el JS. Escribir el contenido directamente como cadena unicode.

---

## Extensiones futuras sugeridas

- **Control por voz**: Web Speech API de Safari (iOS 14.5+). Comandos: "gol equipo uno", "empezar", "terminar". Requiere un toque inicial para activar el micrófono. Factible pero con limitaciones en ambientes ruidosos.
- **Marcador compartido en tiempo real**: Supabase o Firebase para que todos vean desde sus celuslares simultáneamente.
- **Generador de flyer**: exportar card del campeón + equipos como PNG con `html2canvas`.
- **Modo espectador vs organizador**: el organizador carga resultados, el resto solo lee.
- **Backup/restore**: exportar/importar historial completo en JSON.
- **Notificaciones push**: avisar que empieza la pachanga.

---

## Notas de versión

| Keys | Versión | Archivo | Cambios principales |
|---|---|---|---|
| `ph9/pc9/pp3/pak/pnk9/pcfg9` | **v9 (actual)** | `index_local_v2.html` | Diseño v2 (Anton/Barlow/bottom nav), colores de equipo personalizables (8 opciones), `GoalEvent` con minuto, `mkGoalsLog` en tiempo real, `undoGoal` event-aware, `undoMatch`, `mLiveMenu`, `mNextMatch`, `vJornada`, `vEditJornada`, `mkFormation`, tabla posiciones G/E/P siempre visible, wizard con paso 0 de `availPlayers`, `recomputeJornada`, `_saveResumen`, historial con Ver/Borrar |
| `ph9/pc9/pp3/pak/pnk9/pcfg9` | v8 | `index_local.html` | Vanilla JS, paleta verde/dorado, apodos, arqueros, panel admin, perfiles de jugadores, fix "la última", goles en tiempo real (PR0/PR1), bottom-sheet modals, PWA |
| `ph9/pc9/pp3/pak` | v7 | — | Vanilla JS, paleta verde/dorado, UX mejorada |
| `ph9/pc9/pp3/pak` | v6 | — | Migración React → Vanilla JS para fix Safari local |
| `pach_h8/pach_c8/pach_p2` | v5 | — | React + CDN, grid jugadores, popup resultado, IA temáticas |
