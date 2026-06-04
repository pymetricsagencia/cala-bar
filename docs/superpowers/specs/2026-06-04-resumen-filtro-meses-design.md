# Spec — Filtro de meses en "Resumen Anual" (Cala Bar)

- **Fecha:** 2026-06-04
- **Proyecto:** `cala-bar` (`index.html` único, deploy GitHub Pages)
- **Estado:** Diseño aprobado, listo para plan.
- **Base:** se construye sobre la pestaña "Resumen Anual" (spec `2026-06-04-resumen-anual-design.md`).

## 1. Objetivo

Permitir elegir, dentro de la pestaña **Resumen Anual**, qué meses se muestran (subconjunto cualquiera de Ene…Dic), achicando tablas y gráficos a esos meses.

## 2. Decisiones (aprobadas)

1. **Control:** fila de **12 chips** (Ene…Dic) arriba de los dos cuadros, dentro de `#sec-resumen`. Click = prende/apaga. Atajo **"Todos"** para volver a los 12.
2. **Default:** los 12 meses prendidos (= comportamiento actual).
3. **Selector único compartido:** el mismo set de meses aplica a ambos cuadros (Económico y Financiero).
4. **Afecta tablas y gráficos:** las tablas muestran solo las columnas de los meses prendidos; los gráficos muestran solo esos meses en el eje X.
5. **Columna Total:** se muestra **solo cuando están los 12 meses prendidos**. Con cualquier subconjunto → **no hay columna de total**.
6. **%:** sin cambios — cada mes usa la base de ese mes (el filtro no altera el cálculo, solo qué se dibuja).
7. **Regla:** siempre queda ≥1 mes prendido (si se intenta apagar el último, se ignora).
8. **Sin persistencia:** al recargar vuelve a "todos". No afecta otras pestañas.

## 3. Enfoque técnico

**No recalcular.** `buildResumenMatrix(type)` sigue computando los 12 meses como hoy. Se separa el *render* del *cálculo*:

- Nuevo estado global: `resumenMonths` = `Set` de índices 0–11 (default 0..11).
- `renderResumen()` (existente): construye las matrices (eco/fin) y las guarda en `window._resumenMatrix`; luego llama a `renderResumenView()`.
- **Nuevo** `renderResumenView()`: re-renderiza tablas + gráficos de ambos tipos a partir de las matrices ya guardadas, aplicando el filtro de meses. Es lo que llaman los chips (no reconstruye la matriz → rápido).
- `renderResumenTable(containerId,type,matrix)` y `renderResumenChart(type)` pasan a recibir/usar la lista de meses visibles (`raVisibleMonths()` = índices del set, ordenados).

### Detalle de render
- **Tabla:** el `<thead>` emite `<th>` solo para los meses visibles; cada fila emite celdas solo para esos índices (`line.imp[m]`/`line.pct[m]`). La celda **Total** (`impYear`/`pctY`) se emite **solo si** `resumenMonths.size===12`. El `colspan` de los encabezados de sección se ajusta a `nº meses visibles (+1 si hay Total)`.
- **Gráfico:** `labels` = nombres de los meses visibles; cada dataset `data` = valores de esos índices (`s.imp`/`s.pct` mapeados por los índices visibles). Sin cambios en colores/etiquetas/toggle.

## 4. Integración (cambios en `index.html`)

1. **HTML:** dentro de `#sec-resumen`, **antes del primer `.resumen-block`**, un bloque de chips:
   `<div class="ra-month-filter">` con un label, 12 `<button class="ra-chip" data-m="0..11">Ene…Dic</button>` y un `<button class="ra-chip-all">Todos</button>`.
2. **CSS:** clases `.ra-month-filter`, `.ra-chip` (estado normal/activo con la paleta teal), `.ra-chip-all`.
3. **Estado:** `let resumenMonths=new Set([0,1,2,3,4,5,6,7,8,9,10,11]);` + helper `raVisibleMonths(){return [...resumenMonths].sort((a,b)=>a-b);}`.
4. **Funciones nuevas:** `renderResumenView()`, `toggleResumenMonth(i,btn)`, `setAllResumenMonths()`. Refactor menor de `renderResumen` para delegar el render en `renderResumenView`.
5. **Modificar** `renderResumenTable` y `renderResumenChart` para usar `raVisibleMonths()` y la regla de la columna Total.
6. Pintar el estado activo de los chips al iniciar (todos activos).

## 5. Casos borde

- Subconjunto de 1 mes → tabla con 1 columna (sin Total), gráfico con 1 punto por línea.
- Mes prendido sin datos → su columna en "—" (igual que hoy).
- Apagar el último mes prendido → ignorado (queda ≥1).
- Cambiar año/sucursal → recalcula matriz; el filtro de meses se mantiene (no se resetea).
- 12/12 prendidos → idéntico al comportamiento actual (incl. columna Total Año).

## 6. Fuera de alcance (YAGNI)

- Presets (trimestres/semestres), rango desde-hasta, persistencia, selector por-cuadro.
- Deshabilitar chips de meses sin datos (se permiten igual; muestran "—").
