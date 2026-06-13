# Bitácora de Pruebas — Sistema Financiero

> Registro **vivo** de todas las pruebas: qué cubre cada una, su estado, y el **plan recomendado**
> para completar la cobertura. Se actualiza cada vez que se agrega o cambia una prueba.
>
> Documentos relacionados: estrategia e invariantes en
> [`plan-de-pruebas.md`](./plan-de-pruebas.md); reglas en [`../reglas-negocio/`](../reglas-negocio/).

---

## 1. Inventario de pruebas implementadas

> Backend (JUnit 5 + Spring Boot + H2 aislado). Correr con `./mvnw test`.

| Clase | Método / caso | Qué verifica | Cubre | Estado |
|---|---|---|---|---|
| `H2ContextLoadTest` | `contextLoadsWithH2` | el schema JPA carga en H2 | infraestructura | ✅ |
| `AppApplicationTests` | `contextLoads` | el contexto arranca (BD dev) | infraestructura | ✅ |
| `JrxmlCompileTest` | (compila .jrxml) | plantillas Jasper compilan | documentos | ✅ |
| `FlujoPrestamoIntegrationTest` | `flujoEvaluacionAprobacionPrestamo` | flujo completo: evaluación→aprobación→préstamo `PRE_DESEMBOLSO`+cronograma→desembolso `VIGENTE`→pago `PAGADO` | Flujo P1 | ✅ |
| `RbacIntegrationTest` | `desembolso_denegadoParaAnalista` | ANALISTA en /desembolso → **403** | RBAC F | ✅ |
| `RbacIntegrationTest` | `desembolso_denegadoParaAuditor` | AUDITOR en /desembolso → **403** | RBAC F | ✅ |
| `RbacIntegrationTest` | `desembolso_permitidoParaCajero_pasaElGate` | CAJERO pasa el `@PreAuthorize` | RBAC F | ✅ |
| `RbacIntegrationTest` | `desembolso_permitidoParaAdmin_pasaElGate` | ADMIN pasa el gate | RBAC F | ✅ |
| `RbacIntegrationTest` | `desembolso_sinToken_esNoAutorizado` | sin token → 401/403 | RBAC F | ✅ |
| `FlujoNegativoTest` | `aprobar_sinProductoFinal_lanzaExcepcion` | aprobar sin producto → `IllegalArgumentException` | Validación C | ✅ |
| `FlujoNegativoTest` | `desembolsar_sinCajaAbierta_lanzaExcepcion` | desembolsar sin caja → excepción | **Dinero D6** | ✅ |
| `PagosIntegrationTest` | `pagoParcial_dejaCuotaEnParcial` | pago < cuota → estado `PARCIAL` | Pagos D | ✅ |
| `PagosIntegrationTest` | `pagoCuotaVencida_cobraMora` | cuota vencida → `importeMora` > 0 + `PAGADO` | **Dinero D3** | ✅ |
| `DineroConservacionTest` | `desembolsoEfectivo_conservaCaja` | cargo EFECTIVO → saldo teórico = físico | **Dinero D1** | ✅ |
| `DineroConservacionTest` | `desembolsoDescontado_conservaCaja` | cargo DESCONTADO → también conserva (**HALL-06 corregido**) | **Dinero D1** | ✅ |
| `DineroConservacionTest` | `extornoDesembolso_neutralizaCaja` | extornar desembolso → préstamo revierte **y** EGRESO se neutraliza (**HALL-08 corregido**) | **Dinero D5** | ✅ |
| `TasaAprobadaCronogramaTest` | `productoSimple_ignoraLaTasaAprobada_usaLaDelProducto` | SIMPLE: cronograma usa tasa del producto, no la aprobada (**confirma HALL-11**) | **Dinero D3** | ✅ |
| `MovimientoAtomicoTest` | `desembolso_siFallaElMovimientoDeCaja_revierteTodo` | si falla el movimiento → desembolso revierte (**HALL-07 corregido**) | **Dinero D1/D2** | ✅ |
| `CajaCierreTest` | `segundaCajaMismoDia_esRechazada` | un cajero, una caja por día (RN-CAJA-03) | Caja | ✅ |
| `CajaCierreTest` | `cierreCuadrado_quedaCerrada` | contado = teórico → CERRADA, dif 0 (RN-CAJA-05/13) | **Dinero D4** | ✅ |
| `CajaCierreTest` | `cierreDescuadrado_conBilletajeExacto_esBloqueado` | descuadre + billetaje exacto → bloqueado (RN-CAJA-11) | **Dinero D4** | ✅ |
| `CajaCierreTest` | `cierreDescuadrado_paramOff_sinObservacion_exigeObservacion` | descuadre sin observación → rechazado (RN-CAJA-12) | **Dinero D4** | ✅ |
| `CajaCierreTest` | `cierreDescuadrado_paramOff_conObservacion_quedaDescuadre` | descuadre con observación → DESCUADRE (RN-CAJA-13) | **Dinero D4** | ✅ |
| `CronogramaCalculoTest` | `simpleSaldo_conservaCapital_interesDecreciente` | SIMPLE_SALDO: Σamort=capital, interés decreciente (RN-CRON-02) | **Dinero D3** | ✅ |
| `CronogramaCalculoTest` | `simpleFlat_conservaCapital` | SIMPLE_FLAT: Σamort=capital (RN-CRON-01) | **Dinero D3** | ✅ |
| `CronogramaCalculoTest` | `frances_conservaCapital` | FRANCES: Σamort=capital (RN-CRON-03) | **Dinero D3** | ✅ |
| `CronogramaCalculoTest` | `graciaParcial_soloInteres_noAmortiza` | gracia PARCIAL: solo interés (RN-CRON-11) | **Dinero D3** | ✅ |
| `CronogramaCalculoTest` | `graciaTotal_sinPago` | gracia TOTAL: cuota = 0 (RN-CRON-12) | **Dinero D3** | ✅ |
| `PagoLiquidacionTest` | `pagarTodasLasCuotas_liquidaElPrestamo` | pagar todo → `LIQUIDADO` (**HALL-12 corregido**) | RN-PAGO | ✅ |
| `PagoLiquidacionTest` | `pagoExactoDeCuota_quedaPagada` | pago exacto → PAGADO | RN-PAGO | ✅ |
| `PagoLiquidacionTest` | `pagoConExcedente_seAplicaASiguienteCuota` | excedente → abona la cuota siguiente | RN-PAGO | ✅ |
| `MigracionHall12Test` | `migracion_soloPasaALiquidadoLosPagados` | migración HALL-12: pagado→LIQUIDADO, cancelado admin intacto | Migración | ✅ |

**Total backend: 32 pruebas en verde.**

> Frontend (Karma/Jasmine): 9 smoke tests `should create` (`npm run test:ci`).

### E2E (Playwright — frontend, `npm run e2e`)

| Proyecto | Test | Qué verifica | Backend |
|---|---|---|---|
| smoke | redirige a /login y muestra el formulario | app servida + login renderiza | ❌ no requiere |
| smoke | formulario vacío muestra errores requeridos | validación del form | ❌ no requiere |
| setup | authenticate | login real por UI + guarda JWT (`storageState`) | ✅ requiere |
| authenticated | entra autenticado y abre el dashboard | sesión válida, no rebota a login | ✅ requiere |

**Total E2E: 4 pruebas en verde** (verificadas contra dev: `ng serve` :4200 + backend :8081 + `dbFinanciera`).

---

## 2. Cobertura de las invariantes del DINERO (D1–D7)

| Invariante | Descripción | Estado | Prueba |
|---|---|---|---|
| **D1** | Conservación del saldo | ✅ | `DineroConservacionTest` (desembolso EFECTIVO y DESCONTADO conservan, tras fix HALL-06) |
| **D2** | Trazabilidad (todo movimiento registrado) | 🟡 parcial | `DineroConservacionTest` (EGRESO/INGRESO con montos exactos) + `MovimientoAtomicoTest` (atómico, tras fix HALL-07) |
| **D3** | Exactitud de montos | ✅ | `CronogramaCalculoTest` (Σamort=capital, redondeo 0.10) + `pagoCuotaVencida_cobraMora` |
| **D4** | Cuadre cuadrado/descuadrado | ✅ | `CajaCierreTest` (cuadrado, descuadrado, billetaje exacto, observación) |
| **D5** | Extorno reversible | 🟡 parcial | `DineroConservacionTest.extornoDesembolso_neutralizaCaja` (tras fix HALL-08); falta extorno de **pago** |
| **D6** | No hay dinero sin caja | ✅ | `desembolsar_sinCajaAbierta…` |
| **D7** | Sin doble cobro | ⏳ pendiente | — (RN-EXT-03 sin prueba directa) |

> 🟢 **5 de 7 invariantes cubiertas o casi.** Pendientes: extorno de pago (D5), no-doble-extorno (D7),
> y el INGRESO del pago verificado contra el movimiento (D2 completo).

---

## 3. Cobertura por escenario (matriz)

| Área | Cobertura | Estado |
|---|---|---|
| Flujo feliz completo | 1 caso | ✅ |
| RBAC (desembolso) | 5 casos | ✅ |
| Validaciones/rechazos | 2 casos | 🟡 inicial |
| Pagos (parcial, mora, liquidación, excedente) | 5 casos | ✅ |
| Caja: cuadre y cierre | 5 casos (`CajaCierreTest`) | ✅ |
| Caja: conservación / movimientos / atomicidad | 4 casos (`DineroConservacionTest`, `MovimientoAtomicoTest`) | ✅ |
| Extornos | 1 caso (desembolso neutraliza) | 🟡 falta pago y no-doble-extorno |
| Cálculo por producto (FLAT/SALDO/FRANCES + gracia) | 5 casos (`CronogramaCalculoTest`) | ✅ |
| Tasa aprobada vs producto | 1 caso (fija HALL-11, pendiente decisión) | 🟡 |
| **Scope por agencia/rol** | 0 — requiere Testcontainers (SQL nativo PG) | 🔴 pendiente |
| Reportes / documentos | jrxml compila | 🟡 mínimo |
| E2E (Playwright) | 4 casos (smoke + login + dashboard) | ✅ inicial |

---

## 4. ✅ Plan recomendado (roadmap por fases)

> Orden por **criticidad** (dinero y seguridad primero). Cada fase: tests + regla documentada + verde en `./mvnw test`.

### ✅ Fase 0 — Infraestructura + POC *(HECHA, 2026-06-11)*
H2 aislado, flujo completo, RBAC base, negativos, pagos básicos. → 13 tests verdes.

### ✅ Fase 1 — DINERO *(HECHA en lo esencial, 2026-06-12)*
- ✅ Conservación en desembolso EFECTIVO/DESCONTADO (D1) — y **corrigió HALL-06**.
- ✅ Atomicidad del movimiento de caja (D2) — **corrigió HALL-07**.
- ✅ Cuadre cuadrado/descuadrado/billetaje exacto/observación (D4).
- ✅ Extorno de desembolso neutraliza la caja (D5) — **corrigió HALL-08**.
- ✅ Pago: parcial, mora, exacto, excedente, liquidación — **corrigió HALL-12**.
- ⏳ Restan: extorno de **pago** (D5), no-doble-extorno (D7), regularización >24h, descuento por faltante.

### ✅ Fase 3 — Cálculo / Productos *(HECHA en lo esencial, 2026-06-12)*
- ✅ FLAT/SALDO/FRANCES conservan capital; redondeo 0.10; gracia PARCIAL/TOTAL.
- ⏳ Restan: mora FIJO/FIJO_DIARIO_HABILES, mora desde fecha fin, fechas en día hábil.
- ⚠️ Tasa aprobada vs producto: **HALL-11 confirmado**, pendiente decisión de negocio.

### 🔴 Fase 2 — Seguridad / Scope *(BLOQUEADA por infra)*
- `computarScope` vive en SQL nativo PostgreSQL → **no corre en H2**. Requiere **Testcontainers
  + PostgreSQL** (Docker). Al montarlo, entra también la Fase 4 (reportes con SQL nativo).

### 🟨 Fase 4 — Reportes / Documentos
- Cartera, cuadre, cartera mora (con Testcontainers); generación Word/PDF.

### ✅ Fase 5 — E2E de UI *(INICIADA, 2026-06-12)*
- ✅ Playwright montado: smoke (sin backend) + login real + dashboard autenticado (4 tests).
- ⏳ Siguiente: flujos por rol contra **BD de pruebas desechable** (los E2E que mutan dinero).

---

## 5. Convenciones de la bitácora

- Cada fila del inventario (§1) se agrega **al implementar** la prueba.
- Al cerrar una invariante de dinero, marcar ✅ en §2.
- Los hitos se registran abajo (§6) con fecha.

---

## 6. Historial (cronológico)

| Fecha | Hito |
|---|---|
| 2026-06-11 | Fase 0: infraestructura H2 + flujo completo + RBAC + negativos + pagos. 13 tests verdes. |
| 2026-06-11 | Creado `plan-de-pruebas.md` (estrategia + invariantes D1–D7) y esta bitácora. |
| 2026-06-12 | Documentadas las reglas de negocio del núcleo (15 dominios) → 12 hallazgos registrados. |
| 2026-06-12 | Fase 1 DINERO: conservación, cuadre, atomicidad, extornos. **Corregidos HALL-06/07/08** + limpieza HALL-09. |
| 2026-06-12 | Fase 3 Cálculo: FLAT/SALDO/FRANCES + gracia conservan capital (D3). |
| 2026-06-12 | Lote Pago: liquidación/excedente → destapó y **corrigió HALL-12** (pagado → LIQUIDADO). |
| 2026-06-12 | Fase 5 E2E: Playwright montado y verificado contra dev (4 tests: smoke + login + dashboard). |
| 2026-06-12 | **31 tests backend + 4 E2E en verde.** Pendientes: Scope (Testcontainers), decisiones HALL-01/11. |

---

*Bitácora viva: actualizar al agregar/cambiar pruebas y al cerrar invariantes.*
