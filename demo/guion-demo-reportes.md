# Guión de demo — Módulo Reportes Excel CrediActiva

Guión sugerido para presentar el nuevo módulo de exportación Excel ante usuarios/clientes.
Tiempo estimado: **15–20 min** + Q&A.

---

## Antes de empezar

### Setup del entorno (ya debería estar corriendo)
- [ ] Backend QA en `:8082` (vía `build-calidad.bat` ya ejecutado)
- [ ] `iniciar-ngrok-qa.bat` corriendo → URL pública lista
- [ ] Navegador abierto en la URL ngrok
- [ ] Sesión iniciada como **ADMINISTRADOR** (ve todo)
- [ ] Excel cerrado (para que el usuario vea cómo se abre desde cero)
- [ ] Carpeta de descargas limpia (o tener una vista para que se vea cuando llega el archivo)

### Mensaje inicial (~30 seg)

> "Hoy quiero mostrarles el nuevo módulo de **Reportes** del sistema.
> Lo importante: pasamos de tener archivos planos a tener **reportes profesionales con plantilla institucional**.
> Cada reporte tiene el logo de CrediActiva, indicadores clave arriba, y datos detallados abajo —
> todo listo para imprimir, archivar o compartir con la gerencia."

---

## Parte 1 — Navegación organizada (~2 min)

### Lo que mostrás
1. Click en "Reportes" en el sidebar
2. Mostrá la barra superior con los **6 grupos**: General · Caja · Cobranza · Cartera · Comercial · Gerencial
3. Click en cada grupo y mostrá cómo cambia el sub-menú abajo

### Lo que decís

> "El módulo se organiza en 6 grupos según el área de negocio.
> Antes teníamos 14 reportes en una sola barra — ahora cada grupo tiene los reportes
> que le corresponden: Caja para los cajeros, Cartera para los analistas, Gerencial para la dirección."

### Punto clave a destacar

> "Y muy importante: **cada usuario ve solo los grupos que su rol le permite**.
> Un cajero no ve los reportes gerenciales. Un analista no ve cuadre de caja.
> Eso reduce el ruido y mantiene la información compartimentada."

---

## Parte 2 — Reporte estrella: Cartera en Mora (~4 min)

Este es el reporte más visual e impactante. Demostralo primero.

### Lo que mostrás

1. Grupo **Cobranza** → tab "Cartera en Mora"
2. La pantalla muestra los KPIs y la tabla en Angular
3. Click en el botón verde **"Exportar Excel"** (arriba derecha)
4. Se descarga el `.xlsx`
5. Abrirlo en Excel — pausa de unos segundos para que el usuario vea el resultado

### Lo que decís (mientras se descarga)

> "Este es uno de los reportes que más usan los analistas de cobranza diariamente.
> Cuando hago click en 'Exportar Excel', el sistema arma el reporte en el servidor —
> con los datos del momento, no una plantilla pre-armada."

### Cuando se abre el Excel

Pausa. Que el usuario vea. Después señalá:

1. **El logo de CrediActiva** arriba a la izquierda
2. **El título y la fecha** alineados a la derecha
3. **La línea verde divisoria**
4. **Los 5 indicadores clave** con sus etiquetas: Total en Mora, Préstamos Afectados, Mayor Atraso, Cuotas Vencidas, % Mora
5. **La distribución por rango** con barras visuales y porcentajes
6. **La tabla detallada** con badges de color en la columna "Rango"
7. **La fila TOTAL** al final (señalá que los números son fórmulas, no copiar-pegar)

### Punto clave

> "Lo importante: estos NO son archivos generados desde Excel.
> El backend usa **Apache POI** y arma el archivo con el logo, los colores corporativos,
> las fórmulas reales… todo desde Java en menos de un segundo."

---

## Parte 3 — Tour rápido por los reportes (~6 min)

Dependiendo del tiempo y la audiencia, mostrá 2 o 3 más:

### Opción A — Audiencia operativa (cajeros, analistas)

**Mostrar: Historial de Pagos + Movimientos de Caja + Cuadre de Caja**

> "Este es para el cajero — todos los pagos del día con el comportamiento (en fecha / con atraso)…"
> "Este es el cuadre de caja al cierre — sobrantes y faltantes con el responsable…"

### Opción B — Audiencia gerencial (gerentes, directorio)

**Mostrar: Cartera de Clientes + Créditos Desembolsados + Extornos del Mes**

> "Esto es la cartera completa de clientes activos — para análisis estratégico…"
> "Los créditos del período con el % de recuperación calculado…"
> "Los extornos del mes con el flujo de aprobación — auditoría completa…"

### Opción C — Audiencia mixta

Mostrá **Cartera en Mora** + **1 más rápido** (Movimientos o Extornos) + cierra con consolidado.

---

## Parte 4 — El detalle técnico (opcional, ~2 min)

Si te preguntan o si la audiencia es técnica, mencioná:

> "Cada reporte tiene 5 capas:
> 1. **Endpoint REST** que recibe los filtros (fecha, búsqueda, agencia)
> 2. **Servicio Java** que consulta la BD con los mismos filtros que ven en pantalla
> 3. **Plantilla unificada** que arma el header con logo + título + KPIs
> 4. **Apache POI** convierte todo a archivo XLSX nativo
> 5. **El navegador** lo descarga directamente — sin abrir nada en el servidor"

> "Y todo eso garantiza que el reporte que descargás es exactamente lo mismo que ves en pantalla."

---

## Parte 5 — Cierre (~1 min)

> "Resumiendo:
> ✓ **8 reportes** ya disponibles para descargar en Excel
> ✓ **Plantilla unificada** con logo CrediActiva y formato profesional
> ✓ **Filtrado por rol** — cada usuario ve lo suyo
> ✓ **Datos en tiempo real** — siempre lo último, nunca cache
> ✓ **5 reportes más** en camino (Fase 3): Meta vs Ejecutado, Mora por Analista, Flujo de Cobros, Cartera por Asesor, Consolidado por Agencia
>
> ¿Preguntas?"

---

## Plan B — Si algo falla durante la demo

### Si el ngrok está caído
- Tenés `localhost:8082` corriendo localmente — comparte tu pantalla en lugar de mandar URL
- O cambiá a desarrollo `:8081` (los reportes funcionan igual)

### Si un Excel específico falla
- **NO te quedes pegado** — pasá al siguiente. Decí: "Este reporte tiene un caso edge que estamos puliendo, sigamos con…"
- Tené **screenshots** previos de los 8 Excel ya generados como backup en una carpeta `screenshots-demo/`

### Si el servidor cae
- `servicio-qa.bat` opción [3] reinicia el servicio
- Mientras se levanta, mostrá los **prototipos Excel** estáticos en `public/prototipos/excel/` (los `.xlsx` que generamos con Node.js — son visualmente idénticos al backend)

### Si la audiencia te corta con preguntas técnicas
- "Buena pregunta — al final lo vemos en detalle, así no perdemos el flujo"
- Apuntá la pregunta y volvé al guión

---

## Preguntas frecuentes anticipadas

**P: ¿Se puede personalizar el logo o los colores?**
> R: Sí, está centralizado en `ExcelStyleHelper`. Cualquier cambio futuro afecta a los 8 reportes simultáneamente.

**P: ¿Puedo programar reportes automáticos diarios/semanales?**
> R: Funcionalmente sí; el endpoint ya expone los datos. Falta el scheduler — está en backlog para Fase 4.

**P: ¿Funciona en celular?**
> R: La descarga sí. Para abrir el Excel necesitás una app que soporte XLSX (Google Sheets, Excel mobile, WPS).

**P: ¿Y si quiero PDF en lugar de Excel?**
> R: Para Cartera en Mora ya hay PDF disponible (botón al lado del Excel). Los demás están en roadmap.

**P: ¿Los datos sensibles se ofuscan?**
> R: Cada reporte tiene su propio DTO precisamente para que en el futuro podamos ocultar campos según el rol que descarga. Hoy todos los datos van completos.

**P: ¿Qué pasa si dos usuarios piden el mismo reporte al mismo tiempo?**
> R: Cada request es independiente — el servidor genera un Excel nuevo cada vez. No hay race conditions ni archivos compartidos.

---

## Archivos relacionados

- **Smoke test pre-demo**: [smoke-test-reportes-excel.md](./smoke-test-reportes-excel.md)
- **Prototipos visuales**: `financiera-frontend/public/prototipos/excel/`
- **PR backend**: phyluis/financiera-backend#1
- **PR frontend**: phyluis/financiera-frontend#2

**Tiempo total estimado: 15–20 min de demo + 5–10 min de Q&A**
