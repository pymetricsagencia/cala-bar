# Spec — Pestaña "Resumen Anual" (Cala Bar · Dashboard Financiero)

- **Fecha:** 2026-06-04
- **Proyecto:** `cala-bar` (archivo único `index.html`, deploy en GitHub Pages → https://pymetricsagencia.github.io/cala-bar/)
- **Estado:** Diseño aprobado, listo para plan de implementación.

## 1. Objetivo

Agregar una 5ª pestaña **"Resumen Anual"** que muestre los cuadros **Económico** y **Financiero** de forma **anualizada**: las mismas cuentas/subcuentas del estado de resultados actual, pero con **los 12 meses como columnas** (cada celda con importe + %), y **debajo de cada cuadro un gráfico de tendencia** con las cuentas que el usuario elija.

## 2. Contexto del código existente (anclajes)

Todo vive en `index.html` (un solo archivo). Funciones/zonas relevantes a reutilizar:

| Pieza | Ubicación aprox. | Rol |
|---|---|---|
| `<div class="section-tabs">` | L762 | Botones de pestaña (`data-section`) |
| Handler de pestañas | L1716 | Toggle `.active` en `.stab` + muestra `#sec-<name>` |
| `renderAll()` | L1339 | Llama a `renderEconomico/Financiero/CxPagar/Operativa/Temporal` |
| `buildStatementData(data,type)` | L986 | **Reutilizar**: arma secciones, totales y resultados de un período |
| `renderStatement(containerId,stmt,type)` | L987 | Render mensual (layout div, una columna) — referencia de estructura |
| `renderEconomico()` / `renderFinanciero()` | L1011 / L1012 | Patrón de "filtrar → buildStatementData → render" |
| `filterRows(rows,type)` | L977 | Filtra por sucursal + `getDateRange()` |
| `getDateRange()` | L976 | Deriva rango de `selectedYear`/`selectedMonth`/custom |
| `classifyAccount(cuenta,type)` | L975 | Clasifica cada cuenta en sección |
| `fmt` / `fmtPct` | L962 / L963 | `$1.234` (es-AR) / `12.34%` |
| `getCuenta/getSubCuenta/getAccNum/getAccLabel/getIng/getEgr` | L966-973 | Acceso a campos de fila |
| `getDateField(row,type)` + `parseDate` | L965 / L961 | Fecha por tipo (económico=comprobante, financiero=movimiento) |
| `toggleSub(id,rowEl)` | L1008 | Expandir/colapsar subcuentas |
| `renderTemporal()` / `buildOpArtSelect()` / `_opChart` | L1496 / L1423 | Patrón de selector + gráfico Chart.js |
| `_cfChartSaldo` (`new Chart`) | ~L2073 | Patrón de **gráfico de líneas** Chart.js |

Chart.js y PapaParse ya están cargados por CDN. Los datos salen de un Google Sheet publicado vía `rawData.{economico,financiero}`.

## 3. Decisiones de diseño (aprobadas)

1. **Pestaña nueva** "Resumen Anual", 5ª, después de *Operativa*.
2. Contenido: cuadro **Económico** arriba, cuadro **Financiero** debajo (cada uno con scroll horizontal propio).
3. **Formato de celda apilada**: importe arriba, **%** debajo en gris.
4. **Filas**: misma estructura del estado de resultados de cada tipo (secciones, totales, líneas de resultado) con **subcuentas desplegables** (el ▶ despliega).
5. **Columnas**: 12 meses (Ene→Dic) + columna **Total Año**; meses sin dato muestran **"—"**.
6. **Filtros**: aplican **año** y **sucursal**. El selector de **mes** y los controles de rango custom se **ocultan** mientras la pestaña está activa; al salir, reaparecen.
7. **Gráfico de tendencia** debajo de cada cuadro:
   - Líneas = cuentas/subcuentas seleccionadas; eje X = 12 meses.
   - **Selección por clic en la fila** de la tabla (el ▶ sigue siendo solo para desplegar). La fila seleccionada muestra un **puntito de color** que matchea su línea.
   - Botón **Importe ⇄ %** que alterna lo que grafican las líneas.
   - **Etiquetas resumidas** sobre cada punto: `K`/`M`/`MM` con decimal coma (ej. `8,4M`, `850K`). En modo %, la etiqueta es el porcentaje (`15%`).

### Supuestos a confirmar en revisión del spec
- **Qué filas son graficables:** además de cuentas y subcuentas, se permiten también las **líneas de resultado/total** (Resultado Operativo, Utilidad Bruta, Resultado Neto Financiero, Saldo Final), porque son las tendencias más valiosas. Solo los **encabezados de sección** no son clicables. *(Si preferís limitarlo a cuentas+subcuentas, se acota.)*
- **Año = "Todos":** si `selectedYear==='all'`, la pestaña usa el **año más reciente** presente en los datos (y lo indica en el título del cuadro).
- **Color de líneas:** paleta fija del dashboard (teal, dorado, azul, rojo, violeta, …) asignada por orden de selección; máx. recomendado ~6 líneas simultáneas (sin límite duro).

## 4. Enfoque técnico (Enfoque A — pivotear reutilizando `buildStatementData`)

No se reimplementa la clasificación ni los totales: se reutiliza `buildStatementData` corriéndolo **una vez por mes** y se "pivotea" el resultado. Esto garantiza que los números del Resumen Anual **cuadren exactamente** con la vista mensual.

### 4.1 Filtrado por mes
Nuevo helper que **ignora `selectedMonth`/custom** y filtra por sucursal + mes concreto del año elegido:

```
filterRowsByMonth(rows, type, year, monthIndex):
  return rows.filter(r =>
    (selectedSucursal==='all' || getSucursal(r)===selectedSucursal) &&
    fecha(r) dentro de [year-monthIndex-01 00:00, fin de mes 23:59])
```
(misma lógica de sucursal/fecha que `filterRows`, pero rango = un mes puntual).

### 4.2 Construcción de la matriz
`buildResumenMatrix(type) -> { year, lines, months }`:
- `year` = `selectedYear` o, si es `'all'`, el año más reciente con datos.
- Para `m` en 0..11: `stmt[m] = buildStatementData(filterRowsByMonth(rawData[type], type, year, m), type)`.
- `stmtYear = buildStatementData(<filas del año entero, sucursal aplicada>, type)` → columna **Total Año** (cálculo directo, no suma de redondeos).
- `lines` = lista ordenada que **espeja la estructura de `renderStatement`** para ese `type` (ver §5), donde cada línea numérica tiene:
  - `key` (identidad estable para gráfico/expansión): `acc:<num>` para cuenta, `sub:<num>|<subnombre>` para subcuenta, `res:<id>` para resultado/total.
  - `kind`: `section-hdr | account | subaccount | section-total | result | saldo`.
  - `label`, `selectable` (bool), `parentKey` (subcuentas).
  - `imp[12]` y `pct[12]`: importe y % por mes; **el % de cada mes usa la base de ESE mes** (`stmt[m].base`), igual que `renderStatement` (`pct = v => base>0 ? v/base*100 : 0`).
  - `impYear`, `pctYear` (con `stmtYear.base`).
  - El **signo/origen del importe** por sección replica `renderStatement`:
    - INGRESOS → `totalIng` de la cuenta/subcuenta;
    - COSTO / GASTOS / IMPUESTOS → `totalEgr`;
    - MOVIMIENTOS NO OPERATIVOS → neto `egr-ing`;
    - totales/resultados → campos del `stmt` (`totalVentas`, `totalCosto`, `utilidadBruta`, `totalGastos`, `totalImpuestos`, `resultadoOperativo`, `totalNoOperativo`, `resultadoNetoFinanciero`, `totalEgresosAll`, saldo).
- **Unión de cuentas/subcuentas**: una cuenta o subcuenta que aparece en algún mes genera su fila; en los meses donde no aparece, la celda es `0` y se muestra **"—"**.

## 5. Estructura de filas por tipo (espejo de `renderStatement`)

**Económico:**
1. `INGRESOS` (hdr) → cuentas → `Total Ingresos`
2. `COSTO DE VENTAS` (hdr) → cuentas → `Total Costo de Ventas` → `Utilidad Bruta`
3. `GASTOS OPERATIVOS` (hdr) → cuentas → `Total Gastos Operativos`
4. `IMPUESTOS Y TASAS` (hdr) → cuentas → `Total Impuestos`
5. `Resultado Operativo`

**Financiero:**
1. `Saldo Inicial del Período` (calculado con `getRowsBeforePeriod` adaptado al inicio del año/mes según corresponda — ver nota)
2. `INGRESOS` → cuentas → `Total Ingresos`
3. `COSTO DE MERCADERÍA` → cuentas → `Total Costo de Mercadería` → `Utilidad Bruta`
4. `GASTOS OPERATIVOS` → cuentas → `Total Gastos Operativos`
5. `IMPUESTOS Y TASAS` → cuentas → `Total Impuestos`
6. `Resultado Operativo`
7. `MOVIMIENTOS NO OPERATIVOS` (neto) → cuentas → `Total Movimientos No Operativos`
8. `Resultado Neto Financiero del Período`
9. `Total Egresos del Período`
10. `Saldo Final del Período`

> Nota saldo (financiero): en la vista anual, el **Saldo Inicial** de cada mes = saldo final del mes anterior; el de Enero = acumulado anterior al 1-Ene del año. Para el MVP se calcula el saldo **final por mes** (inicial del año + acumulado hasta fin de cada mes) reutilizando la lógica de `getIng-getEgr`. La fila de saldo NO es de las prioritarias para %; su % puede quedar vacío como hoy.

## 6. Render de la tabla (presentación nueva)

El render mensual actual usa divs de una sola columna; la matriz necesita una **tabla real**. Nueva función `renderResumenTable(containerId, type, matrix)`:
- `<table>` con **primera columna fija** (sticky `left:0`) para el nombre de cuenta, `thead` con Mes (Ene…Dic) + **Total Año** (columna con fondo teal suave), `tbody` con las `lines`.
- Cada celda numérica: `<b>{importe}</b>` arriba (vía `fmt`, pero para la tabla se usa el importe completo) + `<span>{%}</span>` debajo (vía `fmtPct`). Vacío → `—`.
- Encabezados de sección como fila full-width; totales/resultados con estilo destacado (reusar clases visuales tipo `stmt-total`/`stmt-result` o equivalentes nuevas `ra-*`).
- **Subcuentas**: filas ocultas por defecto; el ▶ de la cuenta las muestra/oculta (patrón `toggleSub`, adaptado a que cada subcuenta tiene 12 celdas + total).
- **Clic en la fila** (fuera del ▶) → `toggleResumenChart(type, key)`: agrega/saca la línea del gráfico y togglea el puntito de color en la fila.
- Negativos en rojo, resultados positivos en teal (igual criterio que hoy).

## 7. Gráfico de tendencia (Chart.js)

Estado por tipo:
```
resumenChartSel  = { economico: Set<key>, financiero: Set<key> }
resumenChartMode = { economico: 'importe'|'pct', financiero: 'importe'|'pct' }
_resumenChartEco, _resumenChartFin  // instancias Chart.js
```
- `renderResumenChart(type, matrix)`: arma datasets (uno por `key` seleccionada) usando `imp[]` o `pct[]` según `resumenChartMode[type]`; X = 12 meses; colores de paleta por orden de selección.
- **Etiquetas de datos resumidas**: plugin/`afterDatalabels` simple o `tooltip`+labels dibujadas; en modo importe usar nuevo helper `fmtShort(n)` (`850K`, `8,4M`, `1,2MM`, decimal coma); en modo % usar `fmtPct` redondeado.
- Botón **Importe ⇄ %**: cambia `resumenChartMode[type]` y re-renderiza solo ese gráfico.
- Default sin selección: gráfico vacío con leyenda "Tocá una cuenta en la tabla para ver su tendencia".
- **Sizing**: instanciar al renderizar; al activar la pestaña, llamar `.resize()`/re-render para evitar canvas mal dimensionado (Chart.js sobre canvas oculto). Ver patrón en `_opChart`/`renderTemporal`.

### `fmtShort(n)` (nuevo)
```
abs<1e3      -> $n
abs<1e6      -> $X,XK
abs<1e9      -> $X,XM
else         -> $X,XMM
(decimal con coma; signo conservado)
```

## 8. Integración (puntos de cambio en `index.html`)

1. **HTML — pestaña**: agregar `<button class="stab" data-section="resumen">Resumen Anual</button>` en `.section-tabs` (L762).
2. **HTML — sección**: nueva `<section id="sec-resumen" class="section">` con dos paneles (`#resumen-eco` con tabla `#ra-eco-body` + chart `#ra-eco-chart`, y `#resumen-fin` análogo).
3. **CSS**: clases `.ra-*` para la tabla matriz (sticky col, celda apilada, columna Total destacada), leyenda, toggle y puntitos de color. Reutilizar variables de `:root`.
4. **JS — `renderAll()` (L1339)**: agregar `renderResumen();`.
5. **JS — nuevas funciones**: `filterRowsByMonth`, `buildResumenMatrix`, `renderResumen` (orquesta eco+fin), `renderResumenTable`, `renderResumenChart`, `toggleResumenChart`, `toggleResumenSub`, `setResumenChartMode`, `fmtShort`.
6. **JS — handler de pestañas (L1716)**: al cambiar de pestaña, mostrar/ocultar el grupo del selector de **mes** y los controles de rango custom según `section==='resumen'`; al entrar a resumen, `resize` de los charts.
7. La pestaña usa **siempre `selectedYear`** (ignora `selectedMonth`/custom) para su cómputo.

## 9. Casos borde

- Mes sin movimientos / mes futuro del año en curso → columna en `—` (importe 0, % 0/—).
- `selectedSucursal` específica → todas las cuentas filtradas por esa sucursal.
- Cambiar año o sucursal → `renderAll` recalcula la pestaña.
- Cuenta presente en unos meses y no en otros → fila visible, `—` donde falte.
- Año = "Todos" → usar el año más reciente con datos + nota en el título.
- Sin cuentas seleccionadas → gráfico con mensaje placeholder.

## 10. Fuera de alcance (YAGNI)

- No se anualizan **Cuentas a Pagar** ni **Operativa** (solo Económico + Financiero).
- Sin exportar/imprimir, sin pivote tipo Excel (arrastrar campos), sin nuevos KPIs.
- Sin responsive especial para móvil más allá del scroll horizontal.

## 11. Rollout / verificación

- Implementar y **verificar localmente** (abrir `index.html`, comprobar que los totales del Resumen Anual cuadran con la vista mensual mes a mes, que el gráfico responde al clic y al toggle, y que el filtro de mes se oculta/restaura) **antes** de commitear.
- Al mergear, el push a `main` **actualiza el sitio live** (GitHub Pages). 
- La copia de escritorio `…\TABLERO NVO\index_V25.html` queda desactualizada respecto al repo; definir si se sincroniza o se deja el repo como única fuente de verdad (recomendado: repo como fuente única).
