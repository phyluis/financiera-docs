# Flujo del Préstamo

## Estados y transiciones

```
[ANALISTA]           [COMITÉ]              [CAJERO]
    │                    │                     │
 PENDIENTE ──────► EN_COMITE ──────────► PRE_DESEMBOLSO
    │                    │                     │
 OBSERVADO           APROBADO              VIGENTE ──► LIQUIDADO
    │                    │                     │
 RECHAZADO           RECHAZADO             DEVUELTO (máx 2 veces)
    │                    │                     │
 DEVUELTO            DEVUELTO              CANCELADO (3ra devolución)
```

## Reglas clave

### Evaluación → Comité
- Analista crea evaluación con datos del cliente y condiciones del préstamo
- Supervisor envía a comité cuando está lista
- Máximo 2 devoluciones al analista; a la 3ra se rechaza

### Comité → Desembolso
- Comité vota (mínimo quórum requerido)
- Presidente resuelve en caso de empate
- Al aprobar: se crea préstamo en `PRE_DESEMBOLSO` con cronograma provisional

### Desembolso
- Cajero verifica condiciones y documentos
- Al desembolsar: préstamo pasa a `VIGENTE`, fechas del cronograma se actualizan
- Máximo 2 devoluciones; a la 3ra → `CANCELADO` automático
- Cajero NO puede modificar condiciones, solo verificar

### Post-desembolso
- Evaluación y aprobación quedan en modo SOLO LECTURA
- Solo cobranza puede interactuar con el préstamo (pagos de cuotas)

## Flujo de pago de cuotas

1. Cajero selecciona cuota a pagar del cronograma
2. Sistema calcula mora según días hábiles vencidos
3. Cajero ingresa importe entregado (debe cubrir cuota + mora)
4. Sistema registra pago y actualiza saldo de capital
5. Si todas las cuotas quedan pagadas → préstamo pasa a `LIQUIDADO`
