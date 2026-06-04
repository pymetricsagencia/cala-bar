# Filtro de meses en Resumen Anual — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Agregar chips Ene…Dic en la pestaña Resumen Anual para elegir qué meses se muestran en tablas y gráficos (la columna Total solo aparece con los 12 meses).

**Architecture:** No se recalcula la matriz. Se separa render de cálculo: `renderResumen()` construye las matrices y delega en `renderResumenView()`, que dibuja tablas+gráficos aplicando un set global `resumenMonths`. Los chips togglean ese set y llaman a `renderResumenView()`.

**Tech Stack:** HTML/CSS/JS vanilla en `index.html`, Chart.js 4. Sin test runner → verificación con `node --check` (sintaxis) + navegador (preview server local en `.claude/launch.json`, puerto 5599).

**Convención:** rama `feat/resumen-filtro-meses` (ya creada, con el spec commiteado). Commits chicos. NO pushear a main sin OK del usuario (push = deploy). Ubicar por nombre de función, no por línea.

**Verificar en navegador:** `mcp__Claude_Preview__preview_start` con name `cala-bar` → serverId; recargar con `preview_eval` `location.reload()`; inspeccionar con `preview_eval`.

---

## Task 1: Refactor — separar build de render (sin cambio de comportamiento)

**Files:** Modify `index.html` (función `renderResumen`)

- [ ] **Step 1: Reemplazar `renderResumen` por build + `renderResumenView`**

Buscar la función actual:
```js
function renderResumen(){
  try{
    const eco=buildResumenMatrix('economico');
    const fin=buildResumenMatrix('financiero');
    window._resumenMatrix={economico:eco,financiero:fin};
    document.getElementById('ra-eco-year').textContent=eco.year;
    document.getElementById('ra-fin-year').textContent=fin.year;
    renderResumenTable('ra-eco-body','economico',eco);
    renderResumenTable('ra-fin-body','financiero',fin);
    renderResumenChart('economico');
    renderResumenChart('financiero');
    [['economico','ra-eco-body'],['financiero','ra-fin-body']].forEach(([t,bid])=>{
      const body=document.getElementById(bid);
      resumenChartSel[t].forEach((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];const dot=body.querySelector('.ra-dot[data-key="'+CSS.escape(k)+'"]');if(dot){dot.classList.add('on');dot.style.background=c;dot.closest('tr').classList.add('charted');}});
    });
  }catch(e){console.error('renderResumen',e);}
}
```
Reemplazarla por:
```js
function renderResumen(){
  try{
    window._resumenMatrix={economico:buildResumenMatrix('economico'),financiero:buildResumenMatrix('financiero')};
    document.getElementById('ra-eco-year').textContent=window._resumenMatrix.economico.year;
    document.getElementById('ra-fin-year').textContent=window._resumenMatrix.financiero.year;
    renderResumenView();
  }catch(e){console.error('renderResumen',e);}
}
function renderResumenView(){
  if(!window._resumenMatrix)return;
  if(typeof refreshResumenChips==='function')refreshResumenChips();
  renderResumenTable('ra-eco-body','economico',window._resumenMatrix.economico);
  renderResumenTable('ra-fin-body','financiero',window._resumenMatrix.financiero);
  renderResumenChart('economico');
  renderResumenChart('financiero');
  [['economico','ra-eco-body'],['financiero','ra-fin-body']].forEach(([t,bid])=>{
    const body=document.getElementById(bid);
    resumenChartSel[t].forEach((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];const dot=body.querySelector('.ra-dot[data-key="'+CSS.escape(k)+'"]');if(dot){dot.classList.add('on');dot.style.background=c;dot.closest('tr').classList.add('charted');}});
  });
}
```
(El `if(typeof refreshResumenChips==='function')` evita romper hasta que esa función exista en Task 3.)

- [ ] **Step 2: Syntax check**

Run:
```bash
cd "C:/Users/lenri/cala-bar"
node -e "const fs=require('fs');const h=fs.readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync(require('os').tmpdir()+'/ra.js',m[1]);"
node --check "$(node -e "console.log(require('os').tmpdir())")/ra.js" && echo OK
```
Expected: `OK`

- [ ] **Step 3: Verificar en navegador (sin regresión)**

`preview_start` (name `cala-bar`) → recargar → en consola:
```js
(function(){document.querySelector('.stab[data-section="resumen"]').click();const b=document.getElementById('ra-eco-body');return {filas:b.querySelectorAll('tr').length, cols:b.querySelector('thead tr').children.length};})()
```
Expected: `filas` > 0 y `cols` = 14 (Cuenta + 12 meses + Total Año). Idéntico a antes del refactor.

- [ ] **Step 4: Commit**
```bash
git add index.html && git commit -m "refactor(resumen): separar renderResumen (build) de renderResumenView (render)"
```

---

## Task 2: Honrar `resumenMonths` en tabla y gráfico (default 12 = sin cambio visible)

**Files:** Modify `index.html` (estado nuevo; `renderResumenTable`; `renderResumenChart`)

- [ ] **Step 1: Agregar estado + helper**

Buscar la línea:
```js
let resumenChartMode={economico:'importe',financiero:'importe'};
```
Agregar debajo:
```js
let resumenMonths=new Set([0,1,2,3,4,5,6,7,8,9,10,11]);
function raVisibleMonths(){return [...resumenMonths].sort((a,b)=>a-b);}
```

- [ ] **Step 2: Reemplazar `renderResumenTable` para usar meses visibles + Total condicional**

Reemplazar la función `renderResumenTable` completa por:
```js
function renderResumenTable(containerId,type,matrix){
  const body=document.getElementById(containerId);
  const months=raVisibleMonths();
  const showTotal=resumenMonths.size===12;
  const colspan=months.length+(showTotal?1:0);
  let h='<table class="ra-table"><thead><tr><th class="ra-col-cuenta">Cuenta</th>';
  months.forEach(m=>h+='<th>'+RA_MESES[m]+'</th>');
  if(showTotal)h+='<th class="ra-col-total">Total Año</th>';
  h+='</tr></thead><tbody>';
  matrix.lines.forEach(line=>{
    if(line.kind==='hdr'){h+='<tr class="ra-row-hdr"><td class="ra-col-cuenta">'+line.label+'</td><td colspan="'+colspan+'"></td></tr>';return;}
    let rowClass='',nameCell='';
    if(line.kind==='account'){rowClass='ra-row-acc';const caret='<span class="ra-caret'+(line.hasSubs?'':' hidden')+'" onclick="event.stopPropagation();toggleResumenSub(this,\''+line.key+'\')">▶</span>';nameCell=caret+'<span class="ra-dot" data-key="'+line.key+'"></span>'+line.label;}
    else if(line.kind==='subaccount'){rowClass='ra-row-sub" data-parent="'+line.parentKey;nameCell='<span class="ra-dot" data-key="'+line.key+'"></span>'+line.label;}
    else if(line.kind==='section-total'){rowClass='ra-row-total';nameCell=line.label;}
    else if(line.kind==='result'){rowClass='ra-row-result';nameCell=line.label;}
    else if(line.kind==='saldo'){rowClass='ra-row-total';nameCell=line.label;}
    const clickAttr=line.selectable?(' onclick="toggleResumenChart(\''+type+'\',\''+line.key+'\')"'):'';
    h+='<tr class="'+rowClass+'"'+clickAttr+'><td class="ra-col-cuenta">'+nameCell+'</td>';
    months.forEach(m=>h+=raCell(line.imp[m],line.pct?line.pct[m]:null));
    if(showTotal)h+=raCell(line.impYear,line.pctY,'ra-col-total');
    h+='</tr>';
  });
  h+='</tbody></table>';
  body.innerHTML=h;
}
```
(Cambios vs original: `months`/`showTotal`/`colspan` dinámicos; el `<th>`/celdas de meses iteran `months`; el `<th>`/celda Total solo si `showTotal`; el `colspan` del hdr ya no es fijo "13".)

- [ ] **Step 3: Reemplazar `renderResumenChart` para filtrar labels+data a meses visibles**

Reemplazar la función `renderResumenChart` completa por:
```js
function renderResumenChart(type){
  const matrix=window._resumenMatrix?window._resumenMatrix[type]:null;if(!matrix)return;
  const canvasId=type==='economico'?'ra-eco-chart':'ra-fin-chart';
  const legendId=type==='economico'?'ra-eco-legend':'ra-fin-legend';
  const instKey=type==='economico'?'_resumenChartEco':'_resumenChartFin';
  const mode=resumenChartMode[type];
  const months=raVisibleMonths();
  const sel=resumenChartSel[type].filter(k=>matrix.seriesByKey[k]);
  resumenChartSel[type]=sel;
  const legend=document.getElementById(legendId);
  if(sel.length===0){legend.innerHTML='<span class="ra-legend-empty">Tocá una cuenta en la tabla para ver su tendencia</span>';}
  else{legend.innerHTML=sel.map((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];return '<span><span class="ra-dot on" style="background:'+c+'"></span>'+matrix.seriesByKey[k].label+'</span>';}).join('');}
  const datasets=sel.map((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];const s=matrix.seriesByKey[k];const arr=(mode==='pct'?s.pct:s.imp);return{label:s.label,data:months.map(m=>arr[m]),borderColor:c,backgroundColor:c,tension:.25,spanGaps:false,pointRadius:3,borderWidth:2.5};});
  const ctx=document.getElementById(canvasId).getContext('2d');
  if(window[instKey])window[instKey].destroy();
  window[instKey]=new Chart(ctx,{
    type:'line',
    data:{labels:months.map(m=>RA_MESES[m]),datasets},
    options:{responsive:true,maintainAspectRatio:false,layout:{padding:{top:18}},
      plugins:{legend:{display:false},tooltip:{callbacks:{label:c=>c.dataset.label+': '+(mode==='pct'?fmtPct(c.parsed.y):fmt(c.parsed.y))}}},
      scales:{y:{ticks:{color:'#7A90B8',callback:v=>mode==='pct'?v+'%':fmtShort(v)},grid:{color:'rgba(255,255,255,0.06)'}},x:{ticks:{color:'#7A90B8'},grid:{display:false}}}
    },
    plugins:[raLabelPlugin]
  });
  window[instKey].$raMode=mode;
}
```
(Cambios vs original: `const months=raVisibleMonths();`; `data:months.map(m=>arr[m])`; `labels:months.map(m=>RA_MESES[m])`.)

- [ ] **Step 4: Syntax check** (igual comando que Task 1 Step 2). Expected `OK`.

- [ ] **Step 5: Verificar en navegador (default = 12 meses, sin cambio)**

Recargar y en consola:
```js
(function(){document.querySelector('.stab[data-section="resumen"]').click();const b=document.getElementById('ra-eco-body');return {cols:b.querySelector('thead tr').children.length, ult:b.querySelector('thead tr').lastElementChild.textContent};})()
```
Expected: `cols` = 14, `ult` = "Total Año" (sin cambios porque `resumenMonths` tiene los 12).

Probar el filtro a mano (todavía sin chips):
```js
(function(){resumenMonths=new Set([2,3,4]);renderResumenView();const b=document.getElementById('ra-eco-body');const th=[...b.querySelectorAll('thead th')].map(x=>x.textContent);resumenMonths=new Set([0,1,2,3,4,5,6,7,8,9,10,11]);renderResumenView();return th;})()
```
Expected: `["Cuenta","Mar","Abr","May"]` (3 meses, **sin** "Total Año").

- [ ] **Step 6: Commit**
```bash
git add index.html && git commit -m "feat(resumen): render de tabla/gráfico honra resumenMonths (Total solo con 12)"
```

---

## Task 3: Chips Ene…Dic + atajo "Todos" (UI + handlers)

**Files:** Modify `index.html` (HTML de `#sec-resumen`; CSS `.ra-*`; handlers JS)

- [ ] **Step 1: Agregar la fila de chips en el HTML**

Buscar:
```html
<div class="section" id="sec-resumen">
  <div class="resumen-block">
```
Reemplazar por (inserta `#ra-month-filter` antes del primer `.resumen-block`):
```html
<div class="section" id="sec-resumen">
  <div class="ra-month-filter" id="ra-month-filter">
    <span class="ra-month-label">Meses</span>
    <button class="ra-chip active" data-m="0" onclick="toggleResumenMonth(0,this)">Ene</button>
    <button class="ra-chip active" data-m="1" onclick="toggleResumenMonth(1,this)">Feb</button>
    <button class="ra-chip active" data-m="2" onclick="toggleResumenMonth(2,this)">Mar</button>
    <button class="ra-chip active" data-m="3" onclick="toggleResumenMonth(3,this)">Abr</button>
    <button class="ra-chip active" data-m="4" onclick="toggleResumenMonth(4,this)">May</button>
    <button class="ra-chip active" data-m="5" onclick="toggleResumenMonth(5,this)">Jun</button>
    <button class="ra-chip active" data-m="6" onclick="toggleResumenMonth(6,this)">Jul</button>
    <button class="ra-chip active" data-m="7" onclick="toggleResumenMonth(7,this)">Ago</button>
    <button class="ra-chip active" data-m="8" onclick="toggleResumenMonth(8,this)">Sep</button>
    <button class="ra-chip active" data-m="9" onclick="toggleResumenMonth(9,this)">Oct</button>
    <button class="ra-chip active" data-m="10" onclick="toggleResumenMonth(10,this)">Nov</button>
    <button class="ra-chip active" data-m="11" onclick="toggleResumenMonth(11,this)">Dic</button>
    <button class="ra-chip-all active" onclick="setAllResumenMonths()">Todos</button>
  </div>
  <div class="resumen-block">
```

- [ ] **Step 2: Agregar el CSS de los chips**

Buscar `.ra-chart-wrap{position:relative;height:300px;padding:.3rem .2rem 0}` y agregar debajo:
```css
.ra-month-filter{display:flex;align-items:center;flex-wrap:wrap;gap:.35rem;margin:0 .2rem 1rem;padding-bottom:.8rem;border-bottom:1px solid var(--border)}
.ra-month-label{font-family:var(--font-mono);font-size:.6rem;text-transform:uppercase;letter-spacing:.1em;color:var(--muted);margin-right:.4rem}
.ra-chip,.ra-chip-all{padding:.25rem .6rem;border-radius:var(--radius-sm);border:1px solid var(--border);background:var(--bg3);color:var(--muted);font-family:var(--font-display);font-size:.72rem;font-weight:600;cursor:pointer;transition:all .15s}
.ra-chip:hover,.ra-chip-all:hover{border-color:var(--accent);color:var(--text)}
.ra-chip.active{background:rgba(13,217,184,.12);border-color:var(--accent);color:var(--accent)}
.ra-chip-all{margin-left:.5rem;border-style:dashed}
.ra-chip-all.active{background:var(--accent);border-style:solid;color:#062018}
```

- [ ] **Step 3: Agregar los handlers JS**

Buscar `function setResumenChartMode(type,mode,btn){` y agregar **antes** de esa línea:
```js
function refreshResumenChips(){
  document.querySelectorAll('#ra-month-filter .ra-chip').forEach(b=>{b.classList.toggle('active',resumenMonths.has(+b.dataset.m));});
  const allBtn=document.querySelector('#ra-month-filter .ra-chip-all');
  if(allBtn)allBtn.classList.toggle('active',resumenMonths.size===12);
}
function toggleResumenMonth(i,btn){
  if(resumenMonths.has(i)){if(resumenMonths.size===1)return;resumenMonths.delete(i);}
  else resumenMonths.add(i);
  refreshResumenChips();
  renderResumenView();
}
function setAllResumenMonths(){
  resumenMonths=new Set([0,1,2,3,4,5,6,7,8,9,10,11]);
  refreshResumenChips();
  renderResumenView();
}
```

- [ ] **Step 4: Syntax check** (igual comando). Expected `OK`.

- [ ] **Step 5: Verificar en navegador (funcional)**

Recargar, ir a Resumen Anual, y en consola:
```js
(function(){
  const f=document.getElementById('ra-month-filter');
  const chips=f.querySelectorAll('.ra-chip').length;
  // apagar Ene y Feb
  f.querySelector('.ra-chip[data-m="0"]').click();
  f.querySelector('.ra-chip[data-m="1"]').click();
  const b=document.getElementById('ra-eco-body');
  const ths=[...b.querySelectorAll('thead th')].map(x=>x.textContent);
  // reset
  f.querySelector('.ra-chip-all').click();
  const thsReset=[...b.querySelectorAll('thead th')].map(x=>x.textContent);
  return {chips, trasApagarEneFeb:ths, trasTodos_ultCol:thsReset[thsReset.length-1], trasTodos_n:thsReset.length};
})()
```
Expected: `chips`=12; `trasApagarEneFeb` empieza con "Cuenta","Mar",… y **no** incluye "Total Año"; tras "Todos": última col = "Total Año", `n`=14.

También probar manualmente: apagar todos menos uno (no deja apagar el último); el gráfico muestra solo los meses prendidos; los chips prendidos quedan en teal.

- [ ] **Step 6: Commit**
```bash
git add index.html && git commit -m "feat(resumen): chips Ene-Dic + atajo Todos para filtrar meses"
```

---

## Self-Review (autor del plan)

**Cobertura del spec:**
- §2.1 chips Ene…Dic + "Todos" → Task 3 ✓
- §2.2 default 12 → estado inicial `new Set([0..11])` + chips `active` (Task 2/3) ✓
- §2.3 selector compartido eco/fin → `resumenMonths` global usado por ambos `renderResumenTable`/`renderResumenChart` (Task 2) ✓
- §2.4 afecta tablas y gráficos → Task 2 ✓
- §2.5 Total solo con 12 → `showTotal=resumenMonths.size===12` (Task 2) ✓
- §2.6 % sin cambios → se siguen usando `line.pct[m]`/`s.pct` por mes ✓
- §2.7 ≥1 mes → guard en `toggleResumenMonth` (Task 3) ✓
- §3 enfoque (no recomputar) → `renderResumenView` (Task 1) ✓
- §5 casos borde → cubiertos en verificaciones de Task 2/3 ✓

**Placeholders:** ninguno (código completo en cada paso).

**Consistencia de nombres:** `resumenMonths` (Set), `raVisibleMonths()`, `renderResumenView()`, `refreshResumenChips()`, `toggleResumenMonth(i,btn)`, `setAllResumenMonths()`, `#ra-month-filter`, `.ra-chip`/`.ra-chip-all` — usados igual en HTML, CSS y JS en todas las tasks. `renderResumenView` se define en Task 1 y se invoca desde los handlers de Task 3 (orden de definición no importa: function declarations hoisted). ✓
