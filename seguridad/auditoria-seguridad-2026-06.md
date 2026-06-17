# Auditoría de seguridad — junio 2026

> Assessment **autorizado** del sistema Reelige (dueño: equipo Reelige). Combina **recon no
> intrusivo** sobre `https://reeligecloud.com` y **auditoría de código** del backend/frontend
> (OWASP). Sin pruebas destructivas ni acceso a datos reales de clientes.
>
> **Objetivo recon:** `reeligecloud.com` (= servidor piloto `50.16.139.45`).
> **Fecha:** 2026-06-17.

---

## 1. Resumen ejecutivo

| Severidad | Hallazgo | Estado |
|---|---|---|
| 🟧 Media | Cabeceras de seguridad faltantes en el SPA (nginx) | ✅ Corregido en vivo |
| 🟧 Media-Alta | CORS `*` + `allowCredentials(true)` | ✅ Corregido (PR backend #17) |
| 🟧 Media | Secreto JWT por defecto en repo público | ✅ Corregido (PR #17) |
| 🟧 Media | `/api/files` servía PII de clientes sin auth | ✅ Corregido (PR #17 BE + #28 FE) |
| 🟨 Baja | Divulgación de versión de nginx | ✅ Corregido (`server_tokens off`) |
| 🟦 Info | Dependencias de CDN externas (Google Fonts, jsdelivr) | 📋 Pendiente (opcional) |

**Positivos:** sin SQL injection (queries parametrizadas), RBAC sólido (`@PreAuthorize` en 27/29
controllers), BCrypt, rate-limit en login, TLS 1.2 con cifrado fuerte, servicios internos
(8080/5432) no expuestos.

---

## 2. Recon no intrusivo

| Aspecto | Resultado |
|---|---|
| Servidor | `nginx` (versión oculta tras el fix; antes `nginx/1.24.0 (Ubuntu)`) |
| Redirección | HTTP 80 → 301 → HTTPS ✅ |
| TLS | Let's Encrypt, válido jun–sep 2026; TLS 1.2 `ECDHE-ECDSA-AES256-GCM-SHA384` ✅ |
| DNS | `reeligecloud.com` → `50.16.139.45` (= piloto) |
| Puertos públicos | Solo 80/443 (8080 backend y 5432 PostgreSQL internos) ✅ |

### Cabeceras de seguridad (corregido en vivo)
Antes faltaban todas. Añadidas a nivel `http{}` en `nginx.conf` (heredadas a todos los bloques),
con backup `nginx.conf.bak.<timestamp>`:

```nginx
server_tokens off;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Content-Security-Policy-Report-Only "default-src 'self'; img-src 'self' data:; font-src 'self' https://fonts.gstatic.com; style-src 'self' 'unsafe-inline' https://fonts.googleapis.com https://cdn.jsdelivr.net; script-src 'self'" always;
```

> La **CSP está en `Report-Only`** a propósito. Pasar a enforce (`Content-Security-Policy`) tras
> verificar que no genera violaciones con Angular. Idealmente self-hostear fuentes/iconos para
> poder quitar los orígenes de CDN de la política.

---

## 3. Auditoría de código (backend)

### 3.1 CORS — `allowedOriginPatterns('*')` + `allowCredentials(true)` 🟧
Permitía a cualquier sitio hacer peticiones autenticadas cross-origin. La propiedad
`cors.allowed-origins` existía pero `SecurityConfig` la ignoraba (atajo de dev para ngrok).

**Fix:** `SecurityConfig` ahora usa allowlist desde `cors.allowed-origins`. Dev mantiene
`localhost`/ngrok; **prod default** a `reeligecloud.com`, `www.reeligecloud.com`,
`financieracredidar.com`, sobrescribible con `CORS_ALLOWED_ORIGINS`.

### 3.2 Secreto JWT por defecto en repo público 🟧
`application.properties` traía `JWT_SECRET` con default realista
(`reelige-...-clave-secreta-jwt`) en un repo público de GitHub. Cualquier entorno sin la env
seteada firmaría tokens con un secreto conocido → **forja de tokens / bypass total**.

**Verificado:** el piloto/prod sí sobrescribe (secreto de 128 chars). **Fix:** el default pasó a
ser un valor obviamente *dev-only*; el perfil prod ya exige `JWT_SECRET`/`JWT_REFRESH_SECRET` por
entorno (sin fallback). *Recomendación adicional: rotar el secreto histórico ya committeado.*

### 3.3 `/api/files` exponía PII de clientes 🟧
`GET /api/files/**` era `permitAll` para que las `<img src>` cargaran sin token. Servía
croquis/fotos/expedientes (PII, Ley 29733) accesibles ante cualquier URL filtrada.

**Fix (backend):** `/api/files/empresa/**` sigue público (branding, se muestra antes de login);
el resto exige JWT.
**Fix (frontend):** directiva **`secureImg`** que descarga la imagen vía HttpClient (el
interceptor adjunta el token) y la muestra como blob. Aplicada al preview de adjuntos del cliente.

> ⚠️ **Despliegue en lockstep:** backend y frontend deben desplegarse juntos. Si solo va el
> backend, los previews de imágenes de cliente se rompen.

### 3.4 Lo que está BIEN ✅
- **SQL injection: ninguna** — todas las native queries usan parámetros `:nombre` + `CAST`.
- **RBAC** — `@PreAuthorize` en 27/29 controllers (191 usos); `/api/**` exige autenticación.
- **CSRF disabled** correcto (API stateless con JWT en header, sin cookies).
- **BCrypt** para passwords; **`LoginRateLimitFilter`** anti fuerza bruta; HSTS/frameOptions
  también a nivel Spring.

---

## 4. Pendientes / recomendaciones

| Prioridad | Acción |
|---|---|
| 🟨 | Pasar la CSP de `Report-Only` a enforce tras revisar violaciones; self-hostear fuentes/iconos. |
| 🟨 | Rotar el secreto JWT histórico que estuvo committeado. |
| 🟦 | Verificar en SSL Labs que TLS 1.0/1.1 estén deshabilitados. |
| 🟦 | Corregir typo `server_name reeligeclaud.com` en el bloque 443 (funciona por `default_server`). |
| 🟦 | Considerar URLs firmadas para archivos como capa extra. |

---

## 5. Trazabilidad

| Hallazgo | Commit / PR |
|---|---|
| Cabeceras nginx | Aplicado en el servidor (no versionado; backup en `nginx.conf.bak.*`) |
| CORS + JWT | backend `f25ef3f` · [PR #17](https://github.com/phyluis/financiera-backend/pull/17) |
| /api/files (BE) | backend `b44115c` · PR #17 |
| secureImg (FE) | frontend `2c5d9b9` · [PR #28](https://github.com/phyluis/financiera-frontend/pull/28) |

*Auditoría realizada 2026-06-17. Próxima revisión sugerida: tras el merge a calidad y antes del
pase a producción.*
