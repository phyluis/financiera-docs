# Reglas de Negocio — Índice Maestro

> **La fuente de verdad del comportamiento del sistema.** Cada regla se documenta desde el
> código real (no de memoria), con su **diagrama de flujo**, sus **casos borde**, y una
> **prueba que la verifica**. Las pruebas son el *producto* de estas reglas, no al revés.

## Cadena de trazabilidad

```
REGLA DE NEGOCIO  ──►  DIAGRAMA DE FLUJO  ──►  CASO DE PRUEBA  ──►  CÓDIGO ALINEADO
   (qué debe pasar)      (cómo fluye)          (lo verifica)        (lo cumple)
```

Cada regla tiene un **ID** (`RN-XXX-NN`). Cada prueba referencia ese ID en su Javadoc, de modo
que se puede rastrear *"¿qué regla cubre este test?"* y *"¿qué test prueba esta regla?"*.

> 🔎 **Todo bug, deriva o decisión pendiente que aparezca al documentar se registra en
> [`../HALLAZGOS.md`](../HALLAZGOS.md)** — se anota ahora y se revisa/resuelve al final.

---

## Metodología — anatomía de un documento de regla

Cada archivo `reglas-negocio/<modulo>.md` sigue esta plantilla:

1. **Propósito** — qué resuelve el módulo en una frase.
2. **Diagrama de flujo** — Mermaid (`flowchart` o `stateDiagram`) del proceso/estados.
3. **Reglas** — tabla `ID | Regla | Detalle | Fuente en código`.
4. **Casos borde / negativos** — qué NO se permite y qué error se espera.
5. **Invariantes de dinero** (si aplica) — referencia a D1–D7 del plan de pruebas.
6. **Trazabilidad** — tabla `ID regla → prueba que la verifica → estado`.

> Las reglas se extraen **leyendo el código** (servicios, validadores, entidades). Si el código
> y la doc previa difieren, **gana el código** y se anota la corrección en el changelog del doc.

---

## Esquema de IDs por dominio

| Prefijo | Dominio | Archivo destino |
|---|---|---|
| `RN-FLU` | Flujo general del préstamo (estados) | `flujo-prestamo.md` |
| `RN-EVAL` | Evaluación de crédito | `evaluacion.md` |
| `RN-APRO` | Aprobación / comité | `aprobacion-comite.md` |
| `RN-DES` | Desembolso | `desembolso.md` |
| `RN-CRON` | Cálculo de cuotas / cronograma | `calculo-cuotas.md` |
| `RN-MORA` | Mora y feriados (días hábiles/calendario) | `mora-feriados.md` |
| `RN-PAGO` | Pago de cuotas / cobranza | `cobranza-pagos.md` |
| `RN-CAJA` | Caja: apertura, cierre, cuadre | `caja.md` |
| `RN-MOV` | Movimientos: ingresos / egresos | `movimientos-caja.md` |
| `RN-EXT` | Extornos (reversas) | `extornos.md` |
| `RN-BOV` | Bóveda / tesorería | `boveda.md` |
| `RN-GAR` | Garantías y garantes | `garantias-garantes.md` |
| `RN-CART` | Cartera | `cartera.md` |
| `RN-ROL` | Roles, permisos y scope | `roles-permisos.md` |
| `RN-META` | Metas de colocación | `metas-colocacion.md` |

---

## Inventario y estado (roadmap de documentación)

> Prioridad alineada con el riesgo: **dinero y flujo primero**.

| # | Dominio | Doc | Diagrama | Reglas extraídas del código | Prueba enlazada | Prioridad |
|---|---|---|---|---|---|---|
| 1 | Flujo del préstamo | ✅ `flujo-prestamo.md` | ✅ | ✅ + hallazgo mora | ✅ (flujo, negativos, pago) | ✅ doc lista |
| 2 | Caja (apertura/cierre/cuadre) | ✅ `caja.md` | ✅ | ✅ | ✅ `CajaCierreTest` (5) | ✅ doc lista |
| 3 | Movimientos (ingresos/egresos) | ✅ `movimientos-caja.md` | ✅ | ✅ HALL-06/07 **corregidos** | ✅ `DineroConservacionTest` + `MovimientoAtomicoTest` | ✅ doc lista |
| 4 | Pago de cuotas / cobranza | ❌ doc | ❌ | ⚠️ HALL-12 **corregido** | ✅ `PagoLiquidacionTest` + `PagosIntegrationTest` | 🟧 falta doc |
| 5 | Mora y feriados | ❌ | ❌ | ⚠️ deriva (hábiles vs calendario, HALL-01) | 🟡 1 caso (mora %) | 🔴 P1 |
| 6 | Extornos | ✅ `extornos.md` | ✅ | ✅ HALL-08 **corregido** | 🟡 desembolso ✅, pago ❌ | ✅ doc lista |
| 7 | Cálculo de cuotas (FLAT/SALDO/FRANCES) | ✅ `calculo-cuotas.md` | ✅ | ✅ HALL-09 **resuelto** | ✅ `CronogramaCalculoTest` (5) | ✅ doc lista |
| 8 | Evaluación de crédito | ✅ `evaluacion.md` | ✅ | ✅ + HALL-10 | 🟡 parcial | ✅ doc lista |
| 9 | Aprobación / comité | ✅ `aprobacion-comite.md` | ✅ | ✅ + HALL-11 | 🟡 parcial | ✅ doc lista |
| 10 | Desembolso | ✅ `desembolso.md` | ✅ | ✅ | 🟡 parcial | ✅ doc lista |
| 11 | Roles, permisos y scope | ✅ `roles-permisos.md` | ✅ | ✅ corregida | ✅ RBAC + scope (PG real) | ✅ doc lista |
| 12 | Garantías y garantes | ❌ | ❌ | ❌ | ❌ | 🟨 P3 |
| 13 | Bóveda / tesorería | ❌ | ❌ | ❌ | ❌ | 🟨 P3 |
| 14 | Cartera | ❌ | ❌ | ❌ | ❌ | 🟨 P3 |
| 15 | Metas de colocación | ❌ | ❌ | ❌ | ❌ | 🟦 P4 |

Leyenda: ✅ completo · 🟡 parcial · ⚠️ con deriva detectada · ❌ pendiente

---

## Orden de ejecución recomendado

1. **Préstamo + dinero** (1–6): el corazón. Documentar regla + diagrama, luego escribir la prueba.
2. **Cálculo y crédito** (7–11): cuotas exactas, evaluación, aprobación, desembolso, scope.
3. **Soporte** (12–15): garantías, bóveda, cartera, metas.

Cada documento que se cierre actualiza esta tabla y la
[`bitacora-pruebas.md`](../desarrollo/bitacora-pruebas.md).

---

## Derivas conocidas (a resolver al documentar)

| Doc | Deriva detectada | Acción |
|---|---|---|
| ~~`flujo-prestamo.md`~~ | mora "días hábiles" vs caja "días calendario" | ✅ **documentado 2026-06-12** como `RN-MORA-05` (hallazgo en `MoraCalculator`); 🔴 **falta decisión de negocio** para unificar |
| ~~`roles-permisos.md`~~ | ~~"cada usuario solo ve su agencia" — inexacto~~ | ✅ **resuelto 2026-06-12**: scope real documentado |

---

*Índice vivo: se actualiza al documentar cada regla. Última actualización: 2026-06-12.*
