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
| `TasaAprobadaCronogramaTest` | `productoSimple_usaLaTasaAprobadaPorElComite` | SIMPLE: cronograma usa la tasa **aprobada por el comité** (**HALL-11 corregido**) | **Dinero D3** | ✅ |
| `TasaAprobadaCronogramaTest` | `comite_noPuedeAprobarTasaFueraDelRangoDelProducto` | la tasa del comité debe respetar `tasaMin/tasaMax` del producto | **Dinero** · RN-APRO-05b | ✅ |
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
| `PostgresContextLoadTest` 🐘 | `contextLoadsConPostgresReal` | PostgreSQL real (Testcontainers) bootea + SQL nativo | infraestructura | ✅ |
| `ScopeCarteraClientesPostgresTest` 🐘 | `scope_cadaRolVeSoloLoQueLeCorresponde` | scope: analista→sus clientes, gerente→su agencia, admin→todo (RN-ROL) | **Seguridad de datos** | ✅ |
| `ExtornoPagoTest` | `extornoPago_revierteCuotaYNeutralizaCaja` | extorno de pago: revierte cuota + neutraliza INGRESO (RN-EXT) | **Dinero D5** | ✅ |
| `ExtornoPagoTest` | `noSePuedeExtornarDosVeces` | no se extorna dos veces el mismo documento (RN-EXT-03) | **Dinero D7** | ✅ |
| `MovimientoTrazabilidadTest` | `desembolso_registraEgresoConConceptoYMonto` | desembolso → EGRESO `DESEMBOLSO_PRESTAMO` = bruto | **Dinero D2** | ✅ |
| `MovimientoTrazabilidadTest` | `pago_registraIngresoConConceptoYMonto` | pago → INGRESO `COBRO_CUOTA` = importe cobrado | **Dinero D2** | ✅ |
| `MoraDiasHabilesTest` | `fijoDiarioHabiles_excluyeSoloDomingo` | mora días hábiles: excluye solo domingo (sábado cuenta) | Mora · **HALL-01** | ✅ |
| `MoraDiasHabilesTest` | `cajaYCobranzaPorcentaje_coinciden_enDiasHabiles` | caja y cobranza cobran igual, por días hábiles (HALL-01 corregido) | Mora · **HALL-01** | ✅ |

**Total backend: 41 pruebas en verde** (🐘 = requieren Docker/Testcontainers, PostgreSQL real).

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
| **D2** | Trazabilidad (todo movimiento registrado) | ✅ | `MovimientoTrazabilidadTest` (EGRESO `DESEMBOLSO_PRESTAMO` / INGRESO `COBRO_CUOTA` con concepto+monto) + `MovimientoAtomicoTest` (atómico) |
| **D3** | Exactitud de montos | ✅ | `CronogramaCalculoTest` (Σamort=capital, redondeo 0.10) + `pagoCuotaVencida_cobraMora` |
| **D4** | Cuadre cuadrado/descuadrado | ✅ | `CajaCierreTest` (cuadrado, descuadrado, billetaje exacto, observación) |
| **D5** | Extorno reversible | ✅ | `DineroConservacionTest` (desembolso) + `ExtornoPagoTest` (pago revierte + neutraliza caja) |
| **D6** | No hay dinero sin caja | ✅ | `desembolsar_sinCajaAbierta…` |
| **D7** | Sin doble cobro / doble extorno | ✅ | `ExtornoPagoTest.noSePuedeExtornarDosVeces` |

> 🟢 **Las 7 invariantes del dinero (D1–D7) cubiertas.** El bloque dinero —caja, desembolso,
> pago, movimientos, extornos— tiene red de seguridad completa.

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
| Extornos | 3 casos (desembolso + pago neutralizan caja, no doble extorno) | ✅ |
| Cálculo por producto (FLAT/SALDO/FRANCES + gracia) | 5 casos (`CronogramaCalculoTest`) | ✅ |
| Tasa aprobada vs producto | 1 caso (fija HALL-11, pendiente decisión) | 🟡 |
| **Scope por agencia/rol** | 1 caso (`ScopeCarteraClientesPostgresTest`, PG real) | ✅ inicial |
| Reportes / documentos | jrxml compila + 1 reporte con scope (cartera clientes) | 🟡 inicial |
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
- ✅ **Mora en días hábiles** (HALL-01 corregido): PORCENTAJE y FIJO_DIARIO_HABILES; caja y
  cobranza unificadas; solo domingo no laborable.
- ⏳ Restan: mora `FIJO`, mora desde fecha fin, fechas del cronograma en día hábil.
- ✅ **Tasa del comité aplicada** en SIMPLE (HALL-11 corregido): manda la tasa aprobada por el comité.

### ✅ Fase 2 — Seguridad / Scope *(DESBLOQUEADA e iniciada, 2026-06-12)*
- ✅ **Testcontainers + PostgreSQL real** montado (`AbstractPostgresIntegrationTest`, perfil
  `pgtest`, patrón singleton). Corre el SQL nativo PG que H2 no soportaba.
- ✅ `computarScope` validado en `carteraClientes`: analista→sus clientes, gerente→su agencia,
  admin→global (`ScopeCarteraClientesPostgresTest`).
- ⏳ Restan: scope en otros reportes (movimientos, cuadre, cartera mora) y por cajero.
- ⚠️ Requiere **Docker corriendo**. `AppApplicationTests` depende de la BD dev real (frágil en CI).

### 🟨 Fase 4 — Reportes / Documentos *(desbloqueada, infra lista)*
- Con Testcontainers ya disponible: cartera, cuadre, cartera mora (montos + scope); Word/PDF.

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
| 2026-06-12 | Migración HALL-12 (script + test). 32 tests. |
| 2026-06-12 | Fase 2 desbloqueada: **Testcontainers + PostgreSQL** montado; scope de cartera validado (RN-ROL). |
| 2026-06-12 | Bloque extornos completo (extorno de pago D5, no doble extorno D7) + trazabilidad D2. |
| 2026-06-12 | Extornos completos (D5/D7) + trazabilidad D2 → 7 invariantes del dinero cubiertas. |
| 2026-06-12 | **HALL-01 corregido**: mora en días hábiles (solo domingo no laborable); caja=cobranza. |
| 2026-06-12 | **40 tests backend + 4 E2E en verde.** Pendiente decisión: HALL-11 (tasa SIMPLE). |

---

*Bitácora viva: actualizar al agregar/cambiar pruebas y al cerrar invariantes.*
