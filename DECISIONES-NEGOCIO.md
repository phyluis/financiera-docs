# Decisiones de Negocio Pendientes

> Al mapear las reglas de negocio detectamos **dos puntos que el sistema resuelve de forma
> ambigua o inconsistente** y que **no se pueden corregir sin una definición de negocio**. Ambos
> tocan el **dinero que paga el cliente**. Este documento las resume para decidir.
>
> Relacionado: [`HALLAZGOS.md`](./HALLAZGOS.md) (detalle técnico) ·
> [`reglas-negocio/`](./reglas-negocio/) (reglas documentadas).

---

## Decisión 1 — Mora: días hábiles o calendario  (HALL-01) — ✅ RESUELTO (2026-06-12)

> **Definido por el cliente:** la mora se cuenta en **días HÁBILES**; día no laborable por defecto =
> **solo el DOMINGO** (el sábado SÍ cuenta); los feriados se restan aparte. **Fix aplicado** y
> validado (`MoraDiasHabilesTest`): caja y cobranza ahora coinciden. *(El resto de esta sección
> queda como registro histórico de la decisión.)*

### (Histórico) Contexto que se evaluó

### Qué pasa hoy
El sistema calcula la mora de **dos maneras distintas** según dónde se mire:

| Dónde | Cómo cuenta los días de atraso |
|---|---|
| Pantalla de **cobranza** (listados) | días **hábiles** (excluye sábados, domingos y feriados) |
| **Cobro en caja** (cuando el cliente paga) | días **calendario** (todos los días) |

### Por qué importa
Una **misma cuota vencida puede mostrar una mora en la pantalla de cobranza y cobrar otra
distinta en la caja**. Para mora porcentual, la caja suele cobrar **más** (calendario ≥ hábiles).
Es un riesgo de **transparencia y consistencia** frente al cliente.

### Opciones
| Opción | Implica |
|---|---|
| **A. Días calendario** en todo | Más simple y habitual en microfinanzas; cobra desde el día siguiente al vencimiento, incluidos fines de semana. |
| **B. Días hábiles** en todo | Más favorable al cliente; aprovecha el calendario de feriados que ya existe (`FeriadoService`). |

### Recomendación técnica
Cualquiera de las dos es válida; lo importante es **unificar**. Sugerencia: definir la regla a
nivel de **producto** (algunos productos podrían usar una y otros otra), pero como mínimo que
**cobranza y caja coincidan**.

### Qué sigue tras decidir
Unificar `MoraCalculator` (un solo criterio) y blindar con una prueba que compare cobranza vs caja
sobre la misma cuota (debe dar igual).

---

## Decisión 2 — Tasa de interés: ¿la fija el **producto** o el **comité**?  (HALL-11)

### Qué pasa hoy
El comité, al aprobar un crédito, puede definir una **tasa final aprobada**. Pero en los productos
de tipo **SIMPLE** (los de microcrédito), el cronograma **ignora esa tasa** y usa la **tasa del
producto**. (En productos FRANCÉS/ALEMÁN sí se respeta la tasa del comité.)

> Confirmado con una prueba: producto al 5%, comité aprueba 10% → el cliente paga con **5%**.

### Por qué importa
Si el comité aprueba una tasa distinta a la del producto, en créditos SIMPLE **el cliente paga con
una tasa diferente a la que el comité autorizó**. Como el microcrédito suele ser SIMPLE, el
impacto es directo sobre lo que cobra la financiera.

### Opciones
| Opción | Implica |
|---|---|
| **A. Manda la tasa del comité** | El cronograma usa `tasaFinalAprobada` también en SIMPLE. **Fix de 1 línea**; la prueba ya está lista para validarlo. |
| **B. La tasa la fija el producto** | El campo "tasa final" del comité **no aplica** en SIMPLE → conviene **ocultarlo/bloquearlo** en la pantalla de aprobación para no confundir. |

### Recomendación técnica
Si el comité puede ajustar la tasa (el formulario se la pide), lo natural es la **Opción A**: que
la tasa aprobada mande. Pero es **decisión de negocio** confirmar si en SIMPLE la tasa debe ser
fija por producto o ajustable por comité.

### Qué sigue tras decidir
- Si **A**: asignar `tasaInteresPeriodo = tasaFinalAprobada` en productos SIMPLE; la prueba
  `TasaAprobadaCronogramaTest` se ajusta para esperar la tasa del comité.
- Si **B**: ajustar la UI de aprobación para que en SIMPLE no se edite la tasa.

---

## Resumen para la reunión

| # | Decisión | Pregunta para negocio | Esfuerzo del fix |
|---|---|---|---|
| 1 | Mora | ¿Días hábiles o calendario? (y ¿por producto?) | Medio (unificar `MoraCalculator`) |
| 2 | Tasa en SIMPLE | ¿La fija el comité o el producto? | Bajo (1 línea o ajuste de UI) |

> Los **bugs de dinero claros ya fueron corregidos** (HALL-06 cargo descontado, HALL-07
> atomicidad, HALL-08 extorno↔caja, HALL-12 pagado→LIQUIDADO) y la calculadora legacy eliminada
> (HALL-09). Estas **dos decisiones** son lo único que queda del repaso que depende de negocio.
>
> ℹ️ Nota operativa (HALL-12): los préstamos pagados **antes** del fix quedaron como `CANCELADO`
> en la BD. El script de migración ya está listo y validado con prueba
> (`financiera-backend/src/main/resources/db/migration-2026-06-hall12-liquidado.sql`) — **correr
> en dev/qa antes de producción**.

*Documento de decisión — 2026-06-12.*
