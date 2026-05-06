# Smoke test — Reportes Excel (Fases 1+2)

Lista de verificación rápida (~10 min) para validar que los 8 reportes XLSX
funcionan correctamente. Usar en **Etapa 1** (validación local) y
**Etapa 2 paso 6** (validación en QA antes de demo).

## Pre-requisitos

- [ ] Backend corriendo (`:8081` para dev, `:8082` para QA)
- [ ] Frontend corriendo (`:4200` para dev) o desplegado en `static/`
- [ ] Sesión iniciada como **ADMINISTRADOR** (ve los 6 grupos)
- [ ] Hay datos en la BD (al menos 2-3 préstamos vigentes con cuotas)

## Verificación general (aplica a TODOS los reportes)

Para cada uno de los 8 reportes, abrir el `.xlsx` descargado y confirmar:

- [ ] **Logo CrediActiva** aparece en la esquina superior izquierda
- [ ] **Título grande** del reporte alineado a la derecha (azul marino)
- [ ] **Fecha de generación** en español ("30 de abril de 2026")
- [ ] **Línea verde divisoria** debajo del header
- [ ] **KPIs con etiquetas** arriba de los números (no solo el número)
- [ ] **Tabla de detalle** con filas zebra (alternadas blanco/gris claro)
- [ ] **Cabecera de tabla** oscura con texto blanco
- [ ] **Fila TOTAL** al final con fondo oscuro, texto blanco
- [ ] **Fórmulas Excel reales** en TOTAL: Ctrl+Click en una celda de TOTAL
      → debe mostrar `=SUM(...)` o `=MAX(...)`, no un valor hardcoded
- [ ] **Sin errores en consola del backend** durante la descarga
- [ ] **Orientación apaisada** al hacer Vista Previa de Impresión

---

## Reportes uno por uno

### 1. Cartera en Mora — `/xlsx/cartera-mora`

**Cómo:** Tab "Cartera en Mora" → botón verde "Exportar Excel"

- [ ] 5 KPIs: TOTAL EN MORA, PRÉSTAMOS AFECTADOS, MAYOR ATRASO, CUOTAS VENCIDAS, % MORA / CARTERA
- [ ] Sección "DISTRIBUCIÓN POR RANGO" con 4 filas (0-30, 31-60, 61-90, 90+)
- [ ] Barras visuales `█████` en la columna "Visualización"
- [ ] Tabla con: Contrato, DNI, Cliente, Saldo Capital, Cuotas Venc., Días, Mora, Total, Rango
- [ ] Columna "Días" con color condicional (ámbar/naranja/rojo según severidad)
- [ ] Badge "Rango" con color sólido (no solo texto)
- [ ] Probar filtro `search` con DNI o nombre — el Excel respeta el filtro

### 2. Historial de Pagos — `/xlsx/historial-pagos`

**Cómo:** Tab "Historial de Pagos" → setear fechas → botón "Excel"

- [ ] 4 KPIs: PAGOS REGISTRADOS, TOTAL COBRADO, CAPITAL COBRADO, MORA COBRADA
- [ ] Tabla con: Recibo, Contrato, Cliente, DNI, Fecha·Hora, Capital, Interés, Mora, Total, Forma Pago, Comportamiento
- [ ] Columna "Comportamiento" con badge color (verde/ámbar/rojo) y días entre paréntesis
- [ ] Filtro por rango de fechas funciona
- [ ] Filtro `search` con cliente/DNI/contrato funciona

### 3. Movimientos de Caja — `/xlsx/movimientos-caja`

**Cómo:** Tab "Movimientos Caja" → seleccionar INGRESO o EGRESO → botón "Excel"

- [ ] 4 KPIs: N° MOVIMIENTOS, TOTAL [INGRESOS/EGRESOS], EFECTIVO, BANCO
- [ ] Sección "RESUMEN POR CONCEPTO" con tabla agrupada
- [ ] Tabla detalle: Fecha, Hora, Concepto, Descripción, Canal, Banco/Op., Monto, Cajero, Agencia
- [ ] Movimientos anulados se ven **tachados** (strikethrough) en gris
- [ ] El TOTAL solo suma los NO anulados
- [ ] Cambiar entre INGRESO y EGRESO genera archivos distintos (`movimientos_caja_ingresos_*.xlsx` vs `_egresos_*.xlsx`)

### 4. Cuadre de Caja — `/xlsx/cuadre-caja`

**Cómo:** Tab "Cuadre de Caja" → setear fechas opcional → botón "Excel"

- [ ] 4 KPIs: APERTURAS, CUADRES OK, SOBRANTES (S/), FALTANTES (S/)
- [ ] Tabla con: Fecha, Apertura, Cierre, Cajero, Agencia, Estado, Inicial, Ingresos, Egresos, Diferencia, Observación
- [ ] Columna "Estado" con badge color (verde CERRADA, rojo DESCUADRE, ámbar ABIERTA)
- [ ] Columna "Diferencia" con color condicional (verde 0, ámbar +, rojo −)
- [ ] Las horas se muestran como "HH:mm" sin importar si en BD están como TIMESTAMP

### 5. Créditos — `/xlsx/creditos`

**Cómo:** Tab "Créditos" → seleccionar PRE-DESEMBOLSO o DESEMBOLSADOS → botón "Excel"

- [ ] 4 KPIs: N° CRÉDITOS, MONTO COLOCADO, SALDO POR COBRAR, % RECUPERADO
- [ ] Tabla con: Contrato, DNI, Cliente, Producto, Sistema, Frec., Monto, Cuotas, Cuota, F. Desemb., Saldo Capital, Estado
- [ ] Badge "Estado" con color (verde VIGENTE, azul LIQUIDADO, ámbar PRE_DESEMBOLSO, naranja DEVUELTO)
- [ ] El % RECUPERADO es coherente con monto vs saldo
- [ ] Cambiar categoría genera archivos distintos (`creditos_pre_desembolso_*.xlsx` vs `_desembolsados_*.xlsx`)

### 6. Cartera de Clientes — `/xlsx/cartera-clientes`

**Cómo:** Tab "Cartera Clientes" → botón "Exportar Excel"

- [ ] 4 KPIs: CLIENTES ACTIVOS, PRÉSTAMOS VIGENTES, SALDO CAPITAL, MORA TOTAL
- [ ] Tabla con: Cliente, DNI, Asesor, Agencia, Vigentes, Liquidados, Saldo Capital, Cuotas Pendientes, Mora Acum.
- [ ] Solo aparecen clientes con préstamos vigentes
- [ ] El TOTAL al final con fórmulas (no hardcode)

### 7. Historial de Mora — `/xlsx/historial-mora`

**Cómo:** Tab "Historial Mora" → botón "Exportar Excel"

- [ ] 4 KPIs: PRÉSTAMOS EN MORA, DEUDA TOTAL, MORA 31-90 DÍAS, MORA +90 DÍAS
- [ ] Sección "DISTRIBUCIÓN POR RANGO" con barras visuales y porcentajes (fórmulas)
- [ ] Tabla detalle: Contrato, Cliente, DNI, Asesor, Agencia, Cuotas Venc., Días, Total Deuda, Rango
- [ ] Columna "Días" color-coded por severidad
- [ ] Badge "Rango" con color (ámbar/naranja/rojo/rojo oscuro según rango)

### 8. Extornos del Mes — `/xlsx/extornos-mes`

**Cómo:** Tab "Extornos del Mes" → seleccionar año/mes → botón "Excel"

- [ ] 4 KPIs: TOTAL DEL MES, APLICADOS, EN REVISIÓN, RECHAZADOS
- [ ] Tabla con: EXT-ID, Tipo, Categoría, Estado, Monto, Solicitante, Agencia, Solicitado, Aprobador, Decisión
- [ ] Doble badge: Categoría (PRESTAMO/CUADRE/AJUSTE) + Estado (APLICADO/APROBADO/SOLICITADO/RECHAZADO/ANULADO) cada uno con su color
- [ ] Las fechas se muestran sin la parte de hora ("2026-04-15" no "2026-04-15T14:30:00")
- [ ] Cambiar el mes genera archivos con periodo en el nombre (`extornos_2026_04.xlsx`)

---

## Verificación de roles (importante para QA antes de demo)

Salir y entrar con distintos usuarios para confirmar el filtrado por rol:

- [ ] **CAJERO_COBRANZA** ve solo: General, Caja, Cobranza
- [ ] **ANALISTA_CREDITO** ve solo: General, Cobranza, Cartera
- [ ] **SUPERVISOR_CREDITO** ve: General, Caja, Cobranza, Cartera, Comercial
- [ ] **GERENTE_AGENCIA** ve los mismos 5 que el supervisor
- [ ] **GERENTE_GENERAL / ADMINISTRADOR / AUDITOR** ven los 6 grupos completos
- [ ] Click en un grupo → automáticamente se selecciona la primera sub-tab

---

## Verificación de errores comunes

Si alguno de estos pasa, cancelar la demo o tener fallback:

- ❌ **400 Bad Request** al click en "Excel" — revisar logs del backend, probablemente hay un nuevo bug de SQL en QA con datos distintos a dev
- ❌ **El logo no aparece** en el .xlsx — verificar que `src/main/resources/img/logo_crediactiva.png` se copió al JAR correctamente
- ❌ **Los KPIs muestran solo números** sin etiquetas — el `ExcelExportService` o `ExcelStyleHelper` no se reconstruyó (rebuild backend completo)
- ❌ **Filas TOTAL muestran #REF!** — error en cálculo de fila de inicio del SUM
- ❌ **El archivo descargado es 0 bytes** — verificar response headers (Content-Type debe ser `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`)

---

**Tiempo estimado total**: 10–15 min para los 8 reportes + 5 min para verificación de roles
