# PachaBets — Architecture Reference

## State object (S)

```javascript
const S = {
  // ── Config ──
  mode: 'player',            // 'player' | 'org'
  orgPin: '1234',
  orgUsername: 'igna',
  orgUnlockedUntil: 0,       // timestamp — PIN sesión dura 1h
  rake: 0.10,
  status: 'open',            // 'open' | 'closed' | 'resolved'
  alias: 'la.pachanga',      // alias MP del org
  mpNombre: '',

  // ── Early Bird ──
  earlyBird: { active:false, bonus:0.10, everOpen:false },

  // ── Bets (current jornada) ──
  bets: { campeon:[], goleador:[], primero:[], historico:[], equipos:[] },
  // bet shape: {id, player, playerAlias, option, optionLabel, amount,
  //             ref, batchRef, payStatus, cobroStatus, isEarlyBird, oddsAtTime}

  // ── Odds pool (persiste entre jornadas, reset solo en "Borrar todo") ──
  oddsPool: { campeon:[], goleador:[], primero:[], historico:[], equipos:[] },
  // pool item: {option, amount}

  // ── Props (desafíos P2P) ──
  props: [],
  // prop shape: {id, desc, creator, status, subject, condition, threshold,
  //              isCustom, forAmount, forBackers[], againstAmount, againstBackers[],
  //              resolved, actualGoals, won, matchedForAmount, matchedAgainstAmount,
  //              excessRefund}

  // ── Cart ──
  cart: [],
  // cart item: {id, player, playerAlias, market, option, optionLabel, amount,
  //             isEarlyBird, oddsAtTime}

  // ── Selections (current UI state) ──
  sel: { campeon:null, goleador:null, primero:null },
  expandedMarkets: { campeon:false, goleador:false, primero:false },

  // ── Payouts (after resolveAll) ──
  payouts: [],
  // payout shape: {player, playerAlias, invested, gross, net, status, referralEarned}

  // ── Resolution ──
  resolution: { campeon:null, goleador:null, primero:null, resolved:false },

  // ── History ──
  jornadaHistory: [],
  // jornadaArchived: {jornadaId, date, resolution, bets, payouts}

  // ── Jornada date ──
  jornadaDate: null,         // ISO string, próximo viernes

  // ── Bet limits ──
  betLimitDefault: 20000,
  betLimitExtensions: {},    // {playerName: extraAmount}
  betLimitRequests: {},      // {playerName: {player, requested, status}}

  // ── Aliases ──
  aliases: {},               // {playerName: alias}
};
```

---

## AUTH object

```javascript
const AUTH = {
  currentUser: null,         // username string
  users: {
    username: {
      password: string,
      fullName: string,      // "Bruno Pertile" — identifier in bets
      alias: string,         // MP alias
      referredBy: string|null,
      isOrg: bool            // only for igna
    }
  }
}
```

---

## Storage keys

```
// PachaBets (escritas por apuestas.html):
pachanga_auth_v1        → AUTH object
pachanga_bets_v4        → S.bets
pachanga_props_v4       → S.props
pachanga_pays_v4        → S.payouts
pachanga_aliases_v4     → S.aliases
pachanga_history_v4     → S.jornadaHistory
pachanga_odds_pool_v4   → S.oddsPool
pachanga_limits_v4      → {extensions, requests}
pachanga_cfg_v4         → rake, status, orgPin, orgUsername, alias, mpNombre,
                          mode, resolution, earlyBird, orgUnlockedUntil, jornadaDate

// La Pachanga tracker (escritas por index_local.html, leídas por apuestas.html):
pachanga_sync_v1        → { status, date, teams:[{id,name,players[]}],
                             allPlayers:string[], goals:{player:count},
                             firstGoaler, champion, topScorer,
                             teamGoals:{teamName:count}, teamWins:{teamName:count} }
ph9                     → JornadaHistorial[]
                          // Cada jornada: { date, teams:[{id,name,players[]}],
                          //   goals:{player:count}, standings:{teamId:{pj,w,gf,ga}},
                          //   mvp, matches:[{team1,team2,score1,score2,gk:{}}] }
```

---

## Bridge con La Pachanga — funciones clave

```javascript
// Cache variables (module-level)
var _cachedSync = undefined  // undefined=no cargado, null=no hay jornada
var _cachedPh9  = null       // null=re-leer localStorage, array=cacheado

// Lectura
_readSync()               // lee pachanga_sync_v1 con cache
_readPh9()                // lee ph9 con cache (_cachedPh9)
_refreshStats()           // fuerza _cachedPh9=null, re-renderiza mercados

// Datos de jornada activa
_getTeams()               // equipos de la jornada (fallback a hardcoded)
_getAllPlayers()           // fusión: sync.allPlayers + ph9 history + _PLAYERS_FB
_getJorpPlayers()         // jugadores de la jornada actual (para primer goleador)
_getLiveGoals()           // goles en tiempo real desde sync

// Stats históricas desde ph9
_getPlayerStats()         // {player: {pj, dist:{1:n,2:n,...,6:n}}} — distribución por bucket
_getTeamJornadaStats(nm)  // [{goals, wins}] por jornada donde participó el equipo
_getGoalWeights(opts)     // {player: weight} normalizado, para dampening ponderado
_histFreq(p,cond,thr)     // frecuencia histórica de condición para jugador (con pj>=3)
_teamHistFreq(nm,sub,c,t) // frecuencia histórica para equipo (con rows>=3)
_getFreq(type,nm,sub,c,t) // wrapper: real si hay data, Poisson fallback si no

// Seed odds desde stats
_histSeedOdds(mkt,opt)       // goleador/primero: closeness [1x-3x], campeon: xG-based
_histCondSeedOdds(p,cond,t)  // goles del jugador: closeness a condición [1x-3x]

// UI
_updateStatsStatus()      // barra "📊 N jornadas · prom X.Xx/j [🔄 Actualizar]"
_onSyncUpdate(d)          // callback cuando llega nueva jornada sync
```

---

## _getAllPlayers() — fusión de fuentes

```javascript
function _getAllPlayers() {
  var d = _readSync()
  var base = (d?.allPlayers?.length) ? [...d.allPlayers] : _PLAYERS_FB.slice()
  // Merge jugadores del historial ph9 que no estén en base
  _readPh9().forEach(j => {
    j.teams?.forEach(t => {
      t.players?.filter(Boolean).forEach(p => {
        if (!seen[p]) { seen[p]=true; base.push(p) }
      })
    })
  })
  return base
}
// Llamada dinámicamente en mktOpts() — siempre fresca, no cached
```

---

## Option key formats

```javascript
// Mercados estándar: string plano
"Los Peronistas"              // campeon
"Bruno Pertile"               // goleador, primero

// Goles del jugador (historico):
"Bruno Pertile|gte|3"         // player|condition|threshold
// conditions: gte|gt|lte|lt|eq

// Equipos:
"Los Peronistas|goles|gte|5|si"    // team|sub|cond|threshold|side
"Los Peronistas|wins|lte|2|no"     // sub: goles|wins, side: si|no
```

---

## Key flows

### Confirmar pago
```javascript
S.cart.forEach(item => {
  S.bets[item.market].push({ ...item, payStatus:'confirmed', cobroStatus:'pending', batchRef })
  S.oddsPool[item.market].push({ option:item.option, amount:item.amount })
})
S.cart = []
```

### Reiniciar jornada (mantiene oddsPool e historial)
```javascript
S.bets = {campeon:[],goleador:[],primero:[],historico:[],equipos:[]}
S.props = []; S.cart = []; S.payouts = []
S.resolution = {campeon:null,goleador:null,primero:null,resolved:false}
S.betLimitExtensions = {}; S.betLimitRequests = {}
S.jornadaDate = getNextFriday(S.jornadaDate)
// Archiva jornada actual si tenía apuestas y no estaba resuelta
```

### Borrar todo
```javascript
// Limpia TODO: S, oddsPool, jornadaHistory
// Remueve todas las keys localStorage de pachanga_*
// Doble confirmación (iOS no tiene confirm() nativo)
```

---

## Fallback constants (hardcoded en apuestas.html)

```javascript
var _PLAYERS_FB = ['Agustín Begué','Bruno Pertile','Facundo Fernandez',
  'Federico Fiori','Gaspar Botti','Gaspar del Río','Ignacio Mazzi',
  'Manuel Brussino','Mateo Bonacalza','Mateo Caño','Matías Fernandez',
  'Nahuel Torres Arata','Nicolas Moguilevsky','Ramiro Hoqui','Tobías Botti',
  'Tomás Dominguez','Tomás Ingaramo','Tomás Rendo','Toti Manza']

var _TEAMS_FB = [
  {name:'Equipo 1',short:'E1',players:[]},
  {name:'Equipo 2',short:'E2',players:[]},
  {name:'Equipo 3',short:'E3',players:[]}
]
// Estos se usan si pachanga_sync_v1 no está disponible
```

---

## Navigation tabs

```javascript
// Player mode: Apostar · Desafíos · Mis apuestas
// Org mode:    Apostar · Desafíos · Cobros · Apuestas · Resultados · Pagos · Historial · Config
// Apostar screen: campeon + goleador + primero + goles-jugador (collapsed) + equipos (collapsed)
```
