# UI Components y Design System

## Design tokens (CSS variables)

```css
/* Fondos — de más oscuro a más claro */
--bg:     #0e0f13   /* fondo base de la app */
--bg2:    #161820   /* cards, nav */
--bg3:    #1e2028   /* inputs, chips, elementos internos */
--bg4:    #252832   /* hover states, bordes activos */
--border: #2a2d38   /* todos los bordes */

/* Colores de acento */
--accent: #c8f135   /* lima — acción primaria, odds reales */
--a2:     #3dffa0   /* verde menta — retornos positivos, confirmado */
--a3:     #ff6b35   /* naranja — lado "NO" de props, warning */
--a4:     #ffcd38   /* amarillo — organizador, seed odds, early bird base */
--purple: #a855f7   /* violeta — Early Bird */
--blue:   #4a9eff   /* azul — alias MP, info */
--red:    #ff4757   /* rojo — rechazo, pérdida, peligro */
--green:  #2ed573   /* verde — ganancia, pagado, confirmar */

/* Texto */
--text:  #f0f2f7    /* primario */
--text2: #8b90a0    /* secundario, labels */
--text3: #555870    /* terciario, placeholders, disclaimers */

/* Radio */
--r:  12px   /* cards */
--rs: 7px    /* inputs, buttons, chips */

/* Tipografía */
--font: 'DM Sans', system-ui, sans-serif
--mono: 'DM Mono', monospace   /* odds, montos, refs */
/* Bebas Neue — solo títulos de sección y pantalla */
```

---

## Componentes base

### Card
```html
<div class="card">                          <!-- borde --border -->
  <div class="ch">                          <!-- card header -->
    <span class="ci">🏆</span>             <!-- icon -->
    <span class="ct">Título</span>         <!-- title -->
    <span class="cm">Meta</span>           <!-- meta right -->
  </div>
  <div class="cb">                          <!-- card body -->
    <!-- contenido -->
  </div>
</div>

<!-- Variante org (borde amarillo) -->
<div class="card org-c">...</div>
```

### Botones
```html
.btn .bp      <!-- primario: lima sobre negro -->
.btn .bs      <!-- secundario: bg3 con borde -->
.btn .bo      <!-- organizador: amarillo semitransparente -->
.btn .bd      <!-- danger: rojo -->
.btn .bk      <!-- ok/confirmar: verde -->
.btn .bbl     <!-- info/blue: azul -->
.btn .bpu     <!-- purple: early bird -->
.btn .bf      <!-- full width -->
.btn .bsm     <!-- small: padding reducido, font 11px -->
```

### Info boxes
```html
<div class="ib">         <!-- neutro -->
<div class="ib iw">      <!-- warning: amarillo -->
<div class="ib iok">     <!-- success: verde -->
<div class="ib ibl">     <!-- info: azul -->
<div class="ib ipu">     <!-- purple: early bird -->
<div class="ib ie">      <!-- error: rojo -->

<!-- Siempre incluir icono -->
<div class="ib iw">
  <span class="ibi">⚠️</span>
  <span>Texto del mensaje</span>
</div>
```

### Inputs
```html
<!-- Input texto / número -->
<div class="ig">
  <label class="il">Label</label>
  <input class="ifd" type="text" placeholder="..."/>
</div>

<!-- Select -->
<select class="sfd">
  <option value="">Placeholder</option>
</select>

<!-- Input de monto (con prefijo $) -->
<div class="aw">
  <span class="ap">$</span>
  <input class="ai" type="number" placeholder="0"/>
</div>
```

### Modal (bottom sheet)
```html
<div class="mo" id="modal-xxx">
  <div class="modal">
    <div class="mh"></div>          <!-- drag handle -->
    <div class="mt">Título</div>    <!-- Bebas Neue -->
    <!-- contenido -->
  </div>
</div>

<!-- Abrir/cerrar -->
openModal('modal-xxx')
closeModal('modal-xxx')
```

### Toast
```javascript
showToast('Mensaje', 'tok')   // verde
showToast('Mensaje', 'terr')  // rojo
showToast('Mensaje', 'tinf')  // azul
```

### Screen title
```html
<div class="stitle">Pantalla <span>ACENTO</span></div>
<div class="stitle org">Pantalla <span>ORG</span></div>
<!-- org → acento amarillo en lugar de lima -->
```

---

## Componentes de apuestas

### Stats status bar (pantalla Apostar)
```html
<!-- Aparece debajo del sync-status si ph9 tiene datos -->
<div id="stats-status" style="display:flex;align-items:center;justify-content:space-between;">
  <span style="font-size:10px;color:var(--text3);">
    📊 <strong style="color:var(--green);">22 jornadas</strong> cargadas
    · prom. liga <strong style="color:var(--text2);">0.84</strong> goles/j
  </span>
  <button onclick="_refreshStats()"
    style="border:1px solid var(--border);border-radius:6px;padding:3px 8px;font-size:10px;color:var(--text3);">
    🔄 Actualizar
  </button>
</div>
<!-- Se oculta (display:none) si ph9 está vacío -->
```

### Market option (fila de mercado)
```html
<!-- Opción normal -->
<div class="mopt [sel]" onclick="selectOpt('market','option')">
  <div class="oradio"><div class="odot"></div></div>
  <span class="oname">
    Nombre opción
    <!-- badges opcionales: -->
    <span style="font-size:9px;color:#ff4500;">🔥</span>  <!-- hot: avg > leagueAvg×1.6 -->
    <span style="font-size:9px;color:#4a9eff;">🧊</span>  <!-- cold: avg < leagueAvg×0.35 -->
    <span style="font-size:10px;color:var(--a2);">✓</span> <!-- ya apostó este usuario -->
  </span>
  <div class="odds-val [seed|stats|no-bets]">
    2.70x
    <span class="odds-tag tag-seed">BASE</span>   <!-- sin datos históricos -->
    <span class="odds-tag tag-stats">HIST</span>  <!-- desde historial ph9 (verde) -->
    <span class="odds-tag tag-eb">+10%EB</span>   <!-- Early Bird activo -->
  </div>
</div>

<!-- Jugador que no juega esta jornada (.mopt-nj) -->
<div class="mopt mopt-nj" onclick="void(0)">
  <div class="oradio" style="opacity:.3;"></div>
  <span class="oname" style="color:var(--text3);">
    Bruno Pertile
    <span style="font-size:10px;color:var(--text3);font-style:italic;"> · no juega</span>
  </span>
  <span style="font-size:11px;color:var(--text3);">—</span>
</div>
```

**Tipos de odds-val:**
```css
.odds-val           /* real: color var(--a4) amarillo */
.odds-val.seed      /* BASE: color var(--text3) gris */
.odds-val.stats     /* HIST: color var(--green) verde */
.odds-val.no-bets   /* sin datos: color var(--text3) */

.odds-tag           /* base para todos los badges */
.tag-seed           /* fondo gris oscuro, texto gris */
.tag-stats          /* fondo verde translúcido, texto verde */
.tag-eb             /* fondo violeta translúcido, texto violeta */
.tag-hot            /* fondo naranja, texto #ff4500 */
.tag-cold           /* fondo azul, texto var(--blue) */
```

### Odds preview block
```html
<div class="odds-preview [vis]" id="preview-cam">
  <div class="op-row">
    <span class="op-label">Odd actual</span>
    <span class="op-val [seed]">2.70x</span>
  </div>
  <!-- si EB activo: -->
  <div class="op-row">
    <span class="op-label">+ Bonus Early Bird</span>
    <span style="color:var(--purple)">+10%</span>
  </div>
  <div class="op-row" style="border-top...">
    <span class="op-label">Retorno estimado si ganás</span>
    <span class="op-est">$2.700</span>
  </div>
  <div class="op-warn">⚠️ Orientativo — las odds cambian...</div>
</div>
```

### Cart chip
```html
<div class="cart-chip">
  <span>🏆 Peronistas</span>
  <span class="cc-odds">2.70x<span class="cc-eb"> EB</span></span>
  <span style="color:var(--text3);font-family:var(--mono)">$1.000</span>
  <button class="cc-rem" onclick="removeFromCart('id')">✕</button>
</div>
```

### Bet row (mis apuestas)
```html
<div class="brow">
  <div class="bdot dc"></div>   <!-- dc=campeon, dg=goleador, dp=primero, dpr=prop -->
  <div class="binfo">
    <div class="bmkt">CAMPEÓN</div>
    <div class="bsel">Los Peronistas</div>
    <div class="bodds">Al apostar: 2.70x<span class="eb-mark"> · ⚡EB</span></div>
  </div>
  <div>
    <div class="bamt">$1.000</div>
    <div style="color:var(--a2)">→ $2.430</div>
  </div>
</div>
```

### Payment chip (status)
```html
<span class="pchip pp">Pendiente de pago</span>   <!-- amarillo -->
<span class="pchip pc">Confirmada</span>           <!-- verde claro -->
<span class="pchip ppd">Pagada</span>              <!-- verde -->
<span class="pchip pr">Rechazada</span>            <!-- rojo -->
```

### Cobro row (org view)
```html
<div class="cobro">
  <div class="cobro-top">
    <div class="cav">BP</div>                        <!-- iniciales -->
    <div style="flex:1">
      <div class="cname">Bruno Pertile</div>
      <div class="cref">PCH-20250328-1001</div>      <!-- batch ref -->
    </div>
    <div class="camt">$2.000</div>
  </div>
  <div class="csub">🏆 Peronistas $1.000 · ⚽ Rendo $500 · ⚡ Mazzi $500</div>
  <div class="cact">
    <button class="btn bk bsm bf" onclick="confirmBatch(...)">✓ Confirmar</button>
    <button class="btn bd bsm" onclick="rejectBatch(...)">✕</button>
  </div>
</div>
```

### Payout row
```html
<div class="prow">
  <div class="pav [win]">BP</div>           <!-- .win = borde lima -->
  <div class="pname">
    <div class="ppn">Bruno Pertile</div>
    <div class="ppal [none]">bruno.pertile</div>   <!-- .none si sin alias -->
  </div>
  <div>
    <div class="pamt [pos|neg|zer]">+$850</div>   <!-- pos=verde, neg=rojo, zer=gris -->
    <div style="color:var(--text3)">cobra $1.850</div>
  </div>
</div>
```

---

## MP Block (Mercado Pago)

```html
<div class="mp-block">
  <div class="mp-head">
    <span>💳</span>
    <span class="mp-ht">Transferí a Mercado Pago</span>
  </div>
  <div class="mp-body">
    <div class="alias-big">organizador.pachanga</div>    <!-- alias grande, azul -->
    <div style="height:1px;background:var(--border)"></div>
    <div class="mp-row">
      <span class="mp-l">Nombre</span>
      <span class="mp-v">Toti Manza</span>
    </div>
    <div class="mp-row">
      <span class="mp-l">Monto total</span>
      <span class="mp-v hi">$2.000</span>     <!-- .hi = lima, tamaño mayor -->
    </div>
    <div class="mp-row">
      <span class="mp-l">Referencia</span>
      <span class="mp-v ref">PCH-20250328-1001</span>  <!-- .ref = amarillo, pequeño -->
    </div>
  </div>
</div>
```

---

## Cart bar (barra fija inferior)

```html
<div class="cart-bar [hidden]" id="cart-bar">
  <div class="cart-preview" id="cart-preview">
    <!-- cart-chips generados por renderCart() -->
  </div>
  <div class="cart-inner">
    <div class="cart-info">
      <div class="cart-lbl">Total a transferir</div>
      <div class="cart-total">$3.000</div>
      <div class="cart-count">3 apuestas</div>
    </div>
    <button class="cart-go" onclick="openPayModal()">Pagar todo →</button>
  </div>
</div>
```

`.hidden` usa `transform: translateY(100%)` para animar la entrada/salida.

---

## Early Bird risk indicator

```html
<div class="eb-risk-bar">
  <div id="eb-risk-fill" class="eb-risk-fill [eb-risk-safe|eb-risk-warn|eb-risk-danger]"
       style="width:45%"></div>
</div>
```

Clases de color:
- `.eb-risk-safe` → verde → bonus < 50% del break-even
- `.eb-risk-warn` → amarillo → bonus entre 50% y 100% del break-even
- `.eb-risk-danger` → rojo → bonus > break-even

---

## PIN keypad

```html
<div class="pin-dots">
  <div class="pin-dot [f]" id="pd0"></div>  <!-- .f = filled/amarillo -->
  <div class="pin-dot" id="pd1"></div>
  <div class="pin-dot" id="pd2"></div>
  <div class="pin-dot" id="pd3"></div>
</div>
<div class="pin-kp">
  <!-- 12 botones: 1-9, del, 0, cerrar -->
</div>
```

La animación de error usa `@keyframes shake` aplicado al `.pin-dots` container.

---

## Fonts CDN

```
DM Sans    — texto base (300,400,500,600,700)
DM Mono    — monospace para odds, montos, refs (400,500)
Bebas Neue — display, títulos de sección
```

```
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:opsz,wght@9..40,300;9..40,400;9..40,500;9..40,600;9..40,700&family=DM+Mono:wght@400;500&family=Bebas+Neue&display=swap');
```
