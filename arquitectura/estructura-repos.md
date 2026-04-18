# Estructura de Repositorios

## Repositorios

| Repo | GitHub | Ruta local | Descripción |
|------|--------|-----------|-------------|
| `financiera-backend` | github.com/phyluis/financiera-backend | `C:\Proyectos\Reelige\financiera-backend` | Spring Boot API |
| `financiera-frontend` | github.com/phyluis/financiera-frontend | `C:\Proyectos\Reelige\financiera-frontend` | Angular SPA |
| `financiera-docs` | github.com/phyluis/financiera-docs | `C:\Proyectos\Reelige\financiera-docs` | Documentación |

## Ramas (backend y frontend)

| Rama | Propósito | BD | Profile |
|------|-----------|-----|---------|
| `desarrollo` | Trabajo diario | `dbFinanciera` | `dev` |
| `calidad` | Validación e integración | `dbFinanciera_qa` | `qa` |
| `produccion` | Piloto con usuarios reales | `dbFinanciera_prod` | `prod` |
| `main` | Producto certificado y validado | — | — |

## Flujo de pase entre ramas

```
desarrollo  ──PR──►  calidad  ──PR──►  produccion  ──PR──►  main
```

**Regla:** Cada PR se revisa línea por línea antes de aprobar. Sin merges automáticos.

## Rama de documentación

`financiera-docs` tiene una sola rama: `main`.
