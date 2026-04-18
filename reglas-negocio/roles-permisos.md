# Roles y Permisos

## Roles del sistema

| Rol | Descripción |
|-----|-------------|
| `ADMINISTRADOR` | Acceso total al sistema |
| `GERENTE_GENERAL` | Visión global, aprobaciones de alto nivel |
| `GERENTE_AGENCIA` | Gestión de su agencia |
| `SUPERVISOR_CREDITO` | Supervisión de evaluaciones y aprobaciones |
| `ANALISTA_CREDITO` | Creación y gestión de evaluaciones |
| `COMITE_CREDITO` | Votación en comité de crédito |
| `CAJERO_COBRANZA` | Desembolsos, caja, cobranza |
| `AUDITOR` | Solo lectura, auditoría |

## Matriz de acceso por módulo

| Módulo | ADMIN | GER_GEN | GER_AG | SUPERV | ANALISTA | COMITE | CAJERO | AUDITOR |
|--------|-------|---------|--------|--------|----------|--------|--------|---------|
| Evaluación | ✅ | 👁 | 👁 | ✅ | ✅ | 👁 | 👁 | 👁 |
| Aprobación | ✅ | ✅ | ✅ | ✅ | 👁 | ✅ | 👁 | 👁 |
| Desembolso | ✅ | 👁 | 👁 | 👁 | 👁 | 👁 | ✅ | 👁 |
| Cobranza | ✅ | 👁 | 👁 | 👁 | 👁 | — | ✅ | 👁 |
| Caja | ✅ | 👁 | 👁 | 👁 | — | — | ✅ | 👁 |
| Bóveda | ✅ | ✅ | ✅ | ✅ | — | — | ✅ | 👁 |
| Reportes | ✅ | ✅ | ✅ | ✅ | ✅ | — | — | ✅ |
| Metas | ✅ | ✅ | ✅ | ✅ | — | — | — | 👁 |
| Usuarios | ✅ | — | — | — | — | — | — | — |

`✅` = lectura y escritura | `👁` = solo lectura | `—` = sin acceso

## Scoping por agencia
Cada usuario solo ve datos de su propia agencia (filtrado automático por JWT).
El `ADMINISTRADOR` y `GERENTE_GENERAL` ven todas las agencias.
