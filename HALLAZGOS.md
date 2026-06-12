# Registro de Hallazgos — Revisión de Reglas de Negocio

> Bitácora central de **bugs, derivas y decisiones pendientes** que aparecen al documentar el
> sistema regla por regla. La idea: **anotar ahora, revisar y resolver al final** — sin frenar
> la documentación cada vez que aparece algo.
>
> Relacionado: [`reglas-negocio/README.md`](./reglas-negocio/README.md) ·
> [`desarrollo/bitacora-pruebas.md`](./desarrollo/bitacora-pruebas.md)

## Cómo se clasifica

| Campo | Valores |
|---|---|
| **Tipo** | 🐞 BUG (código) · 📄 Deriva doc · 🤝 Decisión de negocio · ✨ Mejora |
| **Severidad** | 🔴 Alta (afecta dinero/seguridad) · 🟧 Media · 🟨 Baja |
| **Estado** | 🆕 Abierto · 🔍 En análisis · ⏳ Decisión pendiente · ✅ Resuelto |

> Regla: todo hallazgo lleva **evidencia** (`archivo:método`) y, si aplica, el **ID de regla**
> (`RN-…`) que lo destapó. Los 🔴 que tocan dinero tienen prioridad de revisión.

---

## Hallazgos abiertos (revisar)

### HALL-01 · Mora divergente: cobranza (hábiles) vs caja (calendario)
- **Tipo:** 🐞 BUG + 🤝 Decisión · **Severidad:** 🔴 Alta (dinero) · **Estado:** ⏳ Decisión pendiente
- **Regla:** `RN-MORA-05` · **Origen:** documentación del Flujo del Préstamo (#2)
- **Evidencia:** `service/MoraCalculator.java` → `paraCobranza()` cuenta días **HÁBILES**;
  `paraPago()` cuenta días **CALENDARIO**. El Javadoc de la clase lo declara *"un bug
  preexistente que queda pendiente de reconciliar como decisión de negocio"*.
- **Impacto:** una misma cuota vencida puede mostrar **una mora en la pantalla de cobranza y
  cobrar otra en caja** (para `PORCENTAJE`, caja suele cobrar más).
- **Acción propuesta:** definir con negocio la regla correcta (hábiles o calendario), unificar
  `MoraCalculator`, y blindar con una prueba que compare ambos cálculos sobre la misma cuota.

### HALL-06 · Doble conteo del cargo en desembolso DESCONTADO  ✅ CORREGIDO
- **Tipo:** 🐞 BUG · **Severidad:** 🔴 Alta (dinero) · **Estado:** ✅ **Corregido (2026-06-12)**
- **Fix aplicado:** `PrestamoServiceImpl.confirmarDesembolso` — el EGRESO `DESEMBOLSO_PRESTAMO`
  ahora es **siempre el bruto** (antes en DESCONTADO era neto). El cargo se sigue registrando
  como INGRESO aparte. Efecto neto en caja = `−neto` en ambos modos → conserva. El test
  `desembolsoDescontado_conservaCaja` (verde) valida la conservación.
- **Regla:** `RN-MOV-05` · **Origen:** documentación de Movimientos de Caja (#4)
- **Evidencia:** `service/PrestamoServiceImpl.java:441-495`. En modo `DESCONTADO`, el EGRESO
  `DESEMBOLSO_PRESTAMO` se reduce a `neto = bruto − cargo` **y además** se registra un INGRESO
  `CARGO_DESEMBOLSO = cargo`. Efecto en caja = `−neto + cargo`; físicamente solo salió `neto`.
- **Prueba (verde):** `DineroConservacionTest` (backend):
  `desembolsoDescontado_inflaSaldoTeoricoEnElCargo` demuestra que el saldo teórico queda por
  encima del físico **exactamente en `cargo`**; `desembolsoEfectivo_conservaCaja` confirma que el
  modo EFECTIVO **sí conserva**.
- **Impacto confirmado:** cada desembolso con cargo descontado genera un **faltante = cargo** al
  cuadrar. Con `CIERRE_CAJA_BILLETAJE_EXACTO=true` (default, RN-CAJA-11) **bloquearía el cierre**.
- **Acción propuesta:** corregir el modelo del EGRESO en DESCONTADO — opción A: EGRESO=bruto +
  INGRESO=cargo (igual que EFECTIVO); opción B: EGRESO=neto **sin** INGRESO del cargo. Tras el
  fix, ajustar el test (el teórico debe igualar al físico).

### HALL-07 · El registro del movimiento de caja no revertía la operación si fallaba  ✅ CORREGIDO
- **Tipo:** 🐞 BUG · **Severidad:** 🔴 Alta (dinero) · **Estado:** ✅ **Corregido (2026-06-12)**
- **Regla:** `RN-MOV-08` · **Origen:** documentación de Movimientos de Caja (#4)
- **Evidencia original:** en `PagoCajeroServiceImpl` y `PrestamoServiceImpl`, `registrarAutomatico`
  iba en un `try/catch` que solo **logueaba un warning** → si fallaba, el pago/desembolso quedaba
  persistido **sin movimiento de caja** (dinero fuera del cuadre).
- **Fix aplicado:** ambos `catch` ahora **re-lanzan** (`IllegalStateException`); como los métodos
  son `@Transactional`, el fallo del movimiento **revierte toda la operación** (atómico).
- **Prueba (verde):** `MovimientoAtomicoTest.desembolso_siFallaElMovimientoDeCaja_revierteTodo`:
  forzando el fallo del movimiento (spy de `CajaService`), el desembolso propaga el error y el
  préstamo queda en `PRE_DESEMBOLSO` (no VIGENTE).

### HALL-08 · Extornar pago/desembolso no neutraliza el movimiento de caja  ✅ CORREGIDO
- **Tipo:** 🐞 BUG · **Severidad:** 🔴 Alta (dinero) · **Estado:** ✅ **Corregido (2026-06-12)**
- **Regla:** `RN-EXT` / HALL-08 · **Origen:** documentación de Extornos (#5)
- **Evidencia original:** `PagoCuotaExtornoHandler` y `DesembolsoExtornoHandler` revertían el lado
  préstamo pero **no marcaban** el `movimiento_caja` automático como `anulado`/`extornado`.
- **Fix aplicado:** ambos handlers ahora **neutralizan los movimientos de caja asociados** (por
  `referencia` = N° contrato / N° recibo): los marcan `anulado=true` + `extornado=true`. Se agregó
  `MovimientoCajaRepository.findByReferencia`. El saldo de caja suma solo `anulado=false`, así el
  reverso se refleja en el cuadre — consistente con `MovimientoCajaExtornoHandler`.
- **Prueba (verde):** `DineroConservacionTest.extornoDesembolso_neutralizaCaja`: tras el extorno,
  el préstamo vuelve a `APROBADO` y `sumEgresosPorApertura` queda en **0**.
- **Supuesto:** el reverso asume que el efectivo vuelve físicamente (cliente devuelve el desembolso
  / cajero devuelve el cobro), que es el caso normal de un extorno.

### HALL-09 · Calculadora de cuotas duplicada / legacy con enum inconsistente
- **Tipo:** 🐞 BUG / ✨ Mejora · **Severidad:** 🟧 Media · **Estado:** 🔍 En análisis
- **Regla:** `RN-CRON` / HALL-09 · **Origen:** documentación de Cálculo de Cuotas (#6)
- **Evidencia:** existen **tres** motores de cálculo de cuotas:
  `utils/PrestamoCalculator` (cronograma persistido al aprobar, +ALEMAN),
  `utils/ProductoCalculator` (simulación) — ambos **duplican** la lógica FLAT/SALDO/FRANCES — y
  `service/CalculoCuotasServiceImpl` (legacy, enum `AMORTIZADO/LINEAL` incompatible, sin gracia
  ni redondeo a 0.10).
- **Impacto:** duplicación → riesgo de divergencia/mantenimiento; el legacy genera cronogramas
  distintos si algún flujo lo invoca.
- **Acción propuesta:** verificar call sites del legacy; si está huérfano, eliminarlo. Evaluar
  unificar `PrestamoCalculator` y `ProductoCalculator` en un solo motor.

### HALL-10 · Cuota estimada de la evaluación puede diferir del cronograma real
- **Tipo:** 🐞 BUG (a confirmar) · **Severidad:** 🟧 Media · **Estado:** 🔍 En análisis
- **Regla:** `RN-EVAL` / HALL-10 · **Origen:** documentación de Evaluación (#7)
- **Evidencia:** `EvaluacionCreditoCalculator` (FRANCES `r = tasa/100`) vs `ProductoCalculator`
  (`tem = (1+TEA)^(1/12)−1`); FLAT real escala con `mesesEquiv`.
- **Impacto:** el análisis de capacidad/ratio usa una cuota que puede no coincidir con la real.
- **Acción propuesta:** confirmar si es tolerable (solo "estimado") o unificar las fórmulas.

### HALL-11 · La tasa aprobada por el comité no se aplica en productos SIMPLE  ✅ CONFIRMADO
- **Tipo:** 🐞 BUG · **Severidad:** 🔴 Alta (dinero) · **Estado:** ✅ **Confirmado con prueba**
- **Regla:** `RN-APRO` / HALL-11 · **Origen:** documentación de Aprobación/Comité (#8)
- **Evidencia:** `AprobacionCreditoServiceImpl.crearPrestamoDesdeAprobacion`: `tasaInteresAnual =
  tasaFinalAprobada` pero `tasaInteresPeriodo = producto.getTasaInteres()`. En `PrestamoCalculator`,
  FRANCES/ALEMAN leen `tasaInteresAnual` (usa aprobada); SIMPLE_FLAT/SIMPLE_SALDO leen
  `tasaInteresPeriodo` (usa la del producto, **ignora la aprobada**).
- **Prueba (verde):** `TasaAprobadaCronogramaTest.productoSimple_ignoraLaTasaAprobada_usaLaDelProducto`:
  producto SIMPLE_SALDO 5%, comité aprueba 10%, crédito de 1 cuota de 1000 → el cronograma cobra
  interés **50.00** (5%), no 100.00 (10% aprobado).
- **Impacto confirmado:** si el comité aprueba una tasa distinta a la del producto, en créditos
  SIMPLE el cliente paga con la del producto, no la aprobada. Microcrédito suele ser SIMPLE.
- **Acción propuesta:** definir si la tasa la fija el producto o el comité (decisión de negocio).
  Si manda el comité, asignar `tasaInteresPeriodo = tasaFinalAprobada` en SIMPLE; tras el fix,
  actualizar el test para esperar 100.00.

---

## Hallazgos resueltos durante la revisión

### HALL-02 · Doc de scope inexacta ("cada usuario solo ve su agencia")
- **Tipo:** 📄 Deriva doc · **Severidad:** 🟧 Media · **Estado:** ✅ Resuelto (2026-06-12)
- **Origen:** regla #1 (Roles y Permisos). El código (`computarScope`) es correcto; la doc no.
- **Detalle:** cajero ve **solo sus registros**, analista **solo sus clientes**, auditor es
  **global** — no "cada quien su agencia". Corregido en `roles-permisos.md`.

### HALL-03 · Matriz de permisos de Garantías incorrecta
- **Tipo:** 📄 Deriva doc · **Severidad:** 🟧 Media · **Estado:** ✅ Resuelto (2026-06-12)
- **Origen:** validación de la regla #1.
- **Detalle:** `ANALISTA_CREDITO` **sí** registra/edita garantías y `GERENTE_GENERAL` **solo**
  consulta (no escribe). Verificado contra `BienGarantiaController` y corregido.

### HALL-04 · Doc del flujo afirmaba "mora por días hábiles" (genérico)
- **Tipo:** 📄 Deriva doc · **Severidad:** 🟨 Baja · **Estado:** ✅ Resuelto (2026-06-12)
- **Detalle:** reemplazado por la descripción real (depende del contexto) → ver HALL-01.

### HALL-05 · README raíz con índice roto (~10 enlaces a archivos inexistentes)
- **Tipo:** 📄 Deriva doc · **Severidad:** 🟧 Media · **Estado:** ✅ Resuelto (2026-06-12)
- **Origen:** reorganización de la documentación.
- **Detalle:** el índice apuntaba a `modulos/evaluacion-credito.md`, `reglas-negocio/mora.md`,
  `desarrollo/convenciones.md`, etc. — ninguno existía. Reescrito para reflejar solo lo real.

---

## Para vigilar (a confirmar al documentar el módulo)

| # | Sospecha | Dónde mirar | Cuándo |
|---|---|---|---|
| ~~V-01~~ | ✅ **Confirmado control correcto** (no bug): `CIERRE_CAJA_BILLETAJE_EXACTO` (default true) bloquea el cierre con descuadre. → `RN-CAJA-11` | `CajaServiceImpl.cerrar()` | ✅ regla #3 |
| ~~V-02~~ | ✅ **Confirmado parcialmente**: manual sí neutraliza caja (RN-EXT-13), idempotente y transaccional (RN-EXT-03/04). **PERO** pago/desembolso no neutralizan caja → ver **HALL-08** | `service/extorno/*Handler` | ✅ regla #5 |
| ~~V-03~~ | ✅ **Confirmado control correcto** (no bug): regularización >24h solo Gerente/Admin. → `RN-CAJA-16` | `CajaServiceImpl.regularizar()` | ✅ regla #3 |
| ~~V-04~~ | ✅ **Confirmado correcto**: cuota redondea a 0.10 superior, delta al interés, última cuota absorbe residuos → Σamort=capital. → `RN-CRON-05..07` | `utils/ProductoCalculator` | ✅ regla #6 |

---

## Resumen para la revisión final

| Estado | Cantidad |
|---|---|
| ✅ Corregido (con fix + prueba) | 3 (HALL-06 doble conteo cargo descontado, HALL-08 extorno no neutraliza caja, HALL-07 movimiento no atómico) |
| 🔴 Confirmado con prueba (falta fix) | 1 (HALL-11 tasa aprobada ignorada en SIMPLE) |
| 🔍 En análisis | 2 (HALL-09 calc duplicada, HALL-10 cuota estimada≠real) |
| ⏳ Decisión pendiente (dinero) | 1 (HALL-01 mora) |
| ✅ Resuelto / confirmado correcto | 6 (HALL-02..05, V-01, V-03, V-04) |
| 👀 Para vigilar | 0 |

> 🔴 **Prioridad de revisión:** HALL-06 y HALL-07 tocan el cuadre directamente. Se confirman/
> descartan al implementar las pruebas de conservación de la **Fase 1 — DINERO**.

*Registro vivo: se agrega un hallazgo cada vez que la documentación o las pruebas destapan algo.
Última actualización: 2026-06-12.*
