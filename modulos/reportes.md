# Módulo de Reportes

## Visión general

Módulo de reportes operativos, gerenciales y documentos legales. Un solo
controller (`/api/reportes/*`) sobre varios servicios, con **scoping automático
por rol** y exportación en múltiples formatos.

```
Frontend reportes.ts (tabs por grupo + visibilidad por rol)
        │
        ▼
ReporteController  /api/reportes/*
        │
        ├── ReporteService            → cartera mora, flujo cobros, asesor, analista, Word (simulación/recibo)
        ├── ReportesAvanzadosService  → movimientos, créditos, cuadre, mora, cartera clientes, agencias, asesor
        ├── ReporteHistorialPagosService → historial de pagos
        ├── ReporteExtornosService    → extornos del mes
        ├── ExcelExportService        → 8 reportes .xlsx (delega data a los servicios de arriba)
        └── JasperReportService       → PDF (simulación, cronograma, recibo, contrato, cartera mora)
```

## Scoping automático por rol

`ReportesAvanzadosService.computarScope()` (y equivalente en los demás):

| Rol | Alcance |
|---|---|
| `CAJERO_COBRANZA` | Solo sus registros |
| `ANALISTA_CREDITO` | Sus clientes |
| `SUPERVISOR_CREDITO`, `GERENTE_AGENCIA` | Su agencia |
| `GERENTE_GENERAL`, `ADMINISTRADOR`, `AUDITOR` | Todas las agencias |

## Filtro por agencia (2026-06-01)

Los roles con **visibilidad global** (`GERENTE_GENERAL`, `ADMINISTRADOR`,
`AUDITOR`) pueden filtrar cualquier reporte por una agencia específica o ver
**todas las agencias**.

- Endpoint de agencias para el selector: `GET /api/v1/agencias` (cualquier
  autenticado) → `AgenciaService.listar()` en el front.
- Cada endpoint de reporte acepta `@RequestParam(required=false) Integer agenciaId`.
- Regla: el filtro `agenciaId` **solo aplica para roles globales**. Para roles
  ya restringidos por scope (cajero, asesor, supervisor, gerente agencia) se
  **ignora** — su scope manda y no pueden ver otras agencias.
- `agenciaId = null` ⇒ todas las agencias (para roles globales).
- Implementación central: `computarScope(Integer agenciaIdReq)` setea
  `scope.agenciaId` solo cuando el rol no impone ya una agencia.
- Frontend: un selector de agencia visible **solo** para roles globales, que se
  aplica a todas las pestañas de reportes.

## Formatos de salida

| Formato | Generador | Documentos |
|---|---|---|
| Pantalla (JSON) | servicios → DTO | todos los reportes de datos |
| PDF | JasperReportService + `.jrxml` | simulación, cronograma, recibo, recibo ticket 80mm, contrato, cartera mora |
| Word `.docx` | ReporteService, PrestamoDocumentService, DacionPago | simulación, recibo, contrato, pagaré, acta dación, resumen evaluación |
| Excel `.xlsx` | ExcelExportService | cartera mora, historial pagos, movimientos, cuadre, créditos, cartera clientes, historial mora, extornos |
| CSV | Frontend (`csvDownload`) | fallback para tabs sin Excel |

## Auto-verificación de cuadre y KPIs (2026-06-01)

Endpoint `GET /api/reportes/validacion` (ADMIN/GERENTE_GENERAL/AUDITOR) +
pestaña **Validación** (grupo Gerencial). Verifica exactitud sin valores
"esperados" manuales:

**Cuadre de caja** — por cada caja cerrada valida dos identidades contables:
1. `monto_inicial + ingresos − egresos == saldo_teorico` — recalcula
   ingresos/egresos en vivo; si falla hay **drift** (movimiento cambiado/anulado
   tras el cierre).
2. `monto_contado − saldo_teorico == diferencia` — aritmética pura; si falla, bug.

**KPIs** — recalcula cada indicador del dashboard con SQL independiente y lo
compara contra el valor del servicio. Detecta, entre otros, la **discrepancia de
definición de "cuota vencida"**: el dashboard usa `estado='VENCIDO'` mientras los
reportes usan `estado<>'PAGADO' AND fecha_venc < hoy` (lo marca con una nota).

## ⚠️ Issues conocidos (pendientes)

### Branding de documentos (white-label)

**✅ Fase 1 — nombre de marca unificado (2026-06-01):** todos los generadores
leen la razón social desde **`ParametroSistema.EMPRESA_RAZON_SOCIAL`** (default
REELIGE). Para rebrandear a un cliente, un admin solo cambia ese parámetro en
**Parámetros del Sistema** (ej. `CREDIDAR - Sistema Financiero`) y se aplica a:
- PDF (Jasper): cronograma, ticket, recibo, simulación, cartera mora, contrato
  (vía `$P{EMPRESA}`; footers de recibo y cartera mora actualizados).
- Word: simulación y recibo (`ReporteService.empresa()`).
- Excel: pies de los 8 reportes (`ExcelExportService.empresa()`).

**✅ Fase 2 — logos (2026-06-01):** los logos también son configurables vía
**`EMPRESA_LOGO_URL`** (logo subido por el cliente), con **fallback al bundled**
(cero cambio para Reelige):
- PDF: `recibo_pago`, `cartera_mora`, `contrato_prestamo`, `simulacion_prestamo`
  usan `$P{EMPRESA_LOGO_STREAM}` (stream resuelto en `JasperReportService.empresaLogoStream()`:
  archivo de `EMPRESA_LOGO_URL` si existe, si no `/img/reelige.png`).
- Excel: `ExcelHeaderBuilder` acepta un logo override; `ExcelExportService.brandLogoBytes()`
  lo lee desde `EMPRESA_LOGO_URL` (con detección PNG/JPEG) y fallback al bundled.
- Footer de simulación convertido a `$P{EMPRESA}`.
- ✔ Verificado por `JrxmlCompileTest` (compila las 6 plantillas Jasper).

**Cómo rebrandear a un cliente (ej. CREDIDAR):** en *Parámetros del Sistema*:
1. `EMPRESA_RAZON_SOCIAL = CREDIDAR - Sistema Financiero`
2. Subir el logo y setear `EMPRESA_LOGO_URL` (ruta `/api/files/...`)
3. (opcional) `EMPRESA_RUC`, `EMPRESA_DIRECCION`, etc.

**Pendiente menor:** en `contrato_prestamo.jrxml` quedan menciones "REELIGE" en
`<text>` **estáticos del cuerpo legal** (cláusulas) — requieren revisión legal +
conversión manual a `$P{EMPRESA}`; se dejaron intactas a propósito.

> ⚠️ Recomendado: generar un recibo/cartera-mora/contrato PDF de prueba con un
> `EMPRESA_LOGO_URL` cargado para validar el encuadre visual del logo del cliente.

### Columnas que devolvían 0
- ✅ **Interés por Cobrar** (consolidado agencia): la fórmula `GREATEST(0, X - GREATEST(0,X))`
  (siempre 0) se reemplazó por `SUM(cp.interes)` de las cuotas no pagadas de
  préstamos vigentes (`ReportesAvanzadosService.reporteAgencias`).
- ✅ **Mora Acumulada** (cartera clientes): el `0::numeric` se reemplazó por la
  deuda vencida pendiente: `SUM(cuota_total − monto_pagado)` de cuotas con
  `fecha_venc < hoy` y `estado <> 'PAGADO'` de los préstamos vigentes del cliente.
- ✅ **Total Mora** (cartera en mora): la mora-penalidad es **dinámica** (no se
  almacena en cuotas pendientes; la calcula cobranza al pagar). Ahora se computa
  invocando el motor de mora extraído (`MoraCalculator.paraCobranza`) por cada
  cuota vencida dentro de `ReporteService.getCarteraMora`. El método se marcó
  `@Transactional(readOnly = true)` para cargar el producto/préstamo (asociaciones
  lazy) dentro de la transacción. Usa la misma regla que cobranza (días hábiles
  vencidos × tasa, gracia, redondeo snapshot del préstamo, y "mora desde fecha fin"
  para diarios con el flag).
- ⏳ **Meta / % Meta** (cartera asesor): **no existe meta por asesor** en el modelo
  (las metas son por agencia). Decisión de producto: agregar metas por asesor o
  quitar las columnas de la vista.
- ✅ **Mora Total** (préstamos por asesor): es un reporte de **colocación**
  (cantidad + monto desembolsado). La mora real por asesor/analista ya está
  cubierta por el reporte dedicado `getMoraAnalista()` (campo `carteraMora`,
  desde SQL agregado). La columna `moraTotal` de `getPrestamosPorAsesor` se deja
  en 0 a propósito (documentado en el código) para no duplicar la métrica. No es
  una columna rota.

### Otros
- Código muerto: `cronogramaRepo.agingMora()` se ejecuta y no se usa (`ReporteService.getCarteraMora`).
- Simulador público (`ProductoCalculator`) no aplica el redondeo de cuota a 0.10 que sí tienen el cronograma real y la evaluación.
