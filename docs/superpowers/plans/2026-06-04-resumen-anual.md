# Pestaña "Resumen Anual" — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Agregar una 5ª pestaña "Resumen Anual" al dashboard `index.html` con los cuadros Económico y Financiero anualizados (12 meses como columnas, celda apilada importe+%) y, debajo de cada uno, un gráfico de tendencia por cuenta.

**Architecture:** Todo el dashboard es un único archivo `index.html` (vanilla JS + Chart.js + PapaParse, datos en vivo de Google Sheets). Se reutiliza `buildStatementData(data,type)` corriéndolo una vez por mes (Enfoque A) y se pivotea el resultado en una matriz `{lines, seriesByKey, baseByMonth}`. Una función de render nueva dibuja la tabla matriz; el gráfico usa Chart.js con un plugin inline para las etiquetas resumidas.

**Tech Stack:** HTML/CSS/JS vanilla en un solo archivo, Chart.js 4 (ya cargado por CDN), PapaParse (ya cargado). **Sin framework de tests** → verificación manual en navegador + consola del navegador.

**Convención de trabajo:** rama `feat/resumen-anual` (ya creada). Commits frecuentes en esa rama. NO se pushea a `main` hasta que el usuario lo apruebe (push = deploy a GitHub Pages). Para verificar: abrir `C:\Users\lenri\cala-bar\index.html` en el navegador (jala datos reales del Sheet).

**Nota sobre números de línea:** se desplazan a medida que se edita. Ubicar siempre por **nombre de función** (ej. "buscar `function renderAll`"), no por línea fija.

---

## Mapa de cambios en `index.html`

| Zona | Qué se agrega |
|---|---|
| `<div class="section-tabs">` | Botón `data-section="resumen"` |
| Después de la última `<section>` | `<section id="sec-resumen">` con 2 bloques (eco/fin): título, tabla, head de gráfico, canvas |
| Bloque `<style>` | Clases `.ra-*` (tabla sticky, celda apilada, leyenda, toggle, dot) |
| JS helpers (cerca de `fmt`) | `fmtShort`, `filterRowsByMonth`, `filterRowsByYear`, `resolveResumenYear` |
| JS nuevo | `buildResumenMatrix`, `renderResumen`, `renderResumenTable`, `renderResumenChart`, `toggleResumenChart`, `toggleResumenSub`, `setResumenChartMode` + estado global |
| `function renderAll` | Llamar `renderResumen()` |
| Handler de `.stab` | Ocultar/mostrar filtro de mes + custom en pestaña resumen; `resize` de charts |

---

## Task 1: Andamiaje — pestaña, sección vacía y ocultar filtro de mes

**Files:**
- Modify: `index.html` (tab button en `.section-tabs`; nueva `<section id="sec-resumen">`; handler de `.stab`)

- [ ] **Step 1: Agregar el botón de pestaña**

Buscar `<div class="section-tabs">` y agregar el 5º botón al final, antes de `</div>`:

```html
<button class="stab" data-section="resumen">Resumen Anual</button>
```

- [ ] **Step 2: Agregar la sección (vacía por ahora)**

Buscar la última `<section ...>...</section>` dentro del contenedor de secciones (la de `id="sec-operativa"`) y, **inmediatamente después de su `</section>`**, insertar:

```html
<section class="section" id="sec-resumen">
  <div class="resumen-block">
    <div class="resumen-title">Económico — Resumen Anual <span id="ra-eco-year" class="resumen-year"></span></div>
    <div class="ra-table-scroll"><div id="ra-eco-body"></div></div>
    <div class="ra-chart-head">
      <div id="ra-eco-legend" class="ra-legend"><span class="ra-legend-empty">Tocá una cuenta en la tabla para ver su tendencia</span></div>
      <div class="ra-toggle">
        <button class="ra-toggle-btn active" data-mode="importe" onclick="setResumenChartMode('economico','importe',this)">Importe</button>
        <button class="ra-toggle-btn" data-mode="pct" onclick="setResumenChartMode('economico','pct',this)">%</button>
      </div>
    </div>
    <div class="ra-chart-wrap"><canvas id="ra-eco-chart"></canvas></div>
  </div>

  <div class="resumen-block">
    <div class="resumen-title">Financiero — Resumen Anual <span id="ra-fin-year" class="resumen-year"></span></div>
    <div class="ra-table-scroll"><div id="ra-fin-body"></div></div>
    <div class="ra-chart-head">
      <div id="ra-fin-legend" class="ra-legend"><span class="ra-legend-empty">Tocá una cuenta en la tabla para ver su tendencia</span></div>
      <div class="ra-toggle">
        <button class="ra-toggle-btn active" data-mode="importe" onclick="setResumenChartMode('financiero','importe',this)">Importe</button>
        <button class="ra-toggle-btn" data-mode="pct" onclick="setResumenChartMode('financiero','pct',this)">%</button>
      </div>
    </div>
    <div class="ra-chart-wrap"><canvas id="ra-fin-chart"></canvas></div>
  </div>
</section>
```

- [ ] **Step 3: Ocultar el filtro de mes (y rango custom) en la pestaña resumen**

Buscar el handler de pestañas:
```js
document.querySelectorAll('.stab').forEach(btn=>{btn.addEventListener('click',()=>{document.querySelectorAll('.stab').forEach(b=>b.classList.remove('active'));btn.classList.add('active');document.querySelectorAll('.section').forEach(s=>s.classList.remove('active'));document.getElementById('sec-'+btn.dataset.section).classList.add('active');});});
```
Reemplazarlo por (agrega el manejo del filtro de mes + resize de charts):
```js
document.querySelectorAll('.stab').forEach(btn=>{btn.addEventListener('click',()=>{
  document.querySelectorAll('.stab').forEach(b=>b.classList.remove('active'));btn.classList.add('active');
  document.querySelectorAll('.section').forEach(s=>s.classList.remove('active'));
  const sec=btn.dataset.section;
  document.getElementById('sec-'+sec).classList.add('active');
  const isResumen=sec==='resumen';
  const mg=document.getElementById('filter-month').closest('.filter-group');
  if(mg)mg.style.display=isResumen?'none':'';
  const ct=document.getElementById('filter-custom-toggle');
  if(ct){const cg=ct.closest('.filter-group');if(cg)cg.style.display=isResumen?'none':'';}
  if(isResumen){if(window._resumenChartEco)window._resumenChartEco.resize();if(window._resumenChartFin)window._resumenChartFin.resize();}
});});
```

- [ ] **Step 4: Verificar en el navegador**

Abrir `index.html`. Esperado:
- Aparece la pestaña "Resumen Anual" como 5ª opción.
- Al clickearla: se ve la sección con dos títulos ("Económico — Resumen Anual" y "Financiero — Resumen Anual") y dos toggles Importe/%; las tablas y gráficos aún vacíos.
- El selector de **Mes** desaparece al estar en esa pestaña y reaparece al volver a otra.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(resumen): andamiaje de pestaña Resumen Anual + ocultar filtro de mes"
```

---

## Task 2: Helpers de datos

**Files:**
- Modify: `index.html` (agregar funciones cerca de `const fmt=` / `function filterRows`)

- [ ] **Step 1: Agregar `fmtShort` (abreviación K/M/MM)**

Buscar `const fmt=n=>{...};` (la línea de `fmt`) y agregar debajo:

```js
const fmtShort=n=>{const neg=n<0?'-':'';const a=Math.abs(n);let s;
  if(a<1000)s=Math.round(a).toLocaleString('es-AR');
  else if(a<1e6)s=(a/1e3).toFixed(a<1e4?1:0).replace('.',',')+'K';
  else if(a<1e9)s=(a/1e6).toFixed(1).replace('.',',')+'M';
  else s=(a/1e9).toFixed(1).replace('.',',')+'MM';
  return neg+'$'+s;};
```

- [ ] **Step 2: Agregar filtros por mes/año y resolución de año**

Buscar `function filterRows(rows,type){...}` y agregar debajo:

```js
function filterRowsByMonth(rows,type,year,monthIndex){
  const from=new Date(year,monthIndex,1,0,0,0);
  const to=new Date(year,monthIndex+1,0,23,59,59);
  return rows.filter(row=>{
    if(selectedSucursal!=='all'&&getSucursal(row)!==selectedSucursal)return false;
    const d=parseDate(getDateField(row,type));
    if(!d)return false; // sin fecha no se puede ubicar en un mes
    return d>=from&&d<=to;
  });
}
function filterRowsByYear(rows,type,year){
  const from=new Date(year,0,1,0,0,0);
  const to=new Date(year,11,31,23,59,59);
  return rows.filter(row=>{
    if(selectedSucursal!=='all'&&getSucursal(row)!==selectedSucursal)return false;
    const d=parseDate(getDateField(row,type));
    if(!d)return false;
    return d>=from&&d<=to;
  });
}
function resolveResumenYear(){
  if(selectedYear!=='all')return parseInt(selectedYear);
  let maxY=null;
  rawData.economico.forEach(r=>{const d=parseDate(getDateField(r,'economico'));if(d){const y=d.getFullYear();if(maxY===null||y>maxY)maxY=y;}});
  rawData.financiero.forEach(r=>{const d=parseDate(getDateField(r,'financiero'));if(d){const y=d.getFullYear();if(maxY===null||y>maxY)maxY=y;}});
  return maxY||_now.getFullYear();
}
```

- [ ] **Step 3: Verificar por consola del navegador**

Abrir `index.html`, esperar que cargue, abrir DevTools → Console y correr:
```js
fmtShort(8450000)   // "$8,4M"
fmtShort(850000)    // "$850K"
fmtShort(1200000000)// "$1,2MM"
fmtShort(-2960000)  // "-$3,0M"
resolveResumenYear()// un año (ej. 2026)
filterRowsByMonth(rawData.economico,'economico',resolveResumenYear(),0).length // nº de filas de Enero
```
Esperado: los valores entre comentarios y conteos coherentes (Enero ≤ total del año).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(resumen): helpers fmtShort + filtros por mes/año"
```

---

## Task 3: `buildResumenMatrix` (modelo de datos) + stub de `renderResumen`

**Files:**
- Modify: `index.html` (nueva función; llamada en `renderAll`)

- [ ] **Step 1: Agregar `buildResumenMatrix`**

Agregar (cerca de `buildStatementData`):

```js
function buildResumenMatrix(type){
  const year=resolveResumenYear();
  const months=[];
  for(let m=0;m<12;m++) months.push(buildStatementData(filterRowsByMonth(rawData[type],type,year,m),type));
  const stmtYear=buildStatementData(filterRowsByYear(rawData[type],type,year),type);
  const baseByMonth=months.map(s=>s.base);
  const baseYear=stmtYear.base;
  const lines=[]; const seriesByKey={};
  const pctOf=(v,base)=> v==null?null : (base>0? v/base*100 : 0);
  const accAmt=(acc,cat)=>cat==='ingresos'?acc.totalIng:cat==='noOperativo'?(acc.totalEgr-acc.totalIng):acc.totalEgr;
  const subAmt=(sd,cat)=>cat==='ingresos'?sd.ing:cat==='noOperativo'?(sd.egr-sd.ing):sd.egr;

  function register(key,label,imp,impYear){
    const pct=imp.map((v,m)=>pctOf(v,baseByMonth[m]));
    const pctY=pctOf(impYear,baseYear);
    seriesByKey[key]={label,imp,pct};
    return {imp,pct,impYear,pctY};
  }
  // Devuelve true si la sección tiene al menos una cuenta en algún mes
  function addSection(cat,hdrLabel){
    const accs={};
    for(let m=0;m<12;m++){
      months[m].sections[cat].forEach(acc=>{
        const k=acc.cuenta;
        if(!accs[k])accs[k]={cuenta:acc.cuenta,num:acc.num,label:getAccLabel(acc.cuenta),imp:Array(12).fill(null),subs:{},impYear:null,subsYear:{}};
        accs[k].imp[m]=accAmt(acc,cat);
        Object.entries(acc.subcuentas).forEach(([sn,sd])=>{
          if(!accs[k].subs[sn])accs[k].subs[sn]=Array(12).fill(null);
          accs[k].subs[sn][m]=subAmt(sd,cat);
        });
      });
    }
    stmtYear.sections[cat].forEach(acc=>{const a=accs[acc.cuenta];if(!a)return;a.impYear=accAmt(acc,cat);Object.entries(acc.subcuentas).forEach(([sn,sd])=>{a.subsYear[sn]=subAmt(sd,cat);});});
    const ordered=Object.values(accs).sort((x,y)=>x.num.localeCompare(y.num));
    if(ordered.length===0)return false;
    lines.push({kind:'hdr',label:hdrLabel,cat});
    ordered.forEach(a=>{
      const key='acc:'+type+':'+a.cuenta;
      const sumYear=a.impYear!=null?a.impYear:a.imp.reduce((t,v)=>t+(v||0),0);
      const s=register(key,a.label,a.imp,sumYear);
      const subNames=Object.keys(a.subs).sort((x,y)=>{
        const sx=a.subs[x].reduce((t,v)=>t+(v||0),0), sy=a.subs[y].reduce((t,v)=>t+(v||0),0);
        return Math.abs(sy)-Math.abs(sx);
      });
      lines.push({kind:'account',label:a.label,key,selectable:true,hasSubs:subNames.length>0,imp:s.imp,pct:s.pct,impYear:s.impYear,pctY:s.pctY});
      subNames.forEach(sn=>{
        const skey='sub:'+type+':'+a.cuenta+'||'+sn;
        const subYear=(a.subsYear&&a.subsYear[sn]!=null)?a.subsYear[sn]:a.subs[sn].reduce((t,v)=>t+(v||0),0);
        const ss=register(skey,sn,a.subs[sn],subYear);
        lines.push({kind:'subaccount',label:sn,key:skey,parentKey:key,selectable:true,imp:ss.imp,pct:ss.pct,impYear:ss.impYear,pctY:ss.pctY});
      });
    });
    return true;
  }
  function fieldLine(kind,label,field){
    const imp=months.map(s=>s[field]);
    const impYear=stmtYear[field];
    const key='res:'+type+':'+field;
    const s=register(key,label,imp,impYear);
    lines.push({kind,label,key,selectable:true,imp:s.imp,pct:s.pct,impYear:s.impYear,pctY:s.pctY});
  }

  addSection('ingresos','INGRESOS');
  fieldLine('section-total','Total Ingresos', type==='economico'?'totalVentas':'totalIngresosAll');
  const noun=type==='economico'?'Ventas':'Mercadería';
  if(addSection('costoVentas','COSTO DE '+(type==='economico'?'VENTAS':'MERCADERÍA'))){
    fieldLine('section-total','Total Costo de '+noun,'totalCosto');
    fieldLine('result','Utilidad Bruta','utilidadBruta');
  }
  if(addSection('gastosOperativos','GASTOS OPERATIVOS')){
    fieldLine('section-total','Total Gastos Operativos','totalGastos');
  }
  if(addSection('impuestos','IMPUESTOS Y TASAS')){
    fieldLine('section-total','Total Impuestos','totalImpuestos');
  }
  if(type==='economico'){
    fieldLine('result','Resultado Operativo','resultadoOperativo');
  }else{
    fieldLine('result','Resultado Operativo','resultadoOperativo');
    if(addSection('noOperativo','MOVIMIENTOS NO OPERATIVOS')){
      fieldLine('section-total','Total Movimientos No Operativos','totalNoOperativo');
    }
    fieldLine('result','Resultado Neto Financiero del Período','resultadoNetoFinanciero');
    fieldLine('section-total','Total Egresos del Período','totalEgresosAll');
    // Saldo final por mes (neto crudo acumulado, incluye todas las filas)
    let saldoIni=0;
    const yStart=new Date(year,0,1,0,0,0);
    rawData.financiero.forEach(r=>{if(selectedSucursal!=='all'&&getSucursal(r)!==selectedSucursal)return;const d=parseDate(getDateField(r,'financiero'));if(d&&d<yStart)saldoIni+=getIng(r)-getEgr(r);});
    let running=saldoIni; const saldoImp=[];
    for(let m=0;m<12;m++){let net=0;filterRowsByMonth(rawData.financiero,'financiero',year,m).forEach(r=>{net+=getIng(r)-getEgr(r);});running+=net;saldoImp.push(running);}
    const skey='res:'+type+':saldoFinal';
    seriesByKey[skey]={label:'Saldo Final del Período',imp:saldoImp,pct:saldoImp.map(()=>null)};
    lines.push({kind:'saldo',label:'Saldo Final del Período',key:skey,selectable:true,imp:saldoImp,pct:saldoImp.map(()=>null),impYear:running,pctY:null});
  }
  return {year,lines,seriesByKey,baseByMonth};
}
```

- [ ] **Step 2: Stub de `renderResumen` + llamada en `renderAll`**

Agregar:
```js
function renderResumen(){
  try{
    window._resumenMatrix={economico:buildResumenMatrix('economico'),financiero:buildResumenMatrix('financiero')};
  }catch(e){console.error('renderResumen',e);}
}
```
Buscar `function renderAll(){renderEconomico();renderFinanciero();renderCxPagar();renderOperativa();renderTemporal();}` y agregar `renderResumen();` antes del `}`:
```js
function renderAll(){renderEconomico();renderFinanciero();renderCxPagar();renderOperativa();renderTemporal();renderResumen();}
```

- [ ] **Step 3: Verificar por consola que los números cuadran**

Abrir `index.html`, esperar carga, Console:
```js
const M=window._resumenMatrix.economico;
M.year;                       // año resuelto
M.lines.filter(l=>l.kind==='result'); // incluye Resultado Operativo
// Comparar Total Año de "Resultado Operativo" del resumen vs la suma de los 12 meses:
const ro=M.lines.find(l=>l.label==='Resultado Operativo');
ro.impYear;                   // total anual
ro.imp.reduce((t,v)=>t+(v||0),0); // ≈ impYear (puede diferir por redondeo mínimo: NO debería, ambos son sumas exactas)
```
Esperado: `ro.impYear` y la suma de `ro.imp` coinciden; `M.lines` contiene encabezados (`kind:'hdr'`), cuentas, subcuentas y resultados.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(resumen): buildResumenMatrix (pivot por mes reutilizando buildStatementData)"
```

---

## Task 4: `renderResumenTable` — render de la matriz

**Files:**
- Modify: `index.html` (nueva función; llamarla desde `renderResumen`)
- Modify: `index.html` (bloque `<style>`: clases `.ra-*`)

- [ ] **Step 1: CSS de la tabla**

Buscar el cierre del bloque `<style>` (`</style>`) y agregar antes:

```css
#sec-resumen .resumen-block{margin-bottom:2rem}
.resumen-title{font-family:var(--font-display);font-weight:600;color:var(--text);font-size:1rem;margin:0 0 .6rem;padding:0 .2rem}
.resumen-year{color:var(--accent);font-family:var(--font-mono);font-size:.8rem;margin-left:.4rem}
.ra-table-scroll{overflow-x:auto;border:1px solid var(--border);border-radius:var(--radius-md);background:var(--bg2)}
.ra-table{border-collapse:collapse;width:100%;font-family:var(--font-mono);font-size:.72rem;color:var(--text);white-space:nowrap}
.ra-table th,.ra-table td{padding:.34rem .6rem;text-align:right;border-bottom:1px solid var(--border)}
.ra-table th{position:sticky;top:0;background:var(--bg4);color:var(--muted);font-weight:600;text-transform:uppercase;font-size:.6rem;letter-spacing:.05em;z-index:2}
.ra-table .ra-col-cuenta{text-align:left;position:sticky;left:0;background:var(--bg2);font-family:var(--font-display);font-weight:500;min-width:190px;z-index:1}
.ra-table th.ra-col-cuenta{background:var(--bg4);z-index:3}
.ra-table .ra-col-total{background:rgba(13,217,184,.06);border-left:1px solid var(--border)}
.ra-table th.ra-col-total{background:var(--bg4)}
.ra-cell b{display:block;font-weight:600}
.ra-cell .ra-pct{color:var(--muted);font-size:.64rem}
.ra-cell.neg b{color:var(--red)}
.ra-cell.pos b{color:var(--accent)}
.ra-row-hdr td{background:var(--bg3);color:var(--muted);text-align:left;font-family:var(--font-display);font-weight:700;font-size:.62rem;letter-spacing:.08em;text-transform:uppercase}
.ra-row-total .ra-col-cuenta,.ra-row-total td{font-weight:700}
.ra-row-result td{border-top:2px solid var(--accent);font-weight:700}
.ra-row-result .ra-col-cuenta{color:var(--accent)}
.ra-row-acc{cursor:pointer}
.ra-row-acc:hover .ra-col-cuenta,.ra-row-sub:hover .ra-col-cuenta{color:var(--accent)}
.ra-row-acc.charted,.ra-row-sub.charted{background:rgba(13,217,184,.05)}
.ra-row-sub{cursor:pointer;display:none}
.ra-row-sub.open{display:table-row}
.ra-row-sub .ra-col-cuenta{padding-left:2rem;color:var(--muted);font-weight:400}
.ra-caret{display:inline-block;width:1em;color:var(--muted);cursor:pointer;transition:transform .15s}
.ra-caret.open{transform:rotate(90deg)}
.ra-caret.hidden{visibility:hidden}
.ra-dot{display:inline-block;width:9px;height:9px;border-radius:50%;margin:0 .35rem 0 .15rem;vertical-align:middle;visibility:hidden}
.ra-dot.on{visibility:visible}
.ra-chart-head{display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:.5rem;margin:.7rem .2rem .4rem}
.ra-legend{display:flex;gap:.9rem;flex-wrap:wrap;font-family:var(--font-display);font-size:.72rem;color:var(--text)}
.ra-legend span{display:flex;align-items:center}
.ra-legend-empty{color:var(--muted)}
.ra-toggle{display:inline-flex;border:1px solid var(--border);border-radius:var(--radius-sm);overflow:hidden}
.ra-toggle-btn{padding:.25rem .8rem;font-family:var(--font-display);font-size:.68rem;color:var(--muted);background:transparent;border:none;cursor:pointer}
.ra-toggle-btn.active{background:var(--accent);color:#062018;font-weight:700}
.ra-chart-wrap{position:relative;height:300px;padding:.3rem .2rem 0}
```

- [ ] **Step 2: `renderResumenTable`**

Agregar:
```js
const RA_MESES=['Ene','Feb','Mar','Abr','May','Jun','Jul','Ago','Sep','Oct','Nov','Dic'];
function raCell(imp,pct,extraClass){
  if(imp==null)return '<td class="ra-cell '+(extraClass||'')+'">—</td>';
  const sign=imp<0?'neg':(imp>0?'pos':'');
  const pctTxt=(pct==null)?'':'<span class="ra-pct">'+fmtPct(pct)+'</span>';
  return '<td class="ra-cell '+sign+' '+(extraClass||'')+'"><b>'+fmt(imp)+'</b>'+pctTxt+'</td>';
}
function renderResumenTable(containerId,type,matrix){
  const body=document.getElementById(containerId);
  let h='<table class="ra-table"><thead><tr><th class="ra-col-cuenta">Cuenta</th>';
  RA_MESES.forEach(m=>h+='<th>'+m+'</th>');
  h+='<th class="ra-col-total">Total Año</th></tr></thead><tbody>';
  matrix.lines.forEach(line=>{
    if(line.kind==='hdr'){
      h+='<tr class="ra-row-hdr"><td class="ra-col-cuenta">'+line.label+'</td><td colspan="13"></td></tr>';
      return;
    }
    let rowClass='', nameCell='';
    if(line.kind==='account'){
      rowClass='ra-row-acc'; 
      const caret='<span class="ra-caret'+(line.hasSubs?'':' hidden')+'" onclick="event.stopPropagation();toggleResumenSub(this,\''+line.key+'\')">▶</span>';
      nameCell=caret+'<span class="ra-dot" data-key="'+line.key+'"></span>'+line.label;
    }else if(line.kind==='subaccount'){
      rowClass='ra-row-sub" data-parent="'+line.parentKey; // cierra el atributo class y abre data-parent
      nameCell='<span class="ra-dot" data-key="'+line.key+'"></span>'+line.label;
    }else if(line.kind==='section-total'){
      rowClass='ra-row-total'; nameCell=line.label;
    }else if(line.kind==='result'){
      rowClass='ra-row-result'; nameCell=line.label;
    }else if(line.kind==='saldo'){
      rowClass='ra-row-total'; nameCell=line.label;
    }
    const clickAttr=line.selectable?(' onclick="toggleResumenChart(\''+type+'\',\''+line.key+'\')"'):'';
    h+='<tr class="'+rowClass+'"'+clickAttr+'><td class="ra-col-cuenta">'+nameCell+'</td>';
    for(let m=0;m<12;m++)h+=raCell(line.imp[m],line.pct?line.pct[m]:null);
    h+=raCell(line.impYear,line.pctY,'ra-col-total');
    h+='</tr>';
  });
  h+='</tbody></table>';
  body.innerHTML=h;
}
```
> Nota sobre `data-parent`: el truco `rowClass='ra-row-sub" data-parent="...'` inyecta el atributo dentro del `class="..."`. Si preferís evitar el truco, reescribir esa rama para construir el `<tr>` con su propio template. Funciona porque el `<tr class="ra-row-sub" data-parent="KEY"` queda bien formado.

- [ ] **Step 3: Llamar la tabla desde `renderResumen`**

Actualizar `renderResumen`:
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
  }catch(e){console.error('renderResumen',e);}
}
```

- [ ] **Step 4: Stubs para que no rompa el onclick (se completan en Task 5 y 6)**

Agregar stubs temporales (se reemplazan luego):
```js
function toggleResumenSub(caretEl,key){}
function toggleResumenChart(type,key){}
```

- [ ] **Step 5: Verificar en navegador**

Abrir `index.html` → pestaña Resumen Anual. Esperado:
- Dos tablas (Económico, Financiero) con primera columna fija, 12 meses + Total Año.
- Celdas apiladas (importe arriba, % abajo), "—" en meses vacíos, negativos en rojo.
- El año aparece junto al título.
- Comparar a ojo: el "Total Ingresos / Total Año" del Económico debe coincidir con "Ventas" de la pestaña Económico cuando el filtro de año es el mismo y mes="Todos".

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(resumen): render de la tabla matriz (12 meses + Total Año, celda apilada)"
```

---

## Task 5: Subcuentas desplegables

**Files:**
- Modify: `index.html` (reemplazar el stub `toggleResumenSub`)

- [ ] **Step 1: Implementar `toggleResumenSub`**

Reemplazar el stub por:
```js
function toggleResumenSub(caretEl,parentKey){
  const tr=caretEl.closest('tr');
  let sib=tr.nextElementSibling;
  const open=caretEl.classList.toggle('open');
  while(sib&&sib.classList.contains('ra-row-sub')&&sib.dataset.parent===parentKey){
    sib.classList.toggle('open',open);
    sib=sib.nextElementSibling;
  }
}
```

- [ ] **Step 2: Verificar en navegador**

Abrir → Resumen Anual. Esperado:
- Las cuentas con subcuentas muestran ▶; al clickear el ▶ se despliegan las subcuentas (indentadas, en gris) en todas las columnas; el ▶ rota a ▼; volver a clickear las oculta.
- Clickear el ▶ **no** dispara la selección de gráfico (se detiene la propagación).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(resumen): subcuentas desplegables en la tabla anual"
```

---

## Task 6: Gráfico de tendencia — selección por clic en fila (modo importe)

**Files:**
- Modify: `index.html` (estado global; `toggleResumenChart`; `renderResumenChart`; estilos ya están)

- [ ] **Step 1: Estado global del gráfico**

Buscar la línea que declara `let selectedSucursal='all';...` y agregar después (misma zona de variables globales):
```js
const RA_PALETTE=['#0DD9B8','#f0b429','#2463EB','#f87171','#a78bfa','#34d399','#fb923c','#22d3ee'];
let resumenChartSel={economico:[],financiero:[]};
let resumenChartMode={economico:'importe',financiero:'importe'};
```

- [ ] **Step 2: Plugin inline de etiquetas + `renderResumenChart`**

Agregar:
```js
const raLabelPlugin={id:'raLabels',afterDatasetsDraw(chart){
  const ctx=chart.ctx;const mode=chart.$raMode||'importe';
  chart.data.datasets.forEach((ds,i)=>{
    const meta=chart.getDatasetMeta(i);if(meta.hidden)return;
    meta.data.forEach((pt,idx)=>{
      const v=ds.data[idx];if(v==null)return;
      ctx.save();ctx.fillStyle=ds.borderColor;ctx.font='600 9px '+'monospace';ctx.textAlign='center';
      ctx.fillText(mode==='pct'?fmtPct(v):fmtShort(v),pt.x,pt.y-7);
      ctx.restore();
    });
  });
}};
function renderResumenChart(type){
  const matrix=window._resumenMatrix[type];
  const canvasId=type==='economico'?'ra-eco-chart':'ra-fin-chart';
  const legendId=type==='economico'?'ra-eco-legend':'ra-fin-legend';
  const instKey=type==='economico'?'_resumenChartEco':'_resumenChartFin';
  const mode=resumenChartMode[type];
  const sel=resumenChartSel[type].filter(k=>matrix.seriesByKey[k]); // descarta keys ya inexistentes
  resumenChartSel[type]=sel;
  // leyenda
  const legend=document.getElementById(legendId);
  if(sel.length===0){legend.innerHTML='<span class="ra-legend-empty">Tocá una cuenta en la tabla para ver su tendencia</span>';}
  else{legend.innerHTML=sel.map((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];return '<span><span class="ra-dot on" style="background:'+c+'"></span>'+matrix.seriesByKey[k].label+'</span>';}).join('');}
  // datasets
  const datasets=sel.map((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];const s=matrix.seriesByKey[k];return{label:s.label,data:(mode==='pct'?s.pct:s.imp),borderColor:c,backgroundColor:c,tension:.25,spanGaps:false,pointRadius:3,borderWidth:2.5};});
  const ctx=document.getElementById(canvasId).getContext('2d');
  if(window[instKey])window[instKey].destroy();
  window[instKey]=new Chart(ctx,{
    type:'line',
    data:{labels:RA_MESES,datasets},
    options:{
      responsive:true,maintainAspectRatio:false,
      layout:{padding:{top:18}},
      plugins:{legend:{display:false},tooltip:{callbacks:{label:c=>c.dataset.label+': '+(mode==='pct'?fmtPct(c.parsed.y):fmt(c.parsed.y))}}},
      scales:{
        y:{ticks:{color:'#7A90B8',callback:v=>mode==='pct'?v+'%':fmtShort(v)},grid:{color:'rgba(255,255,255,0.06)'}},
        x:{ticks:{color:'#7A90B8'},grid:{display:false}}
      }
    },
    plugins:[raLabelPlugin]
  });
  window[instKey].$raMode=mode;
}
```

- [ ] **Step 3: `toggleResumenChart` (clic en fila)**

Reemplazar el stub por:
```js
function toggleResumenChart(type,key){
  const arr=resumenChartSel[type];
  const idx=arr.indexOf(key);
  if(idx>=0)arr.splice(idx,1); else arr.push(key);
  // actualizar puntito + clase en las filas afectadas
  const bodyId=type==='economico'?'ra-eco-body':'ra-fin-body';
  const body=document.getElementById(bodyId);
  // limpiar
  body.querySelectorAll('.ra-dot').forEach(d=>{d.classList.remove('on');d.style.background='';});
  body.querySelectorAll('.charted').forEach(r=>r.classList.remove('charted'));
  arr.forEach((k,i)=>{
    const c=RA_PALETTE[i%RA_PALETTE.length];
    const dot=body.querySelector('.ra-dot[data-key="'+CSS.escape(k)+'"]');
    if(dot){dot.classList.add('on');dot.style.background=c;dot.closest('tr').classList.add('charted');}
  });
  renderResumenChart(type);
}
```

- [ ] **Step 4: Render inicial de los gráficos en `renderResumen`**

Agregar al final del `try` de `renderResumen` (después de las dos `renderResumenTable`):
```js
    renderResumenChart('economico');
    renderResumenChart('financiero');
    // re-aplicar dots de selección existente tras re-render de tablas
    [['economico','ra-eco-body'],['financiero','ra-fin-body']].forEach(([t,bid])=>{
      const body=document.getElementById(bid);
      resumenChartSel[t].forEach((k,i)=>{const c=RA_PALETTE[i%RA_PALETTE.length];const dot=body.querySelector('.ra-dot[data-key="'+CSS.escape(k)+'"]');if(dot){dot.classList.add('on');dot.style.background=c;dot.closest('tr').classList.add('charted');}});
    });
```

- [ ] **Step 5: Verificar en navegador**

Abrir → Resumen Anual. Esperado:
- Clickear una fila de cuenta (no el ▶) agrega su línea al gráfico de ese cuadro, con un puntito de color en la fila y entrada en la leyenda.
- Clickear de nuevo la quita. Se pueden agregar varias (colores distintos).
- Las subcuentas también se pueden graficar.
- Los puntos muestran etiquetas resumidas (ej. `8,4M`).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(resumen): gráfico de tendencia con selección por clic en fila + etiquetas resumidas"
```

---

## Task 7: Toggle Importe ⇄ % y verificación cruzada

**Files:**
- Modify: `index.html` (`setResumenChartMode`)

- [ ] **Step 1: Implementar `setResumenChartMode`**

Agregar:
```js
function setResumenChartMode(type,mode,btn){
  resumenChartMode[type]=mode;
  const head=btn.closest('.ra-chart-head');
  head.querySelectorAll('.ra-toggle-btn').forEach(b=>b.classList.toggle('active',b===btn));
  renderResumenChart(type);
}
```

- [ ] **Step 2: Verificar en navegador**

Abrir → Resumen Anual, seleccionar 2 cuentas. Esperado:
- Botón **Importe** activo por defecto: las líneas y etiquetas muestran montos (`8,4M`), eje Y en `fmtShort`.
- Click en **%**: las líneas y etiquetas pasan a porcentaje (`15%`), eje Y en `%`. El botón % queda activo.
- Cada cuadro (eco/fin) tiene su toggle independiente.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(resumen): toggle Importe/% por cuadro"
```

---

## Task 8: Casos borde + verificación final

**Files:**
- Modify: `index.html` (solo si algún caso borde falla)

- [ ] **Step 1: Probar casos borde en navegador**

Con `index.html` abierto, pestaña Resumen Anual, probar y anotar:
1. **Cambiar el filtro de Año** → ambas tablas y gráficos se recalculan; el año del título cambia.
2. **Año = "Todos"** → usa el año más reciente con datos (lo muestra en el título). Sin error en consola.
3. **Cambiar Sucursal** → recalcula filtrando por esa sucursal.
4. **Mes futuro / sin datos** (año en curso) → columnas posteriores al último mes con datos en "—".
5. **Subcuenta seleccionada + cambio de año** → el dot se re-aplica solo si la subcuenta sigue existiendo (la selección de keys inexistentes se descarta sin romper).
6. **Volver a otra pestaña y regresar** → el gráfico se ve bien dimensionado (no chico/cortado).

- [ ] **Step 2: Verificación de consistencia (la importante)**

Para el año seleccionado, comparar el **Total Año** de varias filas del Resumen Económico contra la pestaña **Económico** con ese año y Mes="Todos":
- "Total Ingresos" (Total Año) == "Ventas" KPI.
- "Resultado Operativo" (Total Año) == "Resultado Operativo" KPI.
Deben coincidir exactamente (mismo `buildStatementData`).

- [ ] **Step 3: Si todo OK, commit final**

```bash
git add index.html
git commit -m "test(resumen): verificación de casos borde y consistencia (sin cambios o ajustes menores)"
```

- [ ] **Step 4: Resumen para el usuario (no commitear)**

Reportar: qué se verificó, capturas si corresponde, y recordar que el **deploy** (push a `main`) queda pendiente de su aprobación explícita.

---

## Self-Review (completado por el autor del plan)

**Cobertura del spec:**
- §3.1 pestaña nueva → Task 1 ✓
- §3.2 eco arriba / fin abajo → Task 1 (markup) ✓
- §3.3 celda apilada → Task 4 (`raCell`) ✓
- §3.4 subcuentas desplegables → Task 5 ✓
- §3.5 12 meses + Total Año + "—" → Task 3/4 ✓
- §3.6 año+sucursal aplican, mes oculto → Task 1 (handler) + Task 2 (filtros) ✓
- §3.7 gráfico clic-en-fila / toggle / etiquetas resumidas → Tasks 6 y 7 ✓
- §4 Enfoque A (buildStatementData por mes) → Task 3 ✓
- §5 estructura de filas por tipo → Task 3 (`addSection`/`fieldLine`) ✓
- §6 render tabla → Task 4 ✓
- §7 gráfico + `fmtShort` → Tasks 2,6,7 ✓
- §9 casos borde → Task 8 ✓
- Supuestos (líneas de resultado graficables, año=all→último, paleta) → implementados en Task 3/6 ✓

**Placeholders:** ninguno (todos los pasos con código real). El truco de `data-parent` está documentado con alternativa.

**Consistencia de tipos/nombres:** `buildResumenMatrix` devuelve `{year,lines,seriesByKey,baseByMonth}`; `lines[].key` usado consistentemente en `toggleResumenChart`/`renderResumenChart`/dots; `window._resumenChartEco/_resumenChartFin` y `resumenChartSel/Mode` con claves `economico|financiero` en todo el plan. `RA_MESES` definido en Task 4 y reutilizado en Task 6. ✓
