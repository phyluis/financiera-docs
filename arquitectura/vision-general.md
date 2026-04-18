# Visión General

Sistema financiero full-stack para gestión de créditos, cobranza y tesorería de cooperativas o financieras con múltiples agencias.

## Componentes

```
financiera-frontend  (Angular 20)   :4200
       │
       │  HTTP/JWT
       ▼
financiera-backend   (Spring Boot)  :8081
       │
       │  JPA/Hibernate
       ▼
    PostgreSQL
```

## Agencias
El sistema maneja 3 sedes en Junín:
| Código | Nombre |
|--------|--------|
| ACU | Agencia Central Unificada |
| AC  | Agencia Central |
| AJ  | Agencia Junín |

## Estados principales

### Préstamos
`PRE_DESEMBOLSO` → `VIGENTE` → `LIQUIDADO`
También: `DEVUELTO`, `CANCELADO`, `VENCIDO`, `JUDICIAL`, `REFINANCIADO`

### Evaluaciones
`PENDIENTE` → `EN_COMITE` → `APROBADO` → `DESEMBOLSADO`
También: `OBSERVADO`, `RECHAZADO`, `DEVUELTO`
