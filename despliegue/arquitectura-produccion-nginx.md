# Arquitectura de Producción — Nginx + Spring Boot + PostgreSQL

> Documento de referencia para el pase a producción del sistema Reelige (CrediActiva)
> sobre **AWS EC2 Ubuntu** con **Docker Compose** y **Nginx** como reverse proxy.

---

## 1. Visión general

En producción, **Nginx** es la puerta de entrada (recepcionista). El navegador
**nunca habla directo con Spring Boot**: todo pasa por Nginx, que decide:

- **`/`** (la app Angular) → la sirve Nginx directo desde disco (rápido + cache).
- **`/api/**`** → la reenvía (proxy) a Spring Boot en un puerto **interno** (`8080`),
  cerrado a internet.

```
┌─────────────── ANTES (QA) ───────────────┐
│ Navegador ─:8082─> Spring Boot ─> PostgreSQL │
│                    (hace TODO)             │
└────────────────────────────────────────────┘

┌─────────────── DESPUÉS (PROD) ────────────┐
│             ┌── static (/) ──> disco       │
│ Navegador ──┤                              │
│  (HTTPS)    └── /api/** ──> Spring Boot ──> PostgreSQL │
│           ▲                  (solo API)     │
│         NGINX (SSL + cache)                 │
└────────────────────────────────────────────┘
```

---

## 2. Diagrama de Despliegue (procesos en el EC2)

```
                              ☁ INTERNET
                                  │
                                  │  HTTPS :443  /  HTTP :80 (redirige a 443)
                                  ▼
╔══════════════════════════════════════════════════════════════════════╗
║                    AWS EC2  ·  Ubuntu 22.04 LTS                        ║
║                    Security Group: 80, 443 abiertos · 8080/5432 NO     ║
║                                                                        ║
║   ┌──────────────────────────────────────────────────────────────┐   ║
║   │                    DOCKER ENGINE                               │   ║
║   │                                                                │   ║
║   │   ┌────────────────────┐         ┌──────────────────────────┐ │   ║
║   │   │  Contenedor NGINX  │         │ Contenedor SPRING BOOT   │ │   ║
║   │   │                    │         │ (financiera-backend)     │ │   ║
║   │   │  :443 SSL          │         │                          │ │   ║
║   │   │  :80  → 443        │         │  Tomcat embebido         │ │   ║
║   │   │                    │ proxy   │  127.0.0.1:8080          │ │   ║
║   │   │  / → /usr/share/   │────────>│  perfil: prod            │ │   ║
║   │   │      nginx/html    │ /api/** │  JWT, JPA/Hibernate      │ │   ║
║   │   │   (Angular dist)   │         │                          │ │   ║
║   │   │                    │         └────────────┬─────────────┘ │   ║
║   │   │  gzip, cache,      │                      │ JDBC :5432    │   ║
║   │   │  headers seguridad │                      │               │   ║
║   │   └────────────────────┘                      │               │   ║
║   │            ▲                                   ▼               │   ║
║   │            │ volumen        ┌──────────────────────────────┐  │   ║
║   │   ┌────────┴─────────┐      │  Contenedor POSTGRESQL       │  │   ║
║   │   │ certbot (SSL)    │      │  (opción A: en EC2)          │  │   ║
║   │   │ /etc/letsencrypt │      │  :5432  (solo red interna)   │  │   ║
║   │   └──────────────────┘      │  volumen: pgdata (persiste)  │  │   ║
║   │                             └──────────────────────────────┘  │   ║
║   │                                                                │   ║
║   │   Red interna Docker: solo Nginx expone puertos al host        │   ║
║   └──────────────────────────────────────────────────────────────┘   ║
║                                                                        ║
║   Volúmenes persistentes (en disco del EC2):                          ║
║     · pgdata            → datos de PostgreSQL                          ║
║     · letsencrypt       → certificados SSL                             ║
║     · uploads           → adjuntos (croquis, fotos, logo)             ║
╚════════════════════════════════════════════════════════════════════════╝

        (Opción B recomendada para finanzas: PostgreSQL en AWS RDS,
         fuera del EC2, con backups automáticos diarios)
```

### Opción B — con AWS RDS (recomendada para datos financieros)

```
╔════════════════════ AWS EC2 ════════════════════╗      ╔══════════════════════╗
║  Docker: [NGINX] ── [SPRING BOOT] ───────────────╫─────>║  AWS RDS PostgreSQL  ║
║                                          JDBC     ║ :5432║  backups automáticos ║
╚═══════════════════════════════════════════════════╝      ║  point-in-time       ║
                                                            ╚══════════════════════╝
```

---

## 3. Componentes y responsabilidades

| Componente | Puerto | Expuesto a internet | Responsabilidad |
|---|---|---|---|
| **Nginx** | 443 / 80 | ✅ Sí | SSL/TLS, servir Angular estático, proxy `/api`, gzip, cache, headers de seguridad, rate-limit |
| **Spring Boot** | 8080 | ❌ No (solo red Docker) | API REST, JWT, lógica de negocio, JPA |
| **PostgreSQL** | 5432 | ❌ No (solo red Docker / RDS privado) | Persistencia |
| **Certbot** | — | — | Emisión y renovación automática de certificado Let's Encrypt |

---

## 4. Flujos de petición (resumen)

### 4.1 Carga inicial de la app
```
Navegador → Nginx → (¿/api? NO) → lee index.html + JS/CSS del disco → responde
```

### 4.2 Login (API)
```
Navegador → Nginx → (¿/api? SÍ) → proxy 127.0.0.1:8080 → Spring Boot
          → valida credenciales en PostgreSQL → genera JWT → responde
```

### 4.3 Petición autenticada
```
Navegador (Bearer JWT) → Nginx → proxy 8080 → Spring Boot
          → JwtFilter valida → FiltroSede aplica scope agencia → SELECT → responde
```

---

## 5. Tabla "quién hace qué" (antes vs después)

| Responsabilidad | QA (hoy) | Producción (Nginx) |
|---|---|---|
| Recibir HTTPS / terminar SSL | — | **Nginx** |
| Servir Angular (.js/.css/.html) | Spring Boot | **Nginx** (disco) |
| Atender `/api/**` | Spring Boot | Spring Boot (vía proxy) |
| Hablar con PostgreSQL | Spring Boot | Spring Boot |
| Comprimir (gzip) / cachear estáticos | — | **Nginx** |
| Rate-limiting / headers seguridad | App | **Nginx** + App |
| Puerto público | 8082 | **443 / 80** (8080 interno) |

---

## 6. Seguridad de red (AWS Security Group)

| Puerto | Origen | Acción | Motivo |
|---|---|---|---|
| 443 (HTTPS) | 0.0.0.0/0 | **Permitir** | Tráfico web público |
| 80 (HTTP) | 0.0.0.0/0 | **Permitir** | Solo para redirigir a 443 + reto Certbot |
| 22 (SSH) | IP del admin | **Permitir restringido** | Administración (no abrir a todos) |
| 8080 (Spring Boot) | — | **Bloquear** | Solo accesible por Nginx en red interna |
| 5432 (PostgreSQL) | — | **Bloquear** | Solo accesible por Spring Boot en red interna |

> **Regla de oro**: solo Nginx (443/80) y SSH (22, restringido) miran a internet.
> Spring Boot y PostgreSQL viven en la red interna de Docker.

---

## 7. Punto mental clave

> Nginx es un **recepcionista**: mira la URL.
> - Si piden un archivo (`/`, `/main.js`) → lo entrega él mismo del disco.
> - Si piden datos (`/api/...`) → llama por "teléfono interno" a Spring Boot (`localhost:8080`) y devuelve la respuesta.
>
> El navegador **nunca habla directo con Spring Boot** — siempre pasa por Nginx.

---

## 8. Decisiones pendientes de confirmar

| Decisión | Opciones | Estado |
|---|---|---|
| Base de datos | A) PostgreSQL en contenedor · B) AWS RDS gestionado | ⏳ Pendiente (recomendado: B para finanzas) |
| Dominio | Propio + Let's Encrypt · IP temporal | ⏳ Pendiente (HTTPS requiere dominio) |

> ⚠️ **HTTPS es necesario** no solo por seguridad: el modal "Compartir Recibo"
> usa la **Clipboard API** (`navigator.clipboard.write`) que **solo funciona en
> contexto seguro (HTTPS)** o `localhost`. Sin dominio + SSL, esa función falla
> en producción.

---

## 9. Pasos de implementación (checklist)

- [ ] **1.** Provisionar EC2 Ubuntu + configurar Security Group (sección 6)
- [ ] **2.** Instalar Docker + Docker Compose en el EC2
- [ ] **3.** Definir `application-prod.properties` (Spring escucha en 8080 interno)
- [ ] **4.** Crear `Dockerfile` del backend (build multi-stage)
- [ ] **5.** Compilar Angular producción (`ng build --configuration production`, `apiUrl=''`)
- [ ] **6.** Crear `nginx.conf` (static + proxy `/api` + gzip + headers)
- [ ] **7.** Crear `docker-compose.prod.yml` (Nginx + Spring Boot [+ PostgreSQL])
- [ ] **8.** Configurar `.env` de producción (secrets, DB, dominio)
- [ ] **9.** Apuntar dominio (registro A) a la IP pública del EC2
- [ ] **10.** Emitir certificado SSL con Certbot
- [ ] **11.** Primer arranque `docker compose up -d` + smoke test
- [ ] **12.** Configurar backups (RDS automático, o cron `pg_dump` → S3 si es contenedor)

---

## 10. Referencias del proyecto

- Perfil Spring producción: `application-prod.properties` (ya existe en el repo)
- `CORS_ALLOWED_ORIGINS` debe incluir el dominio de producción
- `JWT_SECRET` / `JWT_REFRESH_SECRET` obligatorios por variable de entorno (no hardcodear)
- El frontend usa `window.location.origin` cuando `apiUrl=''` → funciona detrás de Nginx
- Header `X-Forwarded-Proto: https` es necesario para que Spring Boot sepa que está tras SSL
