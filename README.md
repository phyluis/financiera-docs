# Sistema Reelige — Documentación

Sistema financiero white-label para gestión de créditos, cobranza y tesorería.

> 📌 Este índice refleja **solo documentos que existen**. Lo pendiente se rastrea en el
> [roadmap de reglas de negocio](reglas-negocio/README.md) y en [HALLAZGOS.md](HALLAZGOS.md).

## Estructura del repositorio

| Carpeta | Contenido |
|---|---|
| [`arquitectura/`](arquitectura/) | Visión general, stack técnico, estructura de repos |
| [`modulos/`](modulos/) | Especificación funcional por módulo |
| [`reglas-negocio/`](reglas-negocio/) | Reglas del negocio (con diagramas y trazabilidad a pruebas) |
| [`desarrollo/`](desarrollo/) | Guías, convenciones, plan de pruebas y prototipos UI |
| [`despliegue/`](despliegue/) | Ambientes y despliegue |
| [`roadmap/`](roadmap/) | Pendientes y nuevos requerimientos |
| [`demo/`](demo/) | Guion de demo y smoke tests |
| [`plantillas/`](plantillas/) | Plantillas Word/Excel base (contrato, pagaré, acta, autorización, solicitud) |
| [`ejemplos-export/`](ejemplos-export/) | Ejemplos de reportes generados (Excel) |

---

## Índice

### 🏛️ Arquitectura
- [Visión general](arquitectura/vision-general.md)
- [Stack técnico](arquitectura/stack-tecnico.md)
- [Estructura de repositorios](arquitectura/estructura-repos.md)

### 📦 Módulos
- [Productos de préstamo](modulos/productos-prestamo.md)
- [Reportes](modulos/reportes.md)

### 📐 Reglas de negocio  → [índice maestro + roadmap](reglas-negocio/README.md)
- [Roles y permisos](reglas-negocio/roles-permisos.md) ✅
- [Flujo del préstamo](reglas-negocio/flujo-prestamo.md) ✅
- _Pendientes_ (caja, movimientos, mora, extornos, cálculo de cuotas…): ver el índice maestro

### 🧪 Pruebas
- [Plan de pruebas](desarrollo/plan-de-pruebas.md) — estrategia + invariantes del dinero (D1–D7)
- [Bitácora de pruebas](desarrollo/bitacora-pruebas.md) — inventario + roadmap por fases

### 🧰 Desarrollo
- [Guía de inicio](desarrollo/guia-inicio.md)
- [Flujo Git](desarrollo/flujo-git.md)
- [Patrón de validación de formularios](desarrollo/patron-validacion-formularios.md)
- [White-label / marca](desarrollo/white-label-marca.md)
- [Prototipos UI](desarrollo/prototipos/)

### 🚀 Despliegue
- [**Arquitectura de despliegue** (dónde corre el sistema + diagrama)](despliegue/arquitectura-despliegue.md)
- [Ambientes](despliegue/ambientes.md)
- [Arquitectura de producción (Nginx)](despliegue/arquitectura-produccion-nginx.md)
- [Despliegue CrediDar en AWS](despliegue/despliegue-credidar-aws.md)

### 🗺️ Roadmap
- [2FA / doble verificación](roadmap/2fa-doble-verificacion.md)

### 🎬 Demo
- [Guion de demo (reportes)](demo/guion-demo-reportes.md)
- [Smoke test reportes Excel](demo/smoke-test-reportes-excel.md)

---

## 🔎 Seguimiento transversal
- [**HALLAZGOS.md**](HALLAZGOS.md) — bugs, derivas y decisiones pendientes detectadas en la revisión.
