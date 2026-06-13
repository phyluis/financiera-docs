# RN-GAR · Garantías y Garantes

> Los **bienes en garantía** (y los garantes/avales) respaldan el crédito. Se registran sobre la
> **evaluación** del cliente y se valora su cobertura respecto al monto.
>
> Fuente en código: `model/BienGarantia.java`, `service/BienGarantiaServiceImpl.java`,
> `controller/BienGarantiaController.java` (ruta `/api/evaluaciones/{evaluacionId}/bienes-garantia`).

---

## 1. Propósito

Registrar los bienes que respaldan un crédito (vehículos, inmuebles, etc.), su valor y el
**porcentaje de cobertura** respecto al valor comercial, como insumo de la evaluación.

---

## 2. Reglas — Bienes en garantía

| ID | Regla | Fuente |
|---|---|---|
| **RN-GAR-01** | Un bien se registra **sobre una evaluación**; nace con `estadoBien = ACTIVO` | `registrar()` |
| **RN-GAR-02** 💰 | **% de cobertura** = `valorAsignado / valorComercial × 100` (calculado al guardar) | `mapFromRequest` |
| **RN-GAR-03** | No se puede registrar/editar contra una **evaluación inexistente** → 404 | `requireEvaluacion` |
| **RN-GAR-04** | Un bien **pertenece a su evaluación**: no se actualiza/elimina desde otra → 404 | `actualizar()` / `eliminar()` |
| **RN-GAR-05** | Permisos: registrar/editar → `ADMIN`, `GERENTE_AGENCIA`, `SUPERVISOR`, `ANALISTA`; consultar → además `GERENTE_GENERAL`, `AUDITOR` | `@PreAuthorize` |

> Datos del bien: `tipoBien`, `descripcion`, `marca/modelo/anio/numeroSerie`, `estadoConservacion`,
> `valorComercial`, `valorAsignado`, `ubicacion`, título (`tieneTitulo`/`numeroTitulo`),
> seguro (`tieneSeguro`/`numeroPoliza`).

---

## 3. Casos borde / negativos

| Caso | Resultado |
|---|---|
| Registrar contra evaluación inexistente | 404 (`ResponseStatusException`) |
| Actualizar/eliminar un bien vía otra evaluación | 404 |
| `valorComercial = 0` o nulo | no calcula % de cobertura (evita división por cero) |

---

## 4. Trazabilidad (regla → prueba)

| Regla | Prueba | Estado |
|---|---|---|
| RN-GAR-02 (% de cobertura) | `GarantiaBienTest.registrar_calculaPorcentajeCobertura` | ✅ |
| RN-GAR-03 (evaluación inexistente) | `GarantiaBienTest.registrar_evaluacionInexistente_lanza404` | ✅ |
| RN-GAR-04 (bien de otra evaluación) | `GarantiaBienTest.noSePuedeActualizarBienDeOtraEvaluacion` | ✅ |

---

## 5. Pendiente

- **Garantes / avales** (`GaranteService`, `EvaluacionGaranteService`): sub-módulo relacionado
  (personas que avalan el crédito) — aún sin documentar ni probar.

---

## Changelog
- **2026-06-13** — Documento nuevo desde el código: reglas RN-GAR-01..05 de bienes en garantía
  (% de cobertura, pertenencia a la evaluación, permisos). Cubierto por `GarantiaBienTest`.
  Pendiente: garantes/avales.
