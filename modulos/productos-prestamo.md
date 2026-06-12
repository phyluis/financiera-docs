# Productos de Préstamo — Guía detallada con ejemplos

> Documento de referencia del módulo de **Productos de Crédito**: qué es un producto,
> cómo se configura, los **3 tipos de cálculo** de cuotas, frecuencias, mora, gracia,
> redondeo y límites — con **ejemplos numéricos exactos** (replican el motor
> `ProductoCalculator`).
>
> Código de referencia: `ProductoCredito.java` (modelo) · `ProductoCalculator.java` (motor).

---

## 1. ¿Qué es un producto de crédito?

Un **producto** es una plantilla de condiciones de préstamo. Al crear un préstamo se elige
un producto, y de él se heredan: tipo de cálculo, frecuencia, tasa, mora, gracia y límites.
Las políticas relevantes se **copian (snapshot)** al préstamo al desembolsar, de modo que
un cambio posterior en el producto **no** altera préstamos ya emitidos.

### Campos configurables (`productos_credito`)

| Campo | Descripción |
|---|---|
| `nombre` / `descripcion` | Identificación del producto (ej. "Diario CrediActiva", "Mensual Francés"). |
| `tipo_calculo` | **SIMPLE_FLAT · SIMPLE_SALDO · FRANCES** (ver §3). |
| `frecuencia_pago` | **DIARIO · SEMANAL · QUINCENAL · MENSUAL** (ver §4). |
| `tasa_interes` | Tasa por período (interpretación según tipo de cálculo). |
| `tasa_min` / `tasa_max` | Rango permitido al personalizar la tasa. |
| `tasa_mora` | Tasa de mora **mensual** % (default 2.0). Solo si `tipo_mora = PORCENTAJE`. |
| `tipo_mora` | **PORCENTAJE · FIJO · FIJO_DIARIO_HABILES** (ver §6). |
| `monto_mora_fijo` | Monto fijo de mora (para FIJO / FIJO_DIARIO_HABILES). |
| `mora_desde_fecha_fin` | Para diarios: la mora corre desde la **fecha fin** del préstamo (ver §6.4). |
| `plazo_min_cuotas` / `plazo_max_cuotas` | Rango del número de cuotas. |
| `monto_min` / `monto_max` | Rango del monto a prestar. |
| `periodo_gracia` / `tipo_gracia` | Cuotas de gracia y su modo: **NINGUNA · PARCIAL · TOTAL** (ver §5). |
| `activo` | Si el producto está disponible para nuevos préstamos. |

---

## 2. Reglas transversales

- **Redondeo de cuota a 0.10 (hacia arriba):** la cuota total se sube al múltiplo de `0.10`
  superior; el delta se suma al **interés**. La **amortización y el saldo quedan exactos**
  (ver §7). Aplica a los 3 tipos de cálculo.
- **Fechas en día hábil:** las fechas de vencimiento que caen en sábado/domingo o feriado se
  mueven al **siguiente día hábil**. En DIARIO se evita además que dos cuotas colisionen en la
  misma fecha (`DiasHabilesUtil`).
- **Última cuota absorbe residuos:** para que el saldo cierre exactamente en 0, la última
  cuota normal ajusta amortización/interés (a favor de la empresa).
- **Período de gracia:** las primeras `periodo_gracia` cuotas se tratan según `tipo_gracia`.

---

## 3. Tipos de cálculo (con ejemplos)

### 3.1 SIMPLE_FLAT — interés plano (flat)

El interés total es un cargo **fijo** sobre el capital, **distribuido en partes iguales** entre
las cuotas. La amortización también es igual en cada cuota. La cuota es **constante**.

**Fórmula:**
```
mesesEquiv   = cuotasNormales / cuotasPorMes(frecuencia)
interésTotal = capital × (tasa/100) × mesesEquiv
interésCuota = interésTotal / cuotasNormales      (la última absorbe el residuo)
amortCuota   = capital / cuotasNormales
```
Donde `cuotasPorMes`: **DIARIO = 26**, SEMANAL = 4, QUINCENAL = 2, MENSUAL = 1.

**Ejemplo A — S/ 1,000 · 5 cuotas MENSUAL · tasa 5%**
`mesesEquiv = 5/1 = 5` → `interésTotal = 1000 × 0.05 × 5 = 250`

| Cuota | Amort. | Interés | Cuota | Saldo |
|--:|--:|--:|--:|--:|
| 1 | 200.00 | 50.00 | 250.00 | 800.00 |
| 2 | 200.00 | 50.00 | 250.00 | 600.00 |
| 3 | 200.00 | 50.00 | 250.00 | 400.00 |
| 4 | 200.00 | 50.00 | 250.00 | 200.00 |
| 5 | 200.00 | 50.00 | 250.00 | 0.00 |
| **Total** | **1,000** | **250.00** | **1,250.00** | |

**Ejemplo B — caso DIARIO real (S/ 300 · 20 cuotas DIARIO · 10%)**
`mesesEquiv = 20/26 = 0.769` → `interésTotal ≈ 23.08`; amort `15.00`, interés `1.15` por cuota
(la última `1.23`). Cuota diaria **16.15** (última 16.23). Total a pagar **323.08**.
*(Es el formato del cronograma de pagos diario de CrediActiva.)*

> **Uso típico:** préstamos **diarios** y de campaña, donde el cliente entiende "pago fijo por día".

---

### 3.2 SIMPLE_SALDO — interés sobre saldo

El interés de cada cuota se calcula sobre el **saldo pendiente** (va **bajando**). La
amortización es **fija**; por eso la **cuota decrece** cuota a cuota.

**Fórmula (por cuota):**
```
interés = saldoPendiente × (tasa/100)
amort   = capital / cuotasNormales      (fija)
cuota   = amort + interés
```

**Ejemplo — S/ 1,000 · 5 cuotas MENSUAL · tasa 5% por período**

| Cuota | Amort. | Interés | Cuota | Saldo |
|--:|--:|--:|--:|--:|
| 1 | 200.00 | 50.00 | 250.00 | 800.00 |
| 2 | 200.00 | 40.00 | 240.00 | 600.00 |
| 3 | 200.00 | 30.00 | 230.00 | 400.00 |
| 4 | 200.00 | 20.00 | 220.00 | 200.00 |
| 5 | 200.00 | 10.00 | 210.00 | 0.00 |
| **Total** | **1,000** | **150.00** | **1,150.00** | |

> **Uso típico:** clientes que amortizan parejo y pagan **menos interés** que en flat (el interés
> sigue al saldo). Cuota inicial más alta, va aliviando.

---

### 3.3 FRANCES — cuota constante (sistema francés)

Cuota **fija** (PMT). El interés se calcula sobre el saldo (decrece) y la amortización
**crece** cuota a cuota. La tasa ingresada se interpreta como **TEA** y se convierte a tasa
mensual efectiva: `TEM = (1 + TEA)^(1/12) − 1`.

**Fórmula:**
```
TEM      = (1 + TEA)^(1/12) − 1
cuotaFija = capital × TEM / (1 − (1 + TEM)^(−cuotasNormales))
interés  = saldo × TEM ;  amort = cuotaFija − interés
```

**Ejemplo — S/ 1,000 · 5 cuotas MENSUAL · TEA 60%** → `TEM ≈ 3.98%`, cuota fija ≈ **224.60**

| Cuota | Amort. | Interés | Cuota | Saldo |
|--:|--:|--:|--:|--:|
| 1 | 184.65 | 39.95 | 224.60 | 815.35 |
| 2 | 192.02 | 32.58 | 224.60 | 623.33 |
| 3 | 199.69 | 24.91 | 224.60 | 423.64 |
| 4 | 207.67 | 16.93 | 224.60 | 215.97 |
| 5 | 215.97 | 8.63 | 224.60 | 0.00 |
| **Total** | **1,000** | **123.00** | **1,123.00** | |

> **Uso típico:** préstamos **mensuales** formales; cuota pareja y predecible para el cliente.

### Comparación rápida (mismo capital 1,000 / 5 cuotas)

| Tipo | Cuota | Total interés | Característica |
|---|---|--:|---|
| SIMPLE_FLAT (5%) | constante 250 | 250.00 | Interés plano, el más caro |
| SIMPLE_SALDO (5%) | decrece 250→210 | 150.00 | Interés sobre saldo |
| FRANCES (TEA 60%) | constante 224.60 | 123.00 | Cuota fija, amortización creciente |

*(Los totales no son comparables 1:1 porque la tasa significa cosas distintas en cada modelo;
la tabla ilustra el **comportamiento**, no cuál es "mejor".)*

---

## 4. Frecuencias de pago

| Frecuencia | Intervalo entre cuotas | `cuotasPorMes` (flat) |
|---|---|--:|
| DIARIO | +1 día hábil (sin colisiones) | 26 |
| SEMANAL | +7 días | 4 |
| QUINCENAL | +15 días | 2 |
| MENSUAL | +1 mes | 1 |

Toda fecha que cae en fin de semana o feriado se corre al **siguiente día hábil**.

---

## 5. Período de gracia

Las primeras `periodo_gracia` cuotas se tratan según `tipo_gracia`:

| Tipo | Comportamiento en las cuotas de gracia |
|---|---|
| **NINGUNA** | Sin gracia: cálculo normal desde la cuota 1. |
| **PARCIAL** | Solo se cobra **interés** (sin amortizar). El saldo no baja en esas cuotas. |
| **TOTAL** | **Sin pago** (cuota 0). El capital se difiere a las cuotas normales siguientes. |

Las `cuotasNormales = nº cuotas − gracia` reparten el capital y el interés.

---

## 6. Mora (tipos y ejemplos)

La mora se calcula al cobrar una cuota vencida. Hay 3 tipos:

### 6.1 PORCENTAJE (default)
```
mora = capitalDeLaCuota × (tasaMora/100/30) × díasVencidos
```
La `tasaMora` es **mensual** (ej. 2% → 2.0). Se divide entre 30 para obtener tasa diaria.
> En los **listados de cobranza** se cuentan **días hábiles** (con gracia y feriados); en el
> **cobro en caja** se cuentan días calendario. *(Divergencia histórica documentada; el motor
> `MoraCalculator` preserva ambas reglas.)*

**Ejemplo:** cuota con capital 100, tasaMora 2%, 5 días → `100 × (2/100/30) × 5 = 0.33`.

### 6.2 FIJO
Un **cargo único** `monto_mora_fijo` por cuota vencida, sin importar los días.
**Ejemplo:** `monto_mora_fijo = 5.00` → la cuota vencida suma S/ 5.00 de mora (una sola vez).

### 6.3 FIJO_DIARIO_HABILES
```
mora = monto_mora_fijo × díasHábilesVencidos
```
**Ejemplo:** `monto_mora_fijo = 1.00`, 3 días hábiles vencidos → mora S/ 3.00.

### 6.4 Mora desde la fecha fin (solo DIARIO, flag del producto)
Cuando `mora_desde_fecha_fin = true`, para préstamos **DIARIOS** la mora **no** corre por cada
cuota vencida, sino desde la **fecha fin** del préstamo (la última cuota), aplicada al capital
de la cuota.

**Ejemplo:** producto con 0.5 de mora (FIJO_DIARIO_HABILES, sobre 5 cuotas):
`0.5 × 5 × 1 día = 2.50`; a los 2 días `5.00`; a los 3 días `7.50`; **dentro del plazo = 0**.

> El flag se **copia al préstamo** al desembolsar (snapshot); cambiarlo en el producto no
> afecta préstamos ya emitidos.

---

## 7. Redondeo de cuota a 0.10

Tras calcular `cuota = amort + interés`, se sube al **múltiplo de 0.10 superior**:
```
cuota10 = ceil(cuota / 0.10) × 0.10
interés += (cuota10 − cuota)      // el delta va al interés
cuota    = cuota10                // amortización y saldo quedan EXACTOS
```
**Ejemplo:** cuota calculada 16.147 → cuota cobrada **16.20**; los 0.053 se suman al interés.
Esto vale para el cronograma real, la evaluación y el simulador.

---

## 8. Límites y validaciones

- **Monto:** `monto_min ≤ monto ≤ monto_max`.
- **Plazo:** `plazo_min_cuotas ≤ nº cuotas ≤ plazo_max_cuotas`
  (y el límite por configuración de aprobación `plazoMaximoCuotas`).
- **Tasa:** al personalizar, se acota a `[tasa_min, tasa_max]`.

---

## 9. Cargo por desembolso

Independiente del cálculo de cuotas, el sistema puede aplicar un **cargo por desembolso**
(parámetro `CARGO_DESEMBOLSO`, base configurable). El cajero puede **ajustarlo en el modal**
al desembolsar y elegir cómo se cobra:
- **DESCONTADO** — se resta del monto entregado al cliente *(previa consulta al cliente)*.
- **EFECTIVO** — el cliente recibe el monto completo y paga el cargo aparte en caja.

El cargo **no** modifica el monto a desembolsar ni el cronograma; se registra como ingreso de caja.

---

## 10. Resumen — ¿qué producto para qué caso?

| Necesidad | Producto sugerido |
|---|---|
| Cobranza diaria, pago fijo simple | **SIMPLE_FLAT · DIARIO** |
| Cliente quiere pagar menos interés | **SIMPLE_SALDO** |
| Préstamo mensual formal, cuota pareja | **FRANCES · MENSUAL** |
| Negocio estacional (arranque suave) | cualquiera + **gracia PARCIAL/TOTAL** |
| Diario con mora solo si no liquida al final | DIARIO + **mora_desde_fecha_fin** |

---

*Ejemplos verificados contra el motor `ProductoCalculator` (redondeo a 0.10 incluido).*
