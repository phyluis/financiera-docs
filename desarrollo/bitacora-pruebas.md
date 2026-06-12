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
| `DineroConservacionTest` | `desembolsoDescontado_inflaSaldoTeoricoEnElCargo` | cargo DESCONTADO → teórico inflado en `cargo` (**confirma HALL-06**) | **Dinero D1** | ✅ |
| `DineroConservacionTest` | `extornoDesembolso_neutralizaCaja` | extornar desembolso → préstamo revierte **y** EGRESO se neutraliza (**HALL-08 corregido**) | **Dinero D5** | ✅ |
| `TasaAprobadaCronogramaTest` | `productoSimple_ignoraLaTasaAprobada_usaLaDelProducto` | SIMPLE: cronograma usa tasa del producto, no la aprobada (**confirma HALL-11**) | **Dinero D3** | ✅ |
| `MovimientoAtomicoTest` | `desembolso_siFallaElMovimientoDeCaja_revierteTodo` | si falla el movimiento → desembolso revierte (**HALL-07 corregido**) | **Dinero D1/D2** | ✅ |

**Total: 18 pruebas en verde.**

> Frontend (Karma/Jasmine): 9 smoke tests `should create` (`npm run test:ci`).

---

## 2. Cobertura de las invariantes del DINERO (D1–D7)

| Invariante | Descripción | Estado | Prueba |
|---|---|---|---|
| **D1** | Conservación del saldo | 🟡 parcial | `DineroConservacionTest` (desembolso EFECTIVO ✅ / DESCONTADO destapa HALL-06) |
| **D2** | Trazabilidad (todo movimiento registrado) | ⏳ pendiente | — |
| **D3** | Exactitud de montos | 🟡 parcial | `pagoCuotaVencida_cobraMora` |
| **D4** | Cuadre cuadrado/descuadrado | ⏳ pendiente | — |
| **D5** | Extorno reversible | 🟡 parcial | `DineroConservacionTest` (extorno desembolso destapa HALL-08) |
| **D6** | No hay dinero sin caja | ✅ | `desembolsar_sinCajaAbierta…` |
| **D7** | Sin doble cobro | ⏳ pendiente | — |

> 🔴 **Brecha crítica:** lo más delicado (cuadre, extornos, conservación, trazabilidad) **aún sin cubrir**.

---

## 3. Cobertura por escenario (matriz)

| Área | Cobertura | Estado |
|---|---|---|
| Flujo feliz completo | 1 caso | ✅ |
| RBAC (desembolso) | 5 casos | ✅ |
| Validaciones/rechazos | 2 casos | 🟡 inicial |
| Pagos (parcial, mora) | 2 casos | 🟡 inicial |
| **Caja: cuadre / extornos / movimientos** | 0 | 🔴 pendiente |
| **Scope por agencia/rol** | 0 | 🔴 pendiente |
| Cálculo por producto (FLAT/SALDO/FRANCES) | 0 | 🔴 pendiente |
| Reportes / documentos | jrxml compila | 🟡 mínimo |

---

## 4. ✅ Plan recomendado (roadmap por fases)

> Orden por **criticidad** (dinero y seguridad primero). Cada fase: tests + regla documentada + verde en `./mvnw test`.

### Fase 0 — Infraestructura + POC *(HECHA)*
H2 aislado, flujo completo, RBAC base, negativos, pagos básicos. → **13 tests verdes.**

### 🔴 Fase 1 — DINERO (máxima prioridad)
**Objetivo:** cerrar las invariantes D1–D7.
- Apertura de caja → saldo inicial.
- Desembolso EFECTIVO → EGRESO exacto; saldo baja (D1, D3).
- Cargo DESCONTADO / EFECTIVO → INGRESO `CARGO_DESEMBOLSO` (D3).
- Pago al día → INGRESO `COBRO_CUOTA`; vencido → `+ COBRO_MORA` (D2, D3).
- **Cuadre cuadrado** (diferencia 0) y **descuadrado** (detecta/bloquea) (D4).
- **Extorno de pago** y **de desembolso** → revierten exacto (D5).
- **Conservación**: tras N operaciones, `teórico = inicial + Σing − Σegr` (D1).
- **Sin doble cobro**: re-cobrar cuota pagada / re-extornar (D7).
- *Doc:* asegurar `flujo-prestamo.md` (mora días hábiles vs calendario) y conceptos de caja.

### 🔴 Fase 2 — Seguridad / Scope (P1)
**Objetivo:** que nadie vea datos que no le tocan.
- `computarScope` por rol (gerente agencia ≠ otra agencia; analista solo sus clientes; cajero solo suyos; admin/ger.gen/auditor global).
- Cartera de clientes filtrada por scope.
- RBAC ampliado (evaluación, pagos, caja).
- *Doc:* actualizar `roles-permisos.md` con la tabla real de `computarScope`.

### 🟧 Fase 3 — Cálculo / Productos (P2)
- Cuotas exactas: SIMPLE_FLAT, SIMPLE_SALDO, FRANCES (montos al centavo).
- Redondeo a 0.10; período de gracia (PARCIAL/TOTAL).
- Mora: PORCENTAJE / FIJO / FIJO_DIARIO_HABILES + **mora desde fecha fin** (diario).

### 🟨 Fase 4 — Reportes / Documentos (P3)
- Cartera de clientes, cuadre de caja, cartera mora (montos + scope).
- Generación Word/PDF (contrato, pagaré, acta, cronograma) sin error.
- *Nota:* reportes con SQL nativo → evaluar Testcontainers + PostgreSQL si H2 no basta.

### 🟦 Fase 5 — E2E de UI (opcional)
- Playwright: login + flujo en navegador (smoke de extremo a extremo).

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
| _(pendiente)_ | Fase 1 — DINERO (cuadre, extornos, conservación). |

---

*Bitácora viva: actualizar al agregar/cambiar pruebas y al cerrar invariantes.*
