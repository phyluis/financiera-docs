# Despliegue CREDIDAR — Servidor AWS (guía única consolidada)

> **Estado:** ✅ OPERATIVO en **https://financieracredidar.com** 🔒 — consolidado al 2026-06-03.
> Documento único en orden de **despliegue y configuración**.
>
> 🔐 **Credenciales ENMASCARADAS** (`<...>`) por ser repo compartido. Los valores
> reales están **solo** en el doc local `C:\AWS-DAR\despliegue-credidar-aws.md`
> (NO versionado). Pendientes post-arranque: ver **§16**.

---

## 1. Datos del servidor y conexión

| Campo | Valor |
|---|---|
| Nombre | servidor-angular-java |
| Proveedor | AWS Lightsail |
| Región | Virginia, Zone A (us-east-1a) |
| SO | Ubuntu 24.04 LTS |
| Recursos | 4 GB RAM, 2 vCPUs, 80 GB SSD |
| IP pública | 54.237.173.25 |
| Dominio | financieracredidar.com (Cloudflare) |
| Usuario SSH | ubuntu |
| SSH Key | `<SSH_KEY>.pem` |

```bash
ssh -i <SSH_KEY>.pem ubuntu@54.237.173.25
# (o Panel Lightsail → Connect → "Connect using SSH")
```

### Arquitectura
```
Internet (HTTPS 443) ─► NGINX ─┬─ /         ─► /var/www/html (Angular, index.csr.html)
                               └─ /api/**    ─► localhost:8080 (Spring Boot, perfil prod)
                                                       └─► PostgreSQL (localhost:5432)
```
Solo Nginx (80/443) y SSH (22) miran a internet. **8080 y 5432 son internos.**

---

## 2. Provisión base

```bash
# Sistema
sudo apt update && sudo apt upgrade -y
# Java 17
sudo apt install -y openjdk-17-jdk && java -version
# Node 20 + Angular CLI
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs && sudo npm install -g @angular/cli
# Nginx
sudo apt install -y nginx && sudo systemctl enable --now nginx
# PostgreSQL
sudo apt install -y postgresql postgresql-contrib && sudo systemctl enable --now postgresql
# Utilidad para generar el hash BCrypt del admin (§10)
sudo apt install -y apache2-utils
```

---

## 3. Firewall (puertos)

```bash
sudo ufw allow 22
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
sudo ufw status
```

| Puerto | Origen | Acción | Motivo |
|---|---|---|---|
| 22 | IP admin | Permitir | SSH |
| 80 | 0.0.0.0/0 | Permitir | HTTP → redirige a HTTPS + reto Certbot |
| 443 | 0.0.0.0/0 | Permitir | HTTPS |
| 8080 | — | **Bloquear** | Spring Boot interno (solo Nginx por localhost) |
| 5432 | — | **Bloquear** | PostgreSQL interno |

> ⚠️ **NO abrir 8080.** Si en un inicio se abrió, cerrarlo en **ufw** y en el
> **Panel Lightsail → Networking → IPv4 Firewall**.

---

## 4. Estructura de carpetas

```bash
mkdir -p /home/ubuntu/app/{instaladores,scripts,frontend,backups,uploads}
```
```
/home/ubuntu/app/
├── instaladores/  → .jar del backend
├── scripts/       → scripts de despliegue/backup
├── frontend/      → dist/ de Angular antes de desplegar
├── backups/       → backups de la BD
└── uploads/       → adjuntos persistentes (croquis, fotos, logo)   ⟵ ver §6
```

---

## 5. Base de datos PostgreSQL

```bash
sudo -u postgres psql
```
Ejecutar **cada sentencia por separado** (el `#` y comillas de la contraseña
pueden dejar el prompt en `postgres'#` → cancelar con `Ctrl+C`):
```sql
CREATE USER "dbSeusCar" WITH PASSWORD '<DB_PASSWORD>';
CREATE DATABASE "dbCrediCarSeus" OWNER "dbSeusCar";
GRANT ALL PRIVILEGES ON DATABASE "dbCrediCarSeus" TO "dbSeusCar";
\q
```
Verificar:
```bash
psql -U dbSeusCar -h localhost -d dbCrediCarSeus   # debe entrar; salir con \q
```

---

## 6. Archivo `.env` (versión final)

### 6.1 Generar los secretos JWT

El backend firma los tokens con **HS256** (`JwtService` → `Keys.hmacShaKeyFor(secret.getBytes(UTF_8))`),
que exige una clave de **mínimo 256 bits = 32 bytes**. Como usa los bytes UTF-8 del string, el
secreto debe tener **al menos 32 caracteres** (usamos 64 bytes hex = 128 chars para margen).

**No se escriben a mano ni se versionan.** Genéralos en el servidor con `openssl`:

```bash
# Genera DOS secretos distintos (hex evita / + = $ que rompen el parseo del .env)
echo "JWT_SECRET=$(openssl rand -hex 64)"
echo "JWT_REFRESH_SECRET=$(openssl rand -hex 64)"
```

Copia cada valor impreso al `.env` (abajo). Reglas:
- **Distintos entre sí**: `JWT_SECRET` ≠ `JWT_REFRESH_SECRET`.
- **Distintos por ambiente**: dev / calidad / producción cada uno el suyo.
- **Nunca al repo**: en este manual quedan enmascarados (`<JWT_SECRET>`); los valores reales solo
  viven en el `.env` del servidor (chmod 600) y, opcionalmente, en tu vault (`C:\AWS-DAR\`).
- **Rotación**: cambiar el secreto invalida todas las sesiones activas (los usuarios deben
  re-loguearse). Es lo esperado.

### 6.2 Escribir el `.env`

```bash
nano /home/ubuntu/app/.env
chmod 600 /home/ubuntu/app/.env
```
```bash
# ── Base de datos (Spring lee DB_URL, no DB_HOST/DB_NAME por separado) ──
DB_URL=jdbc:postgresql://localhost:5432/dbCrediCarSeus
DB_USERNAME=dbSeusCar
DB_PASSWORD=<DB_PASSWORD>

# ── JWT — obligatorios (generar con openssl, ver §6.1) ──
JWT_SECRET=<JWT_SECRET>
JWT_REFRESH_SECRET=<JWT_REFRESH_SECRET>
JWT_EXPIRATION=600000
JWT_REFRESH_EXPIRATION=604800000

# ── CORS — dominio con HTTPS ──
CORS_ALLOWED_ORIGINS=https://financieracredidar.com,https://www.financieracredidar.com

# ── Perfil ──
SPRING_PROFILES_ACTIVE=prod

# ── FIXES del despliegue (NO quitar) ──
# El application.properties base usa 8081 y el perfil prod no lo sobrescribe;
# Nginx proxea a 8080 → forzar el puerto.
SERVER_PORT=8080
# app.storage.root por defecto es ./uploads y systemd corre con WorkingDirectory=/
# → intentaría crear /uploads (AccessDenied). Apuntar a ruta absoluta del usuario.
APP_STORAGE_ROOT=/home/ubuntu/app/uploads
# La BD nace vacía y prod usa ddl-auto=validate (no crea tablas).
# 'update' SOLO para el primer arranque → revertir luego (ver §16).
SPRING_JPA_HIBERNATE_DDL_AUTO=update
```
> ⚠️ Solo legible por `ubuntu` (chmod 600).

---

## 7. Scripts de despliegue y backup

### 7.1 `scripts/deploy-backend.sh`
```bash
nano /home/ubuntu/app/scripts/deploy-backend.sh && chmod +x /home/ubuntu/app/scripts/deploy-backend.sh
```
```bash
#!/bin/bash
echo "=== Desplegando backend ==="
source /home/ubuntu/app/.env
sudo systemctl stop backend-app 2>/dev/null
cp /home/ubuntu/app/instaladores/*.jar /home/ubuntu/app/backend.jar
if [ ! -f /etc/systemd/system/backend-app.service ]; then
    sudo bash -c 'cat > /etc/systemd/system/backend-app.service <<EOF
[Unit]
Description=Backend Spring Boot
After=network.target postgresql.service
[Service]
User=ubuntu
EnvironmentFile=/home/ubuntu/app/.env
ExecStart=/usr/bin/java -jar /home/ubuntu/app/backend.jar
SuccessExitStatus=143
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target
EOF'
    sudo systemctl daemon-reload && sudo systemctl enable backend-app
fi
sudo systemctl start backend-app
sudo systemctl status backend-app --no-pager
```

### 7.2 `scripts/deploy-frontend.sh` (BLINDADO)

> ⚠️ **Versión segura.** El script anterior borraba `/var/www/html` **antes** de verificar que
> el origen tuviera archivos. Si se corría con `/home/ubuntu/app/frontend` vacío (p. ej. tras un
> `rm -rf` sin volver a subir el build), **dejaba el sitio sin archivos → 403**. Esta versión
> **aborta** si el origen está vacío o no trae `index.csr.html`, y respalda el sitio antes de reemplazar.

```bash
nano /home/ubuntu/app/scripts/deploy-frontend.sh && chmod +x /home/ubuntu/app/scripts/deploy-frontend.sh
```
```bash
#!/bin/bash
# Despliegue del frontend Angular a Nginx — BLINDADO contra origen vacío.
set -euo pipefail

SRC="/home/ubuntu/app/frontend"
DEST="/var/www/html"

echo "=== Desplegando frontend ==="

# 1. Guardas: el origen debe existir, NO estar vacío y traer el entrypoint Angular.
if [ ! -d "$SRC" ] || [ -z "$(ls -A "$SRC" 2>/dev/null)" ]; then
  echo "ABORTADO: '$SRC' está vacío o no existe. El sitio NO se toca."
  exit 1
fi
if [ ! -f "$SRC/index.csr.html" ]; then
  echo "ABORTADO: falta index.csr.html en '$SRC'. ¿Subiste el build? El sitio NO se toca."
  exit 1
fi

# 2. Backup del sitio actual antes de reemplazar (red de seguridad).
if [ -n "$(ls -A "$DEST" 2>/dev/null)" ]; then
  TS="$(date +%Y%m%d_%H%M%S)"
  sudo mkdir -p /home/ubuntu/app/backups/frontend
  sudo tar -czf "/home/ubuntu/app/backups/frontend/html_${TS}.tar.gz" -C "$DEST" . \
    && echo "Backup previo: /home/ubuntu/app/backups/frontend/html_${TS}.tar.gz"
fi

# 3. Reemplazo del contenido y publicación.
sudo rm -rf "${DEST:?}"/*
sudo cp -r "$SRC"/* "$DEST"/
sudo chown -R www-data:www-data "$DEST"/
sudo chmod -R 755 "$DEST"/
sudo nginx -t && sudo systemctl reload nginx

echo "=== Frontend desplegado OK ($(ls -1 "$DEST" | wc -l) elementos publicados) ==="
```

### 7.3 `scripts/backup-db.sh` + cron mensual
```bash
nano /home/ubuntu/app/scripts/backup-db.sh && chmod +x /home/ubuntu/app/scripts/backup-db.sh
```
```bash
#!/bin/bash
source /home/ubuntu/app/.env
BACKUP_DIR="/home/ubuntu/app/backups"; FECHA=$(date +%Y%m%d_%H%M%S)
ARCHIVO="$BACKUP_DIR/backup_dbCrediCarSeus_${FECHA}.sql.gz"
mkdir -p $BACKUP_DIR
PGPASSWORD=$DB_PASSWORD pg_dump -U $DB_USERNAME -h localhost -p 5432 dbCrediCarSeus | gzip > $ARCHIVO
find $BACKUP_DIR -name "*.sql.gz" -mtime +90 -delete
echo "Backup: $ARCHIVO"
```
```bash
crontab -e
# Día 1 de cada mes, 2:00 AM
0 2 1 * * /home/ubuntu/app/scripts/backup-db.sh >> /home/ubuntu/app/backups/backup.log 2>&1
```

---

## 8. Configuración Nginx (final aplicada)

```bash
sudo nano /etc/nginx/sites-available/default
sudo nginx -t && sudo systemctl reload nginx
```
Config base (antes de Certbot — Certbot agrega el 443/SSL en §12):
```nginx
server {
    listen 80;
    server_name financieracredidar.com www.financieracredidar.com;

    root /var/www/html;
    index index.csr.html;                 # ← Angular genera index.csr.html, NO index.html

    location / {
        try_files $uri $uri/ /index.csr.html;
    }

    location /api/ {
        proxy_pass http://localhost:8080;        # ← SIN "/" final (si no, 404 en toda la API)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 12M;                # uploads de croquis/fotos
    }
}
```
> **3 detalles críticos:** proxy **sin `/` final**, `index.csr.html` (no `index.html`),
> y `X-Forwarded-Proto` (para que Spring sepa que está tras HTTPS).

---

## 9. Build local + despliegue del backend (primer arranque)

```bash
# PC local — backend (rama produccion)
mvn clean package -DskipTests          # → target/app-0.0.1-SNAPSHOT.jar
scp -i <SSH_KEY>.pem target/app-0.0.1-SNAPSHOT.jar \
    ubuntu@54.237.173.25:/home/ubuntu/app/instaladores/

# Servidor
/home/ubuntu/app/scripts/deploy-backend.sh
sudo journalctl -u backend-app -f      # esperar "Started AppApplication in X seconds"
sudo ss -ltnp | grep 8080              # confirmar que Java escucha en 8080
```

---

## 10. Seed del usuario administrador (BD vacía)

El endpoint `/api/auth/hash/{password}` requiere rol ADMINISTRADOR (huevo/gallina),
así que el primer admin se crea por SQL con hash **BCrypt**.
```bash
# 1) Generar hash BCrypt (usar el password real del admin)
HASH=$(htpasswd -bnBC 12 "" '<ADMIN_PASSWORD>' | tr -d ':\n' | sed 's/^\$2y/\$2a/'); echo "$HASH"

# 2) Insertar agencia base + admin (pega el hash en <HASH_BCRYPT>)
PGPASSWORD='<DB_PASSWORD>' psql -U dbSeusCar -h localhost -d dbCrediCarSeus <<'SQL'
INSERT INTO agencias (nombre, codigo, ciudad, departamento, activo, created_at, updated_at)
VALUES ('Agencia Central Unificada', 'ACU', 'Huancayo', 'Junín', true, now(), now());

INSERT INTO usuarios (
  dni_cuit, paternal_last_name, maternal_last_name, first_name,
  email, username, password_hash, rol, estado,
  agencia_id, agencia_nombre, monto_maximo_aprobacion,
  requiere_cambio_password, intentos_fallidos_login, version
) VALUES (
  '00000000001', 'Administrador', 'Sistema', 'Admin',
  'admin@credidar.com', 'admin', '<HASH_BCRYPT>', 'ADMINISTRADOR', 'ACTIVO',
  1, 'Agencia Central Unificada', 0, false, 0, 0
);
SQL
```
- Credenciales iniciales: **admin / `<ADMIN_PASSWORD>`** (cambiar desde la UI — §16).
- Resetear hash de un admin existente:
  `UPDATE usuarios SET password_hash='<HASH>', estado='ACTIVO', requiere_cambio_password=false, intentos_fallidos_login=0, fecha_vencimiento_password=NULL WHERE username='admin';`

---

## 11. Despliegue del frontend

```bash
# PC local — build de marca CREDIDAR (white-label)
ng build --configuration credidar      # → dist/financiera-app/browser/
scp -i <SSH_KEY>.pem -r dist/financiera-app/browser/* \
    ubuntu@54.237.173.25:/home/ubuntu/app/frontend/

# Servidor
/home/ubuntu/app/scripts/deploy-frontend.sh
```
> El build excluye `prototipos/**` y usa budgets relajados (16/24kB) — ya configurado en `angular.json`.

---

## 12. Dominio + HTTPS (Cloudflare + Certbot)

### 12.1 DNS en Cloudflare
DNS → Add record (dos registros A, **nube GRIS = DNS only**):

| Type | Name | IPv4 | Proxy | TTL |
|---|---|---|---|---|
| A | `@`   | 54.237.173.25 | **DNS only** | Auto |
| A | `www` | 54.237.173.25 | **DNS only** | Auto |

> ⚠️ Nube **gris**, no naranja (si no, Certbot no valida). Verificar:
> `dig +short financieracredidar.com` → debe devolver la IP (5–30 min).

### 12.2 Certbot (server_name ya apunta al dominio — §8)
```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d financieracredidar.com -d www.financieracredidar.com
# Email real · ToS: Y · Newsletter: N · Redirect HTTP→HTTPS: opción 2
```
Certbot **agrega** el bloque 443/SSL + redirección 80→443 (conserva el resto de la config).
Renovación automática:
```bash
sudo certbot renew --dry-run
sudo systemctl list-timers | grep certbot
```

---

## 13. Actualizar CORS al dominio (ya reflejado en §6)

Si el `.env` aún tuviera la IP, cambiar a:
```bash
CORS_ALLOWED_ORIGINS=https://financieracredidar.com,https://www.financieracredidar.com
sudo systemctl restart backend-app
```

---

## 14. Verificación final

- `https://financieracredidar.com` → carga con candado 🔒 y login.
- `http://financieracredidar.com` → redirige a HTTPS.
- Login `admin / <ADMIN_PASSWORD>` → entra.
- Consola del navegador → sin errores de CORS en llamadas `/api/`.
- `sudo ss -ltnp | grep -E ':80|:443|:8080'` → 80/443 Nginx, 8080 Java (interno).

---

## 15. Operación continua (redeploys y actualizaciones)

```bash
# Redeploy backend
mvn clean package -DskipTests → scp .jar → /home/ubuntu/app/scripts/deploy-backend.sh

# Redeploy frontend CREDIDAR
ng build --configuration credidar → scp browser/* → /home/ubuntu/app/scripts/deploy-frontend.sh
```
**Al promover features nuevos**, aplicar las migraciones SQL en la BD prod
(perfil `validate` NO crea columnas solo):
```bash
PGPASSWORD='<DB_PASSWORD>' psql -U dbSeusCar -h localhost -d dbCrediCarSeus \
  -f migration-2026-06-metas-analista.sql
PGPASSWORD='<DB_PASSWORD>' psql -U dbSeusCar -h localhost -d dbCrediCarSeus \
  -f migration-2026-06-mora-desde-fecha-fin.sql
```

---

## 16. ⏳ Pendientes post-arranque

**Seguridad / estabilidad:**
- [ ] Revertir `ddl-auto`: quitar `SPRING_JPA_HIBERNATE_DDL_AUTO=update` del `.env` + `sudo systemctl restart backend-app` (confirmar arranque con `validate`).
- [ ] Cambiar la contraseña del admin desde la UI.

**Marca CREDIDAR (documentos):**
- [ ] En *Parámetros del Sistema* (UI, como admin): `EMPRESA_RAZON_SOCIAL = CREDIDAR - Sistema Financiero`, subir logo y setear `EMPRESA_LOGO_URL`. Así PDF/Word/Excel salen rebrandeados.

**Promoción Git:**
- [ ] PR `desarrollo → calidad → produccion` cuando se valide QA.

---

## 17. Troubleshooting (errores reales encontrados)

| Síntoma | Causa | Solución |
|---|---|---|
| `502 Bad Gateway` en `/api` | Spring escuchaba en 8081, Nginx proxea a 8080 | `SERVER_PORT=8080` en `.env` + restart |
| `403 Forbidden` (frontend) | Angular genera `index.csr.html`, Nginx buscaba `index.html` | `index index.csr.html;` + `try_files … /index.csr.html` |
| `AccessDeniedException: /uploads` al arrancar | `app.storage.root=./uploads` resuelve a `/` (systemd) | `APP_STORAGE_ROOT=/home/ubuntu/app/uploads` + `mkdir` |
| Login `Credenciales incorrectas` | BD recién creada sin usuarios | Seed de admin por SQL (§10) |
| Backend no arranca con `validate` | BD vacía sin tablas | `ddl-auto=update` SOLO el 1er arranque (§6), revertir luego |
| Certbot `Challenge failed` | Nube naranja en Cloudflare o puerto 80 cerrado | DNS only (gris); abrir 80 (ufw + Lightsail) |
| Certbot `could not find a vhost` | `server_name` no coincide con el dominio | `server_name financieracredidar.com www.…;` (§8) |
| Tras HTTPS, error de CORS | `.env` con CORS de la IP/HTTP | CORS al dominio https + restart (§13) |
| `429 Too Many Requests` (Let's Encrypt) | Demasiados intentos (límite semanal) | Esperar; usar `--dry-run` |
| psql queda en `postgres'#` | Comilla/`#` sin cerrar | `Ctrl+C` y reejecutar la sentencia sola |

---

## Anexo A — Runbook reutilizable (Dominio + HTTPS) para OTROS despliegues

Genérico con variables. Reemplazar: `DOMINIO`, `IP_SERVIDOR`, `EMAIL_ADMIN`.

1. **DNS (Cloudflare):** registros A `@` y `www` → `IP_SERVIDOR`, **DNS only (gris)**.
   `dig +short DOMINIO` debe devolver `IP_SERVIDOR`.
2. **Nginx:** `server_name DOMINIO www.DOMINIO;` → `nginx -t` → `reload`
   (solo cambia `server_name`; el resto se conserva).
3. **Certbot:** `sudo certbot --nginx -d DOMINIO -d www.DOMINIO`
   (Email, ToS=Y, Newsletter=N, Redirect=2). Agrega 443/SSL + redirección 80→443.
4. **CORS:** `CORS_ALLOWED_ORIGINS=https://DOMINIO,https://www.DOMINIO` + restart backend.
5. **Verificar:** `https://DOMINIO` con candado; `certbot renew --dry-run`.

> Requisitos: app ya servida por HTTP, puertos 80/443 abiertos (ufw + proveedor),
> dominio comprado (recomendado Cloudflare Registrar).
