# RN-CART · Cartera (asignación de clientes a analistas)

> La cartera define **qué analista atiende a cada cliente**. Se puede reasignar un cliente o
> transferir carteras completas entre analistas, conservando el **historial**.
>
> Fuente en código: `model/AsignacionCartera.java`, `service/CarteraServiceImpl.java`,
> `controller/CarteraController.java`.

---

## 1. Propósito

Gestionar la asignación de clientes a analistas y sus transferencias, con trazabilidad de quién
atendió a cada cliente y desde cuándo.

---

## 2. Modelo — Asignación

Cada `AsignacionCartera` tiene `clienteId`, `analistaId`, `fechaInicio`, `fechaFin` y
`tipoOperacion` (`INDIVIDUAL` / `MASIVA` / `PARCIAL`).

> **Asignación vigente = la que tiene `fechaFin = NULL`.** Reasignar **cierra** la vigente
> (le pone `fechaFin`) y **abre** una nueva.

---

## 3. Reglas

| ID | Regla | Fuente |
|---|---|---|
| **RN-CART-01** | **Reasignar** un cliente cierra su asignación vigente y crea una nueva con el nuevo analista; el historial acumula ambas | `reasignarIndividual()` |
| **RN-CART-02** | **Transferencia masiva**: mueve **todos** los clientes vigentes del analista origen al destino | `transferirMasiva()` |
| **RN-CART-03** | No se puede transferir al **mismo** analista (origen ≠ destino) | `transferirMasiva()` → `IllegalArgumentException` |
| **RN-CART-04** | No se puede transferir desde un analista **sin clientes** | `transferirMasiva()` → `IllegalStateException` |
| **RN-CART-05** | **Transferencia parcial**: mueve solo los `clienteIds` indicados | `transferirParcial()` |
| **RN-CART-06** | Permisos: gestionar cartera → `GERENTE_GENERAL`, `ADMINISTRADOR` | `@PreAuthorize` |

---

## 4. Casos borde / negativos

| Caso | Resultado |
|---|---|
| Reasignar un cliente sin asignación previa | crea la primera asignación (toma el nombre del cliente) |
| Transferir masiva origen = destino | `IllegalArgumentException` |
| Transferir masiva desde analista sin clientes | `IllegalStateException` |

---

## 5. Trazabilidad (regla → prueba)

| Regla | Prueba | Estado |
|---|---|---|
| RN-CART-01 (reasignar + historial) | `CarteraAsignacionTest.reasignarIndividual_cambiaElAnalistaActual` | ✅ |
| RN-CART-02 (transferencia masiva) | `CarteraAsignacionTest.transferirMasiva_mueveTodosLosClientes` | ✅ |
| RN-CART-03 (mismo analista) | `CarteraAsignacionTest.transferirMasiva_mismoOrigenYDestino_rechazada` | ✅ |
| RN-CART-04 (origen sin clientes) | `CarteraAsignacionTest.transferirMasiva_origenSinClientes_rechazada` | ✅ |
| RN-CART-05 (transferencia parcial) | _pendiente_ | ❌ |

> Nota: el reporte de **cartera de clientes** (saldos/mora por scope) está en
> [RN-ROL/scope](./roles-permisos.md) y se prueba en `ScopeCarteraClientesPostgresTest`.

---

## Changelog
- **2026-06-13** — Documento nuevo desde el código: modelo de asignación (vigente = `fechaFin`
  null), reglas RN-CART-01..06 (reasignar, transferencia masiva/parcial, rechazos). Cubierto por
  `CarteraAsignacionTest` (falta transferencia parcial).
