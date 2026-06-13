# RN-APRO · Aprobación y Comité

> Tras la evaluación, el crédito va al **comité**: los miembros votan, se define la **resolución
> final** (producto, monto, tasa, plazo) y, si se aprueba, se crea automáticamente el préstamo en
> `PRE_DESEMBOLSO` con su cronograma preliminar.
>
> Fuente en código: `model/AprobacionCredito.java`, `service/AprobacionCreditoServiceImpl.java`,
> **`utils/PrestamoCalculator.java`** (motor del cronograma persistido).

---

## 1. Propósito

Gestionar la decisión colegiada del crédito y, al aprobar, instanciar el préstamo con las
condiciones finales y su cronograma.

---

## 2. Diagrama — Estados de la aprobación

```mermaid
stateDiagram-v2
    [*] --> PENDIENTE: crear (evaluación → EN_COMITE)
    PENDIENTE --> PENDIENTE: miembros votan (participar)
    PENDIENTE --> APROBADO: resolución (producto+monto+tasa+plazo)
    PENDIENTE --> RECHAZADO: resolución (+ motivo)
    PENDIENTE --> DEVUELTO: devuelve al analista (+ qué subsanar)
    DEVUELTO --> PENDIENTE: analista corrige y reenvía
    APROBADO --> [*]: préstamo creado en PRE_DESEMBOLSO
```

> Estados reales (`estadoAprobacion`): `PENDIENTE`, `APROBADO`, `RECHAZADO`, `DEVUELTO`.

---

## 3. Reglas — Comité y resolución

| ID | Regla | Fuente |
|---|---|---|
| **RN-APRO-01** | Crear aprobación requiere `presidenteId`; pasa la evaluación a `EN_COMITE` y la aprobación a `PENDIENTE` | `crear()` |
| **RN-APRO-02** | Cada miembro registra **voto + checklist + comentario** (`participarMiembro`) | `:173-208` |
| **RN-APRO-03** | Enviar a comité: `ADMIN`, `GERENTE_AGENCIA`, `SUPERVISOR`, `ANALISTA`; decidir: `ADMIN`, `COMITE`, `ANALISTA` | `@PreAuthorize` |
| **RN-APRO-04** | Una aprobación ya `APROBADO`/`RECHAZADO` **no puede editarse** | `actualizar()` :240 |
| **RN-APRO-05** 💰 | Para **APROBAR** se exige producto, monto, tasa y plazo **finales** (si falta → `IllegalArgumentException`) | `actualizar()` :252 |
| **RN-APRO-05b** 💰 | La **tasa aprobada manda** (decisión final del comité) pero **debe respetar el rango del producto** (`tasaMin`/`tasaMax`); fuera de rango → rechazada (HALL-11) | `actualizar()` |
| **RN-APRO-06** | Para **RECHAZAR/DEVOLVER** se exige comentario (motivo / qué subsanar) | `actualizar()` :260 |
| **RN-APRO-07** | El presidente (sección 5) ajusta monto y tasa, **no cambia el producto** acordado por el comité | `crearPrestamoDesdeAprobacion` comentario |

> ℹ️ **Quórum / desempate:** los votos se registran por miembro, pero la resolución
> (`actualizar`) **no valida quórum** en el código actual — la decisión final se aplica con los
> campos finales. (La idea de "mínimo quórum" de la doc previa no está implementada.)

---

## 4. Reglas — Creación del préstamo (al aprobar)

| ID | Regla | Fuente |
|---|---|---|
| **RN-APRO-08** | Al aprobar se crea el préstamo en `PRE_DESEMBOLSO`, **idempotente** (no duplica si ya existe) | `:323-328` |
| **RN-APRO-09** | Copia condiciones finales: `montoDesembolsado`, `plazoMeses`, producto, y **snapshot** de gracia y regla de mora del producto | `crearPrestamoDesdeAprobacion` |
| **RN-APRO-10** | `saldoCapital = montoFinalAprobado`; `fechaPrimerVencimiento = hoy + 1 mes` (ajustable); genera **N° de contrato** | `:427-431` |
| **RN-APRO-11** | Genera el **cronograma preliminar** con `PrestamoCalculator` según `sistemaCalculo` | `:438` |

---

## 5. ⚠️ Hallazgos detectados

### HALL-11 — La tasa aprobada por el comité no se aplicaba en productos SIMPLE  ✅ CORREGIDO
- **Severidad:** 🔴 Alta (dinero) · **Estado:** ✅ Corregido (2026-06-12)
- **Decisión:** manda el comité (la tasa está viva; se usa su último valor aprobado), **pero debe
  respetar el rango del producto** (`tasaMin`/`tasaMax`).
- Antes: `tasaInteresPeriodo = producto.getTasaInteres()` → los productos SIMPLE leían ese campo e
  ignoraban la tasa aprobada (FRANCES/ALEMAN sí la usaban vía `tasaInteresAnual`).
- **Fix:** (1) `tasaInteresPeriodo = tasaFinalAprobada` → todos los tipos usan la tasa del comité;
  (2) la aprobación **rechaza** una tasa fuera de `[tasaMin, tasaMax]`. Validado por
  `TasaAprobadaCronogramaTest` (usa la del comité dentro de rango; rechaza fuera de rango).

### HALL-09 (ampliado) — Hay **tres** motores de cálculo de cuotas
- `PrestamoCalculator` (cronograma persistido, +ALEMAN) · `ProductoCalculator` (simulación) ·
  `CalculoCuotasServiceImpl` (legacy, enum incompatible). Los dos primeros **duplican** la lógica
  FLAT/SALDO/FRANCES; el tercero es huérfano. → Riesgo de divergencia y mantenimiento.

---

## 6. Casos borde / negativos

| Caso | Resultado |
|---|---|
| Aprobar sin producto/monto/tasa/plazo final | `IllegalArgumentException` (RN-APRO-05) |
| Rechazar/devolver sin comentario | `IllegalArgumentException` (RN-APRO-06) |
| Editar una aprobación ya resuelta | `IllegalStateException` (RN-APRO-04) |
| Aprobar dos veces | no duplica el préstamo (RN-APRO-08) |

---

## 7. Trazabilidad (regla → prueba)

| Regla | Prueba | Estado |
|---|---|---|
| RN-APRO-05 (aprobar exige finales) | `FlujoNegativoTest.aprobar_sinProductoFinal…` | ✅ |
| RN-APRO-08..11 (crea préstamo + cronograma) | `FlujoPrestamoIntegrationTest` | ✅ |
| RN-APRO-03 (permisos comité) | `RbacIntegrationTest` (parcial) | 🟡 |
| HALL-11 (tasa del comité aplicada en SIMPLE) | `TasaAprobadaCronogramaTest.productoSimple_usaLaTasaAprobadaPorElComite` | ✅ |

---

## Changelog
- **2026-06-12** — Documento nuevo desde el código: estados de aprobación, reglas RN-APRO-01..11,
  creación idempotente del préstamo y cronograma con `PrestamoCalculator`. Detecta **HALL-11**
  (tasa aprobada ignorada en productos SIMPLE) y amplía **HALL-09** (tres motores de cálculo).
  Aclara que el **quórum no está implementado** (corrige la doc previa).
