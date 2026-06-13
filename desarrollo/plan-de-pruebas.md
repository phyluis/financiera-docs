# Plan de Pruebas — Sistema Financiero (CrediActiva / white-label)

> Cómo probamos el sistema de forma sistemática. La prioridad #1 es el **dinero**
> (caja: ingresos, egresos, cuadre, extornos): ahí no puede faltar ni sobrar un centavo.
>
> Este plan acompaña al **POC ejecutable** ya implementado en
> `financiera-backend/src/test/java/...` (ver §7). La idea: **regla de negocio → caso de
> prueba que la verifica → código que la cumple**, los tres sincronizados.

---

## 1. Objetivo y alcance

- **Objetivo:** garantizar que el flujo crítico funciona y que **el dinero siempre cuadra**,
  evitando fugas, montos errados, doble cobro o pérdidas entre agencias.
- **Alcance (en orden de criticidad):**
  1. **Caja / dinero** (ingresos, egresos, cuadre, extornos) — *crítico*
  2. **Flujo del préstamo** (evaluación → aprobación → desembolso → cobranza)
  3. **Permisos y visibilidad** (RBAC + scope por agencia/rol)
  4. **Cálculo de cuotas y mora** (productos)
  5. Reportes y documentos
- **Fuera de alcance (por ahora):** pruebas de carga/performance.
- **E2E (Playwright):** ya montado en el frontend (smoke + login + dashboard) — ver §2.

---

## 2. Estrategia y niveles de prueba

| Nivel | Qué prueba | Herramienta | Ambiente |
|---|---|---|---|
| **Unitario** | lógica pura (cálculo de cuota, mora, redondeo) | JUnit 5 / AssertJ | sin contexto |
| **Integración (servicios)** | flujo real a través de servicios + JPA | `@SpringBootTest` | **H2 en memoria** (aislado) |
| **Integración (HTTP/seguridad)** | endpoints + `@PreAuthorize` (RBAC) | `@SpringBootTest` + **MockMvc** + JWT real | **H2 en memoria** |
| **Integración (SQL nativo / scope)** | reportes y `computarScope` con SQL nativo PG | `@SpringBootTest` + **Testcontainers** | **PostgreSQL real** (Docker) |
| **Frontend (componente)** | `should create` + lógica de componentes | Karma + Jasmine | navegador headless |
| **E2E (UI)** | login real + navegación autenticada por la app | **Playwright** (`npm run e2e`) | `ng serve` + backend + BD de pruebas |

**Aislamiento:** las pruebas de backend usan **H2 en memoria** (perfil `test`,
`application-test.properties`, `ddl-auto=create-drop`). **No tocan la BD real.**
Tests con estado mutable usan `@Transactional` (rollback por método).

> ✅ Algunos reportes y el **scope** (`computarScope`) usan **SQL nativo PostgreSQL** que H2 no
> corre. Para esos casos hay **Testcontainers + PostgreSQL real** (perfil `pgtest`, clase base
> `AbstractPostgresIntegrationTest`, patrón singleton). **Requiere Docker corriendo.**

---

## 3. Anatomía de un caso de prueba (cómo se arma)

Cada caso sigue **Given–When–Then**:

```
GIVEN  (preparación)  — sembrar datos: agencia, usuarios por rol, producto, cliente…
WHEN   (acción)       — login como el rol + llamar al servicio/endpoint
THEN   (verificación) — assert de estado, montos, movimientos de caja, o excepción
```

Buenas prácticas:
- **Un escenario por test** (nombre descriptivo: `pagoCuotaVencida_cobraMora`).
- **Login simulado por rol** (helper `loginComo(usuario)` que setea el `SecurityContext`).
- **Datos mínimos** y únicos (evitar choques de claves únicas entre tests).
- Verificar **el efecto observable**: estado persistido, monto, movimiento de caja, o el error esperado.

---

## 4. 💰 Reglas de oro del DINERO (invariantes que SIEMPRE deben cumplirse)

Estas son **no negociables** — cada una debe tener su(s) prueba(s):

| # | Invariante | Por qué importa |
|---|---|---|
| **D1** | **Conservación:** `saldoTeórico = montoInicial + Σingresos − Σegresos`. Ningún movimiento crea ni destruye dinero. | Evita fugas/sobrantes |
| **D2** | **Trazabilidad:** toda operación de dinero genera **un movimiento** con `tipo`+`concepto`+`monto` correctos | Auditoría |
| **D3** | **Exactitud de montos:** desembolso EGRESO = neto entregado; cargo = INGRESO exacto; pago = `COBRO_CUOTA` (capital+interés) + `COBRO_MORA` | Cero centavos de error |
| **D4** | **Cuadre:** al cerrar, `diferencia = contado − teórico`; si se exige exacto, diferencia ≠ 0 **bloquea** el cierre | Detecta faltantes/sobrantes |
| **D5** | **Reversibilidad (extorno):** un extorno **revierte exactamente** el movimiento; el saldo vuelve al estado previo | Sin doble efecto |
| **D6** | **No hay dinero sin caja:** desembolsar/cobrar **requiere caja abierta** del día | Control operativo |
| **D7** | **Sin doble cobro:** una cuota PAGADA no se vuelve a cobrar; un extorno no se aplica dos veces | Integridad |

---

## 5. Matriz de escenarios (priorizada)

### 🟥 P1 — CAJA / DINERO (crítico)
| Caso | Verifica |
|---|---|
| Apertura de caja | saldo inicial = `montoInicial` |
| Desembolso EFECTIVO | EGRESO `DESEMBOLSO_PRESTAMO` = monto; saldo baja exacto (D1, D3) |
| Cargo DESCONTADO | EGRESO = neto + INGRESO `CARGO_DESEMBOLSO` = cargo (D3) |
| Cargo EFECTIVO | EGRESO = bruto + INGRESO `CARGO_DESEMBOLSO` aparte (D3) |
| Pago de cuota al día | INGRESO `COBRO_CUOTA` = capital+interés (D2, D3) |
| Pago de cuota vencida | INGRESO `COBRO_CUOTA` + `COBRO_MORA` (mora > 0) (D3) |
| Pago parcial | cuota → `PARCIAL`; INGRESO por lo entregado |
| Cierre cuadrado | contado = teórico → diferencia 0 (D4) |
| Cierre descuadrado | contado ≠ teórico → diferencia detectada / bloqueo (D4) |
| Extorno de pago | revierte INGRESO; saldo y cuota vuelven (D5, D7) |
| Extorno de desembolso | revierte EGRESO; préstamo vuelve a PRE_DESEMBOLSO (D5) |
| Movimiento manual ingreso/egreso | registra el concepto correcto; afecta saldo (D2) |
| Desembolsar/cobrar sin caja abierta | **rechazado** (D6) |

### 🟥 P1 — Flujo del préstamo
Evaluación → aprobación (comité) → préstamo `PRE_DESEMBOLSO` + cronograma → desembolso `VIGENTE` → pago → `LIQUIDADO`. Rechazos y devoluciones (máx 2 → 3ra cancela).

### 🟥 P1 — Permisos y visibilidad (seguridad de datos)
RBAC por endpoint (403 vs permitido) + **scope por agencia/rol** (`computarScope`): gerente de agencia **no ve otra agencia**; analista solo **sus** clientes; cajero solo **sus** registros; gerente general/admin/auditor global.

### 🟧 P2 — Cálculo de cuotas y mora
Cuotas exactas por producto (SIMPLE_FLAT / SIMPLE_SALDO / FRANCES), redondeo a 0.10, mora (PORCENTAJE/FIJO/FIJO_DIARIO_HABILES) y **mora desde fecha fin** (diario).

### 🟨 P3 — Reportes y documentos
Cartera de clientes, cuadre, cartera mora; generación Word/PDF (contrato, pagaré, acta, cronograma).

---

## 6. Estado actual (qué ya está cubierto)

> El inventario detallado y al día vive en la
> [`bitacora-pruebas.md`](./bitacora-pruebas.md) (§1 inventario, §2 invariantes, §6 historial).

Resumen al 2026-06-12: **31 tests backend + 4 E2E en verde**. Invariantes del dinero:
D1 ✅ · D2 🟡 · D3 ✅ · D4 ✅ · D5 🟡 · D6 ✅ · D7 ⏳. La ejecución de este plan destapó
**12 hallazgos** (ver [`../HALLAZGOS.md`](../HALLAZGOS.md)), de los cuales **5 bugs ya fueron
corregidos con prueba de regresión** (HALL-06/07/08/09/12).

---

## 7. Backlog priorizado (próximos lotes)

1. ~~**Lote DINERO**~~ ✅ hecho (cuadre, conservación, atomicidad, extorno de desembolso).
2. **Lote Scope** (P1 seguridad): visibilidad por agencia/rol — **requiere Testcontainers +
   PostgreSQL** (el `computarScope` vive en SQL nativo PG, no corre en H2).
3. ~~**Lote Cálculo**~~ ✅ hecho en lo esencial (FLAT/SALDO/FRANCES/gracia/redondeo). Restan
   variantes de mora (FIJO, FIJO_DIARIO_HABILES, desde fecha fin).
4. **Lote Extornos restante**: extorno de pago (D5), no-doble-extorno (D7).
5. **Lote Reportes/Docs** (P3, con Testcontainers).
6. **Lote E2E con mutación**: flujos por rol contra BD de pruebas desechable.

---

## 8. Criterios de aceptación (Definition of Done)

Un cambio que toque **dinero** no se mergea sin:
- ✅ Pruebas verdes de las invariantes D1–D7 afectadas.
- ✅ La **regla de negocio documentada** (`reglas-negocio/`) actualizada y alineada con el código.
- ✅ Cero diferencias de centavos en los asserts de montos.
- ✅ `./mvnw test` completo en verde.

---

*Plan vivo: se actualiza al agregar cada lote de pruebas. El POC ejecutable es la prueba de que la estrategia funciona.*
