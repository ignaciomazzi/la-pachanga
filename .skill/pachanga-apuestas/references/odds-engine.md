# PachaBets — Odds Engine Reference

## Core constants

```javascript
const SEED_ODDS_CAP = 1.50   // odds antes de apuestas reales (mercados sin stats)
const ODDS_DAMP = 5000       // virtual pool para suavizar extremos ($5.000)
```

---

## Principio general

Las odds tienen **dos capas**:

1. **Seed (sin apuestas reales en la opción)** → derivadas de estadísticas históricas de `ph9`
2. **Parimutuel (con apuestas reales)** → virtual dampening ponderado por stats

El tipo de odds se indica con un badge:
- `📊 HIST` (verde) — calculadas desde historial real
- `BASE` (gris) — seed sin datos históricos (1.50x)
- *(sin badge)* — odds parimutuel reales

---

## Closeness metric — núcleo del sistema

**Para mercados goleador, primero, y goles del jugador.**

La métrica mide cuán cerca estuvo históricamente cada jugador de cumplir la condición,
acumulando jornada a jornada desde `ph9`.

```javascript
// Para goleador / primero:
// score = avg(goals_jugador / max_goals_esa_jornada) por jornada jugada
// max_goals = máximo anotado por cualquier jugador en esa jornada
score = scoreSum / pj   // ∈ [0, 1]
// score 1.0 = siempre fue el máximo goleador
// score 0.0 = nunca metió un gol

// Para goles del jugador (condición específica):
// gte/gt: closeness = min(goals / threshold, 1)
// lte/lt: closeness = 1 si cumplió, degradado = limit/goals si se pasó
// eq:     closeness = 1 si exacto, 1 - |goals-threshold|/threshold si no

// Mapeo lineal inverso a odds:
val = 1.00 + 2.00 * (1 - score)
// score 0.0 → 3.00x (máximo, nunca cumplió)
// score 0.5 → 2.00x (promedio)
// score 1.0 → 1.00x (siempre lo cumplió)
// Sin jornadas jugadas → 1.50x (seed neutro)
// Cap: Math.min(3.00, Math.max(1.00, val))
```

---

## _histSeedOdds — mercados goleador / primero

```javascript
function _histSeedOdds(market, option) {
  // Campeon: xG-based (suma avg goles de jugadores del equipo)
  if (market === 'campeon') { /* ... xG lógica ... */ }

  // Goleador / Primero: closeness metric
  var hist = _readPh9()
  var data = {}  // {player: {pj, scoreSum}}
  opts.forEach(p => data[p] = {pj:0, scoreSum:0})

  hist.forEach(j => {
    var maxG = Math.max(...Object.values(j.goals||{}), 1)
    jPlayers.forEach(p => {
      data[p].pj++
      data[p].scoreSum += (j.goals[p]||0) / maxG
    })
  })

  var d = data[option]
  if (!d || d.pj === 0) return {val: 1.50, type: 'hist'}  // sin historial
  var score = d.scoreSum / d.pj
  var val = 1.00 + 2.00 * (1 - score)
  return {val: clamp(val, 1.00, 3.00), type: 'hist'}
}
```

---

## _histCondSeedOdds — mercado goles del jugador

```javascript
function _histCondSeedOdds(player, cond, threshold) {
  var hist = _readPh9()
  // Para cada jornada que jugó el player, calcula closeness a la condición
  hist.forEach(j => {
    if (!jPlayers.includes(player)) return
    pj++
    var goals = j.goals[player] || 0
    var thr = Math.max(1, threshold)
    if (cond==='gte'||cond==='gt') {
      closeness = Math.min(1, goals / (cond==='gt'?threshold+1:threshold))
    } else if (cond==='lte'||cond==='lt') {
      var limit = cond==='lt'?threshold-1:threshold
      closeness = goals<=limit ? 1 : Math.max(0, limit/goals)
    } else if (cond==='eq') {
      closeness = goals===threshold ? 1 : Math.max(0, 1-Math.abs(goals-threshold)/thr)
    }
    scoreSum += closeness
  })
  if (pj===0) return {val:1.50, type:'hist'}
  var val = 1.00 + 2.00 * (1 - scoreSum/pj)
  return {val: clamp(val, 1.00, 3.00), type: 'hist'}
}
```

---

## getOdds — mercados 1–3 (campeon, goleador, primero)

```javascript
function getOdds(market, option) {
  const pool = S.oddsPool[market] || []
  const opts = mktOpts(market)
  const n = opts.length || 3
  const realFor = pool.filter(b => b.option===option).reduce(...)

  // Sin apuestas en esta opción → seed desde stats
  if (realFor === 0) {
    const hs = _histSeedOdds(market, option)
    return hs || {val: SEED_ODDS_CAP, type: 'seed'}
  }

  // Con apuestas → parimutuel con dampening ponderado por stats
  const realTotal = pool.reduce((s,b) => s+b.amount, 0)
  const weights = _getGoalWeights(opts)        // stats-weighted
  const vPerOption = weights?.[option] != null
    ? ODDS_DAMP * weights[option]
    : ODDS_DAMP / n
  const adjFor = realFor + vPerOption
  const adjTotal = realTotal + ODDS_DAMP
  return {val: Math.max(1.00, adjTotal*(1-S.rake)/adjFor), type: 'real'}
}
```

---

## _histGetOdds — mercado goles del jugador

```javascript
function _histGetOdds(optKey) {
  // optKey = "Bruno Pertile|gte|3"
  const pool = S.oddsPool.historico || []
  const poolFor = pool.filter(b => b.option===optKey).reduce(...)

  // Sin apuestas en esta opción → closeness seed
  if (poolFor === 0) {
    return _histCondSeedOdds(player, cond, threshold)
    // Rango: [1.00x, 3.00x], sin historial: 1.50x
  }

  // Con apuestas → parimutuel uniforme
  const vPerOpt = ODDS_DAMP / uniqueOpts.size
  const adjFor = poolFor + vPerOpt
  const adjTotal = total + ODDS_DAMP
  return {val: Math.max(1.00, adjTotal*(1-S.rake)/adjFor), type: 'real'}
}
```

---

## _getGoalWeights — pesos para dampening

```javascript
// Convierte avg goles históricos en pesos normalizados (suman 1)
// para distribuir el virtual dampening proporcionalmente (no uniforme)
function _getGoalWeights(opts) {
  var ps = _getPlayerStats()
  var scores = opts.map(p => {
    var s = ps[p]
    if (!s || s.pj < 3) return 0.7  // Poisson fallback
    // Suma ponderada desde distribución de goles (bucket 1..6)
    return totalGoals / s.pj
  })
  var total = scores.reduce((a,b) => a+b, 0)
  if (!total) return null
  return Object.fromEntries(opts.map((o,i) => [o, scores[i]/total]))
}
```

---

## Hot/Cold indicators

```javascript
function getPlayerTemp(playerName) {
  // Usa stats reales si hay datos suficientes
  var allAvgs = Object.values(ps).filter(s => s.pj>=3).map(computeAvg)
  var leagueAvg = mean(allAvgs)
  var playerAvg = computeAvg(ps[playerName])
  if (playerAvg > leagueAvg * 1.6) return 'hot'   // 🔥
  if (playerAvg < leagueAvg * 0.35) return 'cold'  // 🧊
  // Fallback: actividad de apuestas (si no hay stats)
}
```

---

## Payout calculation (siempre fijo al confirmar)

```javascript
// oddsAtTime se guarda al confirmar la apuesta — NUNCA se recalcula
payout = Math.floor(bet.amount * bet.oddsAtTime)

// Rake implícito: org gana la diferencia entre lo apostado y lo pagado
// Floor: 1.00x — el ganador nunca pierde plata apostando
```

---

## Equipos SÍ/NO markets

```javascript
function _calcSiNoOdds(siAmt, noAmt) {
  const virt = ODDS_DAMP / 2           // $2.500 virtual por lado
  const adjSi = siAmt + virt
  const adjNo = noAmt + virt
  const tot = adjSi + adjNo
  return {
    si: Math.max(1.00, tot*(1-S.rake)/adjSi),
    no: Math.max(1.00, tot*(1-S.rake)/adjNo)
  }
}
```

---

## Early Bird bonus

```javascript
// Cuando earlyBird.active=true, bets se marcan isEarlyBird=true
// Al pagar:
const ebBonus = isEarlyBird ? Math.floor(bet.amount * oddsAtTime * S.earlyBird.bonus) : 0
payout = Math.floor(bet.amount * oddsAtTime) + ebBonus
// El bonus sale del rake de la org
```

---

## Desafíos P2P rake

```javascript
// Solo sobre ganancias netas del lado ganador:
const gain = Math.floor(contraPool * (myStake / myTotal))
const net  = Math.floor(gain * (1 - S.rake))
payout     = myStake + net
orgBonus  += Math.floor(contraPool * S.rake)
```
