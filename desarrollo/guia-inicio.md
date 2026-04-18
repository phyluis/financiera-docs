# Guía de Inicio

## Requisitos previos

- Java 17
- Maven 3.8+
- Node.js 20+
- PostgreSQL 14+
- Git

## Clonar los repositorios

```bash
cd C:\Proyectos\Reelige

git clone -b desarrollo https://github.com/phyluis/financiera-backend.git
git clone -b desarrollo https://github.com/phyluis/financiera-frontend.git
```

## Configurar base de datos

1. Crear la base de datos en PostgreSQL:
```sql
CREATE DATABASE "dbFinanciera";
```

2. El esquema se crea automáticamente con `spring.jpa.hibernate.ddl-auto=update`

## Iniciar el backend

```bash
cd financiera-backend
mvn spring-boot:run
# Corre en http://localhost:8081
```

## Iniciar el frontend

```bash
cd financiera-frontend
npm install --legacy-peer-deps
ng serve
# Corre en http://localhost:4200
```

## Credenciales de prueba

| Usuario | Contraseña | Rol |
|---------|-----------|-----|
| admin | Admin123* | ADMINISTRADOR |
| gerente.acu | Admin123* | GERENTE_AGENCIA |
| supervisor.acu | Admin123* | SUPERVISOR_CREDITO |
| analista1.acu | Admin123* | ANALISTA_CREDITO |

## Build para despliegue

```bash
# Frontend (genera estáticos dentro del backend)
cd financiera-frontend
ng build --configuration production

# Backend (incluye el frontend compilado)
cd financiera-backend
mvn clean package -DskipTests
java -jar target/app-0.0.1-SNAPSHOT.jar
```
