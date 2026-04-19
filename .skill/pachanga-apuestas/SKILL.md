---
name: pachanga-apuestas
description: >
  Sistema completo de apuestas (PachaBets) para La Pachanga, pachanga de fútsal semanal.
  Un único archivo HTML standalone — sin servidor, sin instalación, se comparte por WhatsApp.
  Autenticación por sesión, 5 mercados de apuestas, desafíos P2P multi-participante,
  panel de organizador con PIN, flujo de pago via alias, odds parimutuel + stats-based
  desde historial de PachaStats (ph9), rango [1.0x–3.0x] por closeness métrica,
  límite de $20k por usuario, referidos, disclaimer anti-complot, hot/cold indicators.
  Usar este skill siempre que el usuario pida modificar, extender o integrar PachaBets.
---

# PachaBets — Sistema de Apuestas La Pachanga

## Archivo de trabajo

**`C:\Users\ignac\OneDrive\Escritorio\Proyectos\Pachabets\la-pachanga\apuestas.html`**

- Siempre editar este archivo. No usar los assets del skill como base de trabajo.
- Repo: `https://github.com/ignamaxi/la-pachanga.git` (branch `main`)
- Deploy: Vercel, auto-deploy en cada push. URL: `/apuestas`
- Vercel config en `vercel.json` — headers ya tienen `Cache-Control: no-cache` para HTML

## Referencias

| Archivo | Cuándo leerlo |
|---|---|
| `references/architecture.md` | State completo, storage keys, data shapes, flujos, bridge PachaStats |
| `references/odds-engine.md` | Motor de odds, closeness metric, stats-based seeding, dampening |
| `references/ui-components.md` | Design tokens, CSS classes, componentes, patrones UI |

---

## Stack

- Vanilla JS + CSS variables, un solo `.html` (sin build, sin servidor)
- `localStorage` para toda la persistencia
- Firebase Firestore para sync en tiempo real con La Pachanga tracker
- Fonts CDN: DM Sans, DM Mono, Bebas Neue
- iOS-safe: `maximum-scale=1.0`, inputs con `font-size:16px`

---

## Integración con PachaStats (La Pachanga tracker)

**YA IMPLEMENTADA.** El bridge lee datos de La Pachanga via dos canales:

### Canal 1 — Jornada activa (tiempo real)
```javascript
var _SYNC_KEY = 'pachanga_sync_v1'   // localStorage compartido
// Firestore: collection 'jornada', doc 'activa'
// Shape: { status, date, teams, allPlayers, goals, firstGoaler,
//           champion, topScorer, teamGoals, teamWins }
```
- `_readSync()` — lee sync con cache
- `_getTeams()` — equipos de la jornada (fallback a hardcoded)
- `_getAllPlayers()` — lista completa de jugadores (sync + ph9 + fallback)
- `_getJorpPlayers()` — jugadores de la jornada (para mercado "primer goleador")
- `_getLiveGoals()` — goles en tiempo real

### Canal 2 — Historial acumulado (estadísticas)
```javascript
// La Pachanga escribe: localStorage['ph9'] = JornadaHistorial[]
// Firestore: collection 'historial', doc 'all'
// Cada jornada: { date, teams:[{name,players[]}], goals:{player:count},
//                 standings:{teamId:{pj,w,gf,ga}}, mvp, matches }
```
- `_readPh9()` — lee historial con cache (`_cachedPh9`)
- `_getPlayerStats()` — computa distribución de goles por jugador desde ph9
- `_getTeamJornadaStats(teamName)` — historial de goals/wins por jornada para un equipo
- `_getGoalWeights(opts)` — pesos normalizados de gol por jugador (para dampening)

### Refresh manual
```javascript
function _refreshStats()       // limpia _cachedPh9, re-renderiza mercados, muestra toast
function _updateStatsStatus()  // actualiza barra "📊 N jornadas · prom X.Xx/j [🔄 Actualizar]"
```
La barra de stats aparece debajo del header de jornada en la pantalla Apostar.

### _getAllPlayers() — fusión de fuentes
```javascript
// Orden de prioridad y merge:
// 1. sync.allPlayers (lista maestra de PachaStats)
// 2. + jugadores en ph9 que no estén en la lista → evita jugadores faltantes
// 3. Fallback: _PLAYERS_FB (19 jugadores hardcodeados)
```

---

## Modos de usuario

**Modo jugador** (default): Apostar · Desafíos · Mis apuestas
**Modo organizador** (solo usuario `igna`, PIN 4 dígitos, sesión 1h): + Cobros · Apuestas · Resultados · Pagos · Historial · Config

El botón 🔒 es invisible para usuarios que no sean el org.
Al ingresar PIN correcto, la sesión dura 1 hora. Al salir del modo org la sesión se invalida.

---

## Registro y autenticación

```
Campos: nombre + apellido + usuario + contraseña + alias (opcional) + código referido (opcional)
- Checkbox de disclaimer obligatorio (anti-complot, términos)
- fullName = nombre + apellido → identificador del apostador en todos los mercados
- Referido: 10% de lo que la org gana de ese usuario va al referidor (se calcula en resolveAll)
- Usuario org (igna/123) se auto-crea en init() si no existe
```

---

## Flujo de apuesta

```
1. "Apostando como: Bruno Pertile" — desde sesión, sin selector
2. Selecciona opción → ve odd actual (stats-based o parimutuel, floor 1.0x)
3. Ingresa monto → preview: "Cobrás si ganás: $X" — exacto, no orientativo
4. "+ Agregar" → carrito, odd bloqueada (🔒)
5. "Pagar todo →" → modal: alias org, monto, batchRef único PCH-YYYYMMDD-XXXX
6. "Ya pagué ✓" → confirmed inmediato
7. Org: Cobros → confirma o rechaza batch entero
```

---

## Mercados (todos dentro de la tab "Apostar")

### 1–3. Mercados principales (parimutuel + stats-seeding)
| Mercado | Opciones | Odds seed |
|---|---|---|
| 🏆 Campeón de la jornada | equipos de la jornada | xG del equipo (suma avg goals jugadores) |
| ⚽ Goleador de la noche | todos los jugadores | closeness histórica [1.0x–3.0x] |
| ⚡ Primer goleador | jugadores de la jornada | closeness histórica [1.0x–3.0x] |

Muestra `📊 HIST` badge cuando la odd proviene de stats históricas.
Muestra `BASE` badge cuando es seed sin datos.
🔥/🧊 hot/cold basado en `avg > leagueAvg×1.6` / `avg < leagueAvg×0.35`.

### 4. Goles del jugador — esta jornada (parimutuel pool único)
- Opciones dinámicas: `Jugador|condición|threshold` (ej: `Rendo|gte|3`)
- Todas las combinaciones compiten en el mismo pozo
- Múltiples opciones pueden ganar si se cumplen varias condiciones
- Odds seed: closeness histórica a la condición [1.0x–3.0x], sin historial → 1.5x
- En Resultados: org ingresa goles reales por jugador → sistema evalúa cuáles condiciones pasan

### 5. Equipos — dos sub-mercados separados
- **⚽ Goles del equipo**: condición + slider 0–15 goles
- **🏅 Victorias del equipo**: condición + slider 0–10 victorias
- Cada par (equipo+sub+cond+threshold) es un mini-mercado SÍ/NO independiente
- En Resultados: org ingresa valor real → sistema evalúa SÍ o NO

Los mercados 4 y 5 están colapsados (▼) dentro de Apostar.

---

## Motor de odds — resumen

Ver `references/odds-engine.md` para detalle completo. Principios clave:

**Cuando no hay apuestas en la opción → odds seed desde stats:**
- Goleador / Primero: `closeness = avg(goals / max_goals_jornada)` → [1.0x, 3.0x]
- Goles del jugador: `closeness` específica por condición (gte/lte/eq) → [1.0x, 3.0x]
- Sin historial (0 jornadas jugadas): siempre 1.50x
- Nunca cumplió la condición: 3.00x (máximo)

**Cuando hay apuestas reales → parimutuel con dampening:**
- Virtual pool ODDS_DAMP=$5000 distribuido por pesos de stats (no uniforme)
- Odds convergen a parimutuel puro a medida que crece el volumen real

---

## Desafíos P2P

```javascript
// Shape:
{
  id, desc, creator, status,          // pending|partial|matched|cancelled
  subject, condition, threshold,      // jugador|gte/gt/lte/lt/eq|goles
  isCustom,                           // true para desafíos de texto libre
  forBackers:[{player,amount,payStatus,cobroStatus,ref}],
  againstBackers:[...],
  forAmount, againstAmount,
  resolved, actualGoals, won,
  matchedForAmount, matchedAgainstAmount,
  excessRefund: {player, amount}
}
```

- **Cualquier usuario** puede crear desafíos (estructurado o texto libre)
- **Multi-participante**: ambos lados pueden tener múltiples apostadores → payout proporcional
- **Pago requerido** del creador Y del aceptante antes de activarse
- Al cerrar apuestas: parciales se liquidan en min(for, against), exceso se devuelve
- Rake solo sobre las ganancias del ganador (10%)
- Sin límite de monto (desafíos no cuentan para el límite de $20k)

---

## Límite de apuestas

```javascript
const LIMIT_DEFAULT = 20_000  // por usuario por jornada (solo mercados principales)
// Desafíos NO cuentan para el límite
// Solicitar ampliación: aparece en Cobros (panel org)
// Org aprueba +$10k o +$20k
// Límite se resetea al reiniciar jornada
```

---

## Storage keys

```
pachanga_auth_v1        → {currentUser, users:{user:{password,fullName,alias,referredBy}}}
pachanga_bets_v4        → {campeon:[], goleador:[], primero:[], historico:[], equipos:[]}
pachanga_props_v4       → Prop[]
pachanga_pays_v4        → Payout[]
pachanga_aliases_v4     → {playerName: alias}
pachanga_history_v4     → JornadaArchived[]
pachanga_odds_pool_v4   → {campeon:[], goleador:[], primero:[], historico:[], equipos:[]}
pachanga_limits_v4      → {extensions:{}, requests:{}}
pachanga_cfg_v4         → {rake, status, orgPin, orgUsername, alias, mpNombre,
                           mode, resolution, earlyBird, orgUnlockedUntil, jornadaDate}

// Keys escritas por La Pachanga tracker (compartidas en el mismo origin):
pachanga_sync_v1        → { status, date, teams, allPlayers, goals, ... }
ph9                     → JornadaHistorial[]  (array de jornadas pasadas)
```

---

## Deploy

```bash
cd "C:\Users\ignac\OneDrive\Escritorio\Proyectos\Pachabets\la-pachanga"
git add apuestas.html
git commit -m "descripción"
git push origin main   # Vercel auto-deploya en ~30s
```

---

## Configuración inicial

1. Org ingresa con usuario `igna`, contraseña `123`
2. Config → alias (default: `la.pachanga`) → Guardar
3. Config → cambiar PIN (default: `1234`)
4. Ajustar rake (default 10%)
5. Compartir URL `/apuestas` — cada jugador se registra con nombre, apellido y contraseña
6. La Pachanga tracker debe estar en el mismo origin (mismo Vercel) para compartir localStorage
