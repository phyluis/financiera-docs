# Roadmap — Doble Verificación (2FA) y Auditoría de APIs

**Estado:** 🟡 Análisis completo · pendiente decisión final
**Fecha de análisis:** 2026-05-07
**Siguientes pasos bloqueados por:** elección de método (A/B/C/D)

---

## 🎯 Objetivo

Implementar autenticación de doble factor (2FA) para operaciones sensibles del
sistema, junto con una capa de auditoría de APIs que registre quién, cuándo
y qué se hizo en endpoints críticos.

**Valor de negocio:**
- Protección ante credenciales comprometidas
- Trazabilidad ante auditorías o disputas internas
- Compliance con buenas prácticas financieras

---

## 📋 Decisiones tomadas (firmes)

### 1. Arquitectura: **2 bases de datos separadas**

```
┌──────────────────────────┐    ┌──────────────────────────┐
│   dbFinanciera           │    │   dbFinanciera_audit     │
│   (BD principal)         │    │   (BD secundaria)        │
│                          │    │                          │
│  • Préstamos             │    │  • tokens_otp            │
│  • Clientes              │    │  • api_audit_log         │
│  • Movimientos           │    │  • whatsapp_msg_log      │
│  • Caja                  │    │                          │
│  • Reportes              │    │  Aislada por seguridad   │
└──────────────────────────┘    └──────────────────────────┘
        ↑                                   ↑
        └─────── Backend Spring Boot ──────┘
                        ↓
              [Proveedor 2FA — pendiente decisión]
```

**Justificación:**
- Aislamiento de PII / credenciales (no contamina la BD operativa)
- Auditoría inmutable y separada para cumplimiento
- Performance — los logs no compiten por recursos con queries operativas

**Stack recomendado:** misma instancia de PostgreSQL, base de datos separada
con conexión dedicada en Spring Boot vía `@Primary` y `@Qualifier`.

### 2. Datos de auditoría a registrar

Por cada llamada a endpoint sensible:
- Timestamp
- Usuario (ID, nombre, rol, agencia)
- Método HTTP + endpoint
- Request body (con passwords/tokens redactados)
- Response status
- Tiempo de respuesta (ms)
- IP origen

### 3. Eventos que dispararán 2FA (preliminar)

A confirmar con el negocio. Lista candidata:
- Login de usuarios con rol sensible (ADMINISTRADOR, GERENTE_GENERAL, GERENTE_AGENCIA)
- Cambio de contraseña
- Desembolso > monto X (a definir)
- Aprobación de extorno
- Anulación de cierre de caja
- Modificación de datos críticos del cliente

---

## ❓ Decisiones pendientes

### 🔴 Crítica — Elección de método 2FA

**Comparación de opciones evaluadas:**

| Método | Costo recurrente | Setup | UX | Recomendación |
|---|---|---|---|---|
| **(A) WhatsApp Cloud API (Meta)** | ~USD 5-15/mes (volumen interno) | 2-3 semanas (verificación Meta) | Excelente — recibe en el celular que ya usa | Buen balance |
| **(B) SMS (Twilio o local)** | ~USD 0.05/msg ≈ USD 25-50/mes | 1-2 días | Buena — funciona en cualquier celular | Costo más alto |
| **(C) Email (SMTP propio)** | Gratis | 1 día | Regular — requiere abrir email | Solo backup |
| **(D) TOTP (Google Authenticator)** ⭐ | **Gratis para siempre** | 1 día | Buena — instalación inicial + scan QR | **Recomendado para uso interno** |

#### ¿Por qué TOTP es la recomendación más fuerte?

- **0 costo recurrente** — ahora y siempre
- **Funciona offline** — no depende de internet, telefonía ni proveedor externo
- **Igual de seguro** que SMS/WhatsApp (RFC 6238)
- Implementación con librería madura (`googleauth`, `aerogear/keycloak`)
- Soporte universal: Google Authenticator, Microsoft Authenticator, Authy, 1Password

#### ¿Cuándo conviene WhatsApp en lugar de TOTP?

- Si los usuarios son **clientes finales** (no internos)
- Si los usuarios **rotan mucho** (alta vs baja frecuente) y el costo de capacitación de TOTP es alto
- Si existe un requisito específico del negocio de usar el canal WhatsApp
- Si ya tienen presupuesto y la experiencia de WhatsApp es deseada

---

## 💰 Análisis de costos detallado

### Meta Business Manager / WhatsApp Cloud API

**Cuenta y gestión:** Gratis · ✅ siempre

**Conversaciones (modelo per-conversation):**

| Tipo | Precio Perú | Cuándo aplica |
|---|---|---|
| **Authentication** (OTP/2FA) | ~USD 0.0085 | Códigos de verificación |
| **Utility** | ~USD 0.012 | Avisos transaccionales |
| **Marketing** | ~USD 0.0407 | Promociones |
| **Service** (cliente inicia) | **Gratis siempre** | Conversación bidireccional iniciada por cliente |

⚠️ **Precios sujetos a cambio.** Verificar siempre en
https://developers.facebook.com/docs/whatsapp/pricing

### Tier gratuito — historial de cambios

| Fecha | Modelo |
|---|---|
| Hasta jun 2023 | 1000 conversaciones gratis al mes (todas las categorías) |
| Jun 2023 – Jun 2024 | 1000 gratis pero **excluyendo Authentication** |
| **Desde jul 2024** | Tier gratuito de business-initiated **eliminado en muchos países** |
| **Hoy (2026)** | Service: gratis siempre. Authentication: paga desde el msg 1 |

**Conclusión:** no contamos con tier gratuito permanente para 2FA por WhatsApp.
Hay que asumir el costo desde el primer mensaje.

### Cálculo estimado para CrediActiva

Con 16 usuarios internos activos y eventos sensibles:

| Escenario | Mensajes/mes | Costo/mes (S/) |
|---|---|---|
| 2FA solo en login diario | ~480 | S/ 15 |
| + ops sensibles (extorno, cierre) | ~800 | S/ 26 |
| Si extiende a clientes (pagos) | ~5,000 | S/ 160 |

**Comparación con SMS:** WhatsApp resulta ~5x más barato que SMS local en Perú.

---

## 📦 Plan de implementación (cuando se decida)

### Fase 1 — Setup técnico (1-2 días, independiente del proveedor)

1. Crear BD secundaria `dbFinanciera_audit` (mismo Postgres)
2. Configurar 2 datasources en Spring Boot
3. Crear entidades:
   - `TokenOtp` (codigo, usuarioId, expiraEn, usado, intentos)
   - `ApiAuditLog` (timestamp, usuarioId, endpoint, método, status, ms, ip)
   - `WhatsappMessageLog` (to, template, status, errorCode, fecha) — solo si elegimos WhatsApp
4. Crear `OtpService` con métodos `generar` / `validar` / `invalidar`
5. AOP (`@AuditApi`) que captura llamadas a endpoints sensibles automáticamente

### Fase 2 — Integración del proveedor (depende de elección)

#### Si elegimos TOTP (recomendado)
- Agregar dependencia `googleauth` o similar
- Endpoint `POST /api/auth/2fa/setup` → genera secret, devuelve QR base64
- Endpoint `POST /api/auth/2fa/verify` → valida código de 6 dígitos
- Tabla `usuario_2fa_secret` con secret encriptado por usuario

#### Si elegimos WhatsApp Cloud API
- Setup en Meta Business Manager (CrediActiva debe iniciar trámite)
- Verificar número de WhatsApp Business
- Crear plantilla `AUTHENTICATION` con variable `{{1}}` para el código
- `WhatsappService` con HttpClient + Bearer token a Cloud API
- Webhook receptor de status (delivered/read)

#### Si elegimos SMS (Twilio)
- Cuenta Twilio
- Configurar número origen
- `SmsService` con SDK Twilio Java

### Fase 3 — Aplicar 2FA en endpoints sensibles

- Anotación custom `@Require2FA` en endpoints críticos
- Interceptor que verifica token OTP antes de procesar
- Frontend: modal de "Ingresá tu código" con countdown de expiración (5 min)
- Botón "Reenviar código" (con throttle de 60s)

### Fase 4 — Auditoría completa de APIs

- AOP `@AuditApi` aplica a:
  - Todos los endpoints `POST/PUT/PATCH/DELETE` de operaciones sensibles
  - Login / logout / cambio de password
- Vista admin `/admin/auditoria` con filtros (usuario, fecha, endpoint, status)
- Export CSV/Excel del log

### Estimación total

| Fase | Si TOTP | Si WhatsApp | Si SMS |
|---|---|---|---|
| Fase 1 | 2 días | 2 días | 2 días |
| Fase 2 | 1 día | 3 días + 2-3 sem espera Meta | 1 día |
| Fase 3 | 2 días | 2 días | 2 días |
| Fase 4 | 1 día | 1 día | 1 día |
| **Total** | **~6 días** | **~8 días + espera** | **~6 días** |

---

## 🔗 Referencias

- **Meta WhatsApp Cloud API**: https://developers.facebook.com/docs/whatsapp/cloud-api/
- **Meta Pricing**: https://developers.facebook.com/docs/whatsapp/pricing
- **Twilio WhatsApp**: https://www.twilio.com/whatsapp
- **TOTP (RFC 6238)**: https://datatracker.ietf.org/doc/html/rfc6238
- **GoogleAuth Java lib**: https://github.com/wstrange/GoogleAuth

---

## 🚦 Cómo retomar este roadmap

Cuando se quiera avanzar:

1. **Decidir método 2FA** (A/B/C/D) — preferentemente decidir entre TOTP (gratis) o WhatsApp (UX)
2. **Confirmar lista final de eventos sensibles** que requerirán 2FA
3. **Si se elige WhatsApp:**
   - Asignar a alguien que inicie trámite en Meta Business Manager
   - Verificar número de WhatsApp empresarial
   - Crear cuenta de desarrollador y obtener tokens
4. **Asignar Sprint** según estimación de la fase elegida
5. **Volver a este documento** y marcar las decisiones tomadas

**Responsable:** _por asignar_
**Sprint estimado:** _por planificar_
**Bloqueadores actuales:**
- ❓ Decisión TOTP vs WhatsApp vs SMS pendiente
- ❓ Si se elige WhatsApp: trámites en Meta Business no iniciados

---

> 📝 **Nota:** este documento se generó tras conversación de análisis entre
> el equipo y Claude (sesión 2026-05-07). Cualquier cambio de precios, políticas
> de Meta o decisiones del negocio deberían reflejarse aquí.
