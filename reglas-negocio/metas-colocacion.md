# RN-META · Metas de Colocación

> Las metas definen el **objetivo de colocación** (monto, cantidad y clientes nuevos) por
> **agencia/analista** en un **mes/año**. Sirven de referencia para medir el desempeño.
>
> Fuente en código: `model/MetaColocacion.java`, `service/MetaColocacionServiceImpl.java`,
> `controller/MetaColocacionController.java`.

---

## 1. Propósito

Configurar y consultar las metas de colocación por agencia y por analista, por período (mes/año).

---

## 2. Modelo

Cada `MetaColocacion` tiene: `agenciaId`, `analistaId` (**null = meta de la agencia**, no de un
analista puntual), `mes`, `anio`, `metaMonto`, `metaCantidad`, `metaClientesNuevos`.

> **Unicidad:** una sola meta por la combinación `(agencia, analista, mes, año)`
> (`@UniqueConstraint`).

---

## 3. Reglas

| ID | Regla | Fuente |
|---|---|---|
| **RN-META-01** | Una meta por `(agencia, analista, mes, año)` — no se duplica | `@UniqueConstraint` |
| **RN-META-02** | **Guardar es UPSERT**: re-guardar la misma clave **actualiza** la meta existente (no crea otra) | `guardar()` → `findUpsert` |
| **RN-META-03** | `analistaId = null` → meta a nivel de **agencia** (no por analista) | `MetaColocacion` |
| **RN-META-04** | Consultas: por año, por agencia+año, y del **mes actual** | `listar*()` |

---

## 4. Casos borde / negativos

| Caso | Resultado |
|---|---|
| Guardar dos veces la misma (agencia, analista, mes, año) | actualiza la misma meta (mismo `metaId`) |
| Distinta agencia / analista / mes / año | meta independiente |

---

## 5. Trazabilidad (regla → prueba)

| Regla | Prueba | Estado |
|---|---|---|
| RN-META (crear) | `MetaColocacionTest.guardar_creaMeta` | ✅ |
| RN-META-01/02 (upsert, no duplica) | `MetaColocacionTest.guardar_esUpsert_actualizaSinDuplicar` | ✅ |
| RN-META-04 (filtro por agencia/año) | `MetaColocacionTest.listarPorAgenciaYAnio_filtraPorAgencia` | ✅ |

---

## Changelog
- **2026-06-13** — Documento nuevo desde el código: modelo de meta (unicidad por agencia/analista/
  mes/año, `analistaId` null = agencia), regla de **upsert** al guardar y consultas. Cubierto por
  `MetaColocacionTest`. **Con esto, los 15 dominios del sistema quedan documentados.**
