# Proyecto Finanzas Personales — Memoria del Proyecto

> Archivo de contexto para Claude. Última actualización: abril 2026.

---

## 1. Arquitectura general

| Capa | Tecnología | Detalle |
|---|---|---|
| Frontend | HTML único (single-file) | Hosted en GitHub Pages |
| Backend | Google Apps Script | REST JSON sobre `doGet`/`doPost` |
| Base de datos | Google Sheets | Una hoja por tipo de dato |
| PDF parsing | pdf.js 3.11.174 (CDN) | Client-side, sin servidor |
| Gráficos | Chart.js 4.4.1 (CDN) | Bar, Doughnut, Line, Mixed |

**No hay build pipeline, no hay npm, no hay framework.** Todo es HTML/CSS/JS vanilla en un único `index.html`.

---

## 2. Los dos dashboards

### Dashboard de Daniel
- **Repo:** `xellosvzrd-lab/finanzas-dashboard`
- **URL:** GitHub Pages del repo anterior
- **Archivo:** `/sessions/epic-peaceful-dirac/finanzas-dashboard/index.html` (~4555 líneas)
- **Usuario fijo:** `USUARIO = "Daniel"`
- **Responsabilidad opciones:** `["Mío","Compartido","De Ama"]`
- **No tiene modo claro/oscuro** — siempre oscuro (`:root` dark only)
- **Render principal:** `_renderApp()` → llama `_normalizarCategorias()` + `cargarResumenMes()` + `cargarEvolucion()` + `filtrarTabla()` + `inicializarSelectoresCompartidos()` + `inicializarSelectoresPresupuesto()`

### Dashboard de Ama
- **Repo:** `xellosvzrd-lab/finanzas-dashboard-ama`
- **URL:** GitHub Pages del repo anterior
- **Archivo:** `/sessions/epic-peaceful-dirac/ama-dashboard/index.html` (~4682 líneas)
- **Usuario fijo:** `USUARIO = "Ama"`
- **Responsabilidad opciones:** `["Mío","Compartido","De Daniel"]`
- **Tiene modo claro/oscuro** — `[data-theme="light"]` con override de variables CSS
- **Render principal:** NO tiene `_renderApp()`. Llama directamente: `_normalizarCategorias()`, `cargarResumenMes()`, `cargarEvolucion()`, `filtrarTabla()` en cada punto de actualización
- **Tipo de cambio MEP:** `tipoCambioMEP` se fetchea automáticamente al abrir Presupuesto desde una API pública. Se usa para convertir USD → ARS en el cálculo del sueldo efectivo.

---

## 3. Variables globales clave (ambos archivos)

```javascript
let USUARIO = "";          // "Daniel" o "Ama"
let API_URL  = "";          // URL del Apps Script
let allTransac = [];        // TODAS las transacciones cargadas
let categGasto   = [];      // ej: ["Alimentación","Alquiler",...]
let categIngreso = [];      // ej: ["Sueldo","Otros Ingresos",...]
let categFuentes = [];      // ej: ["Efectivo","Débito",...]
let categResponsabilidad = [...]; // ver arriba, distinto por usuario
let presupuestoActual = {}; // { categoria: porcentaje }  ← ahora en %, no montos absolutos
let tipoCambioMEP = null;   // solo Ama — dólar MEP venta, para convertir USD a ARS

// Gráficos
let chartCat       = null;
let chartDonut     = null;
let chartEvolCombo = null;  // gráfico doble eje en Resumen (reemplazó chartEvol + chartBal)
```

---

## 4. Navegación (navIdx)

Después de eliminar la pestaña Evolución, el orden actual del sidebar es:

| idx | pagina | Dashboard |
|---|---|---|
| 0 | resumen | ambos |
| 1 | transacciones | ambos |
| 2 | nueva | ambos |
| 3 | compartidos | ambos |
| 4 | presupuesto | ambos |
| 5 | config | ambos |
| 6 | importar | ambos |

```javascript
const navIdx = { resumen: 0, transacciones: 1, nueva: 2, compartidos: 3, presupuesto: 4, config: 5, importar: 7 };
```

La pestaña **Evolución fue eliminada** del nav — su gráfico (doble eje: barras ingresos/gastos + línea balance acumulado) vive ahora al final de la página Resumen en `<canvas id="chart-evol-combo">`.

---

## 5. Funciones clave

### Normalización case-insensitive
```javascript
function _normalizarCategorias() {
  // Normaliza t.categoria, t.fuente, t.responsabilidad contra las listas canónicas
  // Se llama antes de cada render y después de cargar transacciones
}
```

### Parseo decimal (acepta coma argentina)
```javascript
function parsearDecimal(val) {
  return parseFloat(String(val || 0).replace(',', '.')) || 0;
}
```
**Todos los inputs de monto y % usan `type="text" inputmode="decimal"`** y sus valores se parsean con `parsearDecimal()`.

### Formateo
```javascript
fmt(n)           // $1.234,56  (es-AR, ARS)
fmtMoneda(n, moneda)  // USD → "U$S 1,234.56"
fmtShort(n)      // $1.2k / $1.2M
fmtFecha(s)      // "12 ene. 2026"
```

### Editar transacción (workaround)
El backend **no tiene** `updateTransaccion`. Se usa delete + add:
1. `POST { action: "deleteTransaccion", id }`
2. `POST { action: "addTransaccion", fecha, tipo, categoria, monto, descripcion, usuario, responsabilidad, fuente, moneda }`

---

## 6. Backend API (Apps Script)

### GET calls
```
?action=getTransacciones&usuario=Daniel
?action=getCategorias&usuario=Daniel
?action=getPresupuesto&mes=04&anio=2026&usuario=Daniel   ← usuario REQUERIDO
```

### POST calls (body JSON)
```javascript
{ action: "addTransaccion",    fecha, tipo, categoria, monto, descripcion, usuario, responsabilidad, fuente, moneda }
{ action: "deleteTransaccion", id }
{ action: "addCategoria",      tipo, valor, usuario }   ← usuario REQUERIDO
{ action: "deleteCategoria",   tipo, valor }
{ action: "savePresupuesto",   mes, anio, items, usuario }   ← usuario REQUERIDO
```

> ⚠️ `updateTransaccion` **NO existe** en el backend. Siempre usar delete + add.

---

## 7. Presupuesto

- Los inputs almacenan **porcentajes (0–100)**, no montos absolutos.
- El monto real se calcula en runtime: `(pct / 100) × salaryBase`
- **Daniel:** `salaryBase = ARS ingresos (Sueldo + Otros Ingresos)`
- **Ama:** `salaryBase = ARS ingresos + saldoUSD × tipoCambioMEP`
- El importe ARS se actualiza **en vivo** al tipear (`actualizarKpisPres()` → `.pres-monto-live` span)
- Datos separados por usuario via `usuario` en GET/POST

---

## 8. PDF Parser (importar datos)

- Usa `pdf.js` client-side
- **Formato Galicia VISA:** `DD-MM-YY` con guiones, 6 dígitos de comprobante como ancla estructural, dos columnas ARS/USD
- `_parsearLinea()` tiene regex Galicia primero, luego fallback genérico `DD/MM`
- La preview incluye columna de Responsabilidad (select editable)

---

## 9. Patrones CSS importantes

### Variables de tema (Ama tiene modo claro)
```css
:root { /* dark */ }
[data-theme="light"] { /* Ama only */ }
```

### Clases semánticas clave
```css
.kpi-card          /* tarjeta de KPI */
.kpi-trend         /* ▲/▼ con color verde/rojo */
.tabla-subtotal    /* barra de stats en Transacciones */
#sub-neto          /* valor grande 1.45rem — neto del período */
.sub-neto-pos      /* color: var(--green) */
.sub-neto-neg      /* color: var(--red) */
.nav-sep           /* separador de sección en sidebar */
.rafaga-overlay    /* modal Modo Ráfaga */
.pres-monto-live   /* span de importe ARS en vivo en Presupuesto */
```

### Modal Modo Ráfaga (Ama)
Usa **variables CSS** (`var(--card)`, `var(--border)`, `var(--bg2)`) para respetar el tema claro/oscuro. En Daniel no hay tema claro así que no importa.

---

## 10. Historial de commits relevantes

| Commit | Repo | Descripción |
|---|---|---|
| `7be08a9` | Daniel | fix case-insensitive categorías |
| `ff8b063` | Ama | fix updateTransaccion (delete+add workaround) |
| `f618062` | Daniel | 6 fixes: norm fuente/resp, addCategoria usuario, fusión Evolución, presupuesto %, neto KPI, evol combo |
| `4153d6a` | Ama | ídem + fix Modo Ráfaga tema |
| `779c401` | Daniel | importe ARS en vivo + separador decimal coma |
| `dcbf033` | Ama | ídem |
| `4a8d474` | Daniel | presupuesto separado por usuario (GET+POST con usuario) |
| `67b9390` | Ama | ídem |

---

## 11. Errores conocidos / workarounds

| Problema | Solución aplicada |
|---|---|
| `updateTransaccion is not defined` | delete + add en `guardarEdicionTransaccion()` |
| PDF Galicia no parsea | regex `DD-MM-YY` con comprobante como ancla |
| Categorías case-sensitive | `_normalizarCategorias()` normaliza al cargar |
| `fuente`/`responsabilidad` case-sensitive | incluidos en `_normalizarCategorias()` |
| Categorías nuevas no aparecen en selects | `addCategoria` ahora envía `usuario` |
| Presupuesto compartido entre usuarios | `getPresupuesto` y `savePresupuesto` ahora envían `usuario` |
| Modo Ráfaga siempre oscuro en Ama | CSS variables en lugar de colores hardcodeados |
| Decimal con coma no acepta | `type="text" inputmode="decimal"` + `parsearDecimal()` |

---

## 12. Estructura de una transacción

```javascript
{
  id:              string,   // ID del backend
  fecha:           "YYYY-MM-DD",
  tipo:            "Gasto" | "Ingreso",
  categoria:       string,   // de categGasto o categIngreso
  monto:           number,   // siempre positivo, el tipo indica dirección
  descripcion:     string,
  usuario:         "Daniel" | "Ama",
  responsabilidad: "Mío" | "Compartido" | "De Ama" | "De Daniel",
  fuente:          string,   // de categFuentes
  moneda:          "ARS" | "USD"
}
```

---

## 13. Categorías especiales (constantes en código)

```javascript
// Daniel
CATS_TRANSFERENCIA = ["Transferencia"]  // excluidas de gráficos
CATS_INGRESO_REAL  = ["Sueldo", "Otros Ingresos"]  // para cálculo de sueldo

// Ama
CATS_INGRESO_ARS  = ["Sueldo", "Otros Ingresos", "Intereses"]
CATS_EXCLUIR      = ["Cambio"]  // cambio de divisas, tratamiento especial
// "Cambio" ARS = ingreso del pesos recibido al vender USD
```

---

## 14. Lógica de responsabilidad en Presupuesto y Compartidos

### Daniel
- `"Mío"` gastos Daniel → 100%
- `"Compartido"` gastos (todos) → 50% (neto de reintegros compartidos)
- `"De Ama"` gastos Daniel → 0% (Daniel los pagó, Ama le debe)
- `"De Daniel"` gastos Ama → 100% de Daniel (Ama pagó por él)

### Ama (simétrico)
- `"Mío"` gastos Ama → 100%
- `"Compartido"` → 50%
- `"De Ama"` gastos que pagó Daniel → 100% de Ama
- `"De Daniel"` → 0%
