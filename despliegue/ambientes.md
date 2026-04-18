# Ambientes

## Resumen

| Ambiente | Rama | Base de datos | Profile Spring | Build Angular |
|----------|------|--------------|----------------|---------------|
| Desarrollo | `desarrollo` | `dbFinanciera` | `dev` | `ng serve` |
| Calidad | `calidad` | `dbFinanciera_qa` | `qa` | `ng build --configuration qa` |
| Producción | `produccion` | `dbFinanciera_prod` | `prod` | `ng build --configuration production` |
| Certificado | `main` | — | — | — |

## Variables de entorno por ambiente

### Desarrollo (valores por defecto — no requiere configuración)
El perfil `dev` usa valores hardcodeados en `application-dev.properties`.

### Calidad
```bash
SPRING_PROFILES_ACTIVE=qa
```

### Producción
```bash
SPRING_PROFILES_ACTIVE=prod
DB_URL=jdbc:postgresql://HOST:5432/dbFinanciera_prod
DB_USERNAME=usuario_prod
DB_PASSWORD=password_seguro
JWT_SECRET=clave-secreta-produccion
JWT_REFRESH_SECRET=refresh-secret-produccion
CORS_ALLOWED_ORIGINS=https://dominio-produccion.com
```
