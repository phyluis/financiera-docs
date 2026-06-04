# Patrón de Validación de Formularios (Angular Reactive Forms)

## Principios

| Regla | Detalle |
|---|---|
| Los errores **solo aparecen al hacer clic en Guardar/Registrar** | Nunca al perder el foco (no usar `touched`) |
| Los mensajes de **formato/hint** se muestran en azul debajo del campo | Solo visibles en `modoEdicion`, nunca en color rojo |
| Los mensajes de **error de validación** se muestran en rojo | Solo cuando `submitIntentado = true` y el campo es inválido |
| **Nunca usar** `<div class="invalid-feedback">` de Bootstrap | Bootstrap lo activa también por la pseudo-clase `:invalid` del navegador |
| Agregar `novalidate` al `<form>` | Desactiva la validación nativa del navegador |
| Campos numéricos opcionales usan `null` como valor inicial | El `0` pasa `Validators.required`, el `null` no |

---

## 1. TypeScript — Propiedades y métodos obligatorios

```typescript
// ── Flag de intento de envío ──────────────────────────
submitIntentado = false;

// Resetear al abrir/cambiar formulario (en configurarModo o ngOnChanges)
private configurarModo(): void {
  this.submitIntentado = false;
  // ... resto del método
}

// Activar al hacer submit
onSubmit(): void {
  if (!this.modoEdicion) return;
  this.submitIntentado = true;
  if (this.form.invalid) return;
  // ... guardar
}

// Helper — campo inválido (depende de submitIntentado, NO de touched)
isInvalid(campo: string): boolean {
  const c = this.form.get(campo);
  return !!(this.submitIntentado && c?.invalid);
}

// Helper — error específico de un campo
hasError(campo: string, error: string): boolean {
  const c = this.form.get(campo);
  return !!(this.submitIntentado && c?.hasError(error));
}
```

---

## 2. CSS — Clases obligatorias del componente

Agregar en el `.css` de cada componente que use este patrón:

```css
/* Hint informativo — azul */
.field-hint {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  margin-top: 0.35rem;
  padding: 0.35rem 0.65rem;
  background: #e0f0fb;
  border-radius: 6px;
  font-size: 0.78rem;
  color: #1a6fa8;
}

/* Mensaje de error de validación — rojo */
.field-error {
  font-size: 0.8rem;
  color: #dc3545;
  margin-top: 0.25rem;
}
```

---

## 3. Validadores reutilizables

### Campos de texto

```typescript
// Solo letras (tildes, ñ, espacios y guión) — campos REQUERIDOS como nombre, ap. paterno
private readonly SOLO_LETRAS = Validators.pattern('^[A-Za-záéíóúÁÉÍÓÚñÑüÜ\\s\'-]+$');

// Solo letras para campos OPCIONALES — valida solo si el campo tiene valor
// (Validators.pattern() de Angular devuelve null en cadenas vacías, pero
//  este validador también acepta espacios en blanco sin error)
private soloLetrasOpcional(control: AbstractControl) {
  if (!control.value || control.value.trim() === '') return null;
  return /^[A-Za-záéíóúÁÉÍÓÚñÑüÜ\s'-]+$/.test(control.value) ? null : { pattern: true };
}

// Uso en initForm():
firstName:        ['', [Validators.required, this.SOLO_LETRAS]],       // requerido
maternalLastName: ['', [Validators.required, this.soloLetrasOpcional.bind(this)]], // opcional con patrón
```

### Campos numéricos

```typescript
// ⚠️ IMPORTANTE: usar null como valor inicial, NUNCA 0
// El 0 pasa Validators.required y nunca se marca como inválido en formularios nuevos

monthlyIncome:   [null, [Validators.required, Validators.min(0)]],
monthlyExpenses: [null, [Validators.required, Validators.min(0)]],

// En setDefaults() NO incluir estos campos — dejarlos en null para que
// el usuario los llene obligatoriamente
```

### Teléfono peruano

```typescript
// 9 dígitos comenzando con 9
phoneNumber: ['', [Validators.required, Validators.pattern('^9[0-9]{8}$')]]
// En el input: maxlength="9"
```

### Email

```typescript
// Usar type="text" en el input, NO type="email"
// (type="email" activa validación nativa del navegador que ignora novalidate en algunos browsers)
email: ['', Validators.email]
```

### Número de documento (validador dinámico por tipo)

```typescript
// Valor inicial — DNI por defecto
dniCuit: ['', [Validators.required, Validators.pattern('^[0-9]{8}$')]]

// Método que se suscribe a documentType.valueChanges en initForm():
private actualizarValidadoresDocumento(tipo: string): void {
  const ctrl = this.form.get('dniCuit');
  if (!ctrl) return;
  const validadores: Record<string, ValidatorFn[]> = {
    'DNI':       [Validators.required, Validators.pattern('^[0-9]{8}$')],
    'RUC':       [Validators.required, Validators.pattern('^[0-9]{11}$')],
    'PASAPORTE': [Validators.required, Validators.pattern('^[A-Za-z0-9]{6,12}$')],
    'CE':        [Validators.required, Validators.pattern('^[A-Za-z0-9]{8,12}$')],
  };
  ctrl.setValidators(validadores[tipo] ?? [Validators.required]);
  ctrl.updateValueAndValidity();
}

// En initForm() después de construir el FormGroup:
this.form.get('documentType')!.valueChanges.subscribe(tipo => {
  this.actualizarValidadoresDocumento(tipo);
});

// Getters de apoyo para el template:
get docLabel(): string { ... }
get docPlaceholder(): string { ... }
get docErrorMsg(): string { ... }
```

### FormArray con mínimo un elemento

```typescript
// En initForm():
referencias: this.fb.array([], this.minimoUnaReferencia)

private minimoUnaReferencia(control: AbstractControl) {
  const arr = control as FormArray;
  return arr.length >= 1 ? null : { minimoUno: true };
}

// Getter para el template:
get sinReferencias(): boolean {
  return this.submitIntentado && this.referencias.invalid;
}

// En onSubmit():
this.referencias.markAsTouched();
```

### Checkbox requerido

```typescript
queryConsent: [false, Validators.requiredTrue]
```

---

## 4. HTML — Plantillas de campo

### Campo de texto requerido (con o sin patrón)

```html
<div class="col-md-4">
  <label class="form-label">Nombre del campo <span class="text-danger">(*)</span></label>
  <input type="text" class="form-control"
    [class.is-invalid]="isInvalid('campo')"
    formControlName="campo"
    placeholder="Ej: valor esperado">
  @if (isInvalid('campo')) {
    <div class="field-error">
      @if (hasError('campo', 'required')) { Este campo es obligatorio. }
      @else { Solo se permiten letras y espacios. }
    </div>
  }
</div>
```

### Campo con hint informativo

```html
<div class="col-md-4">
  <label class="form-label">Campo <span class="text-danger">(*)</span></label>
  <input type="text" class="form-control"
    [class.is-invalid]="isInvalid('campo')"
    formControlName="campo"
    placeholder="Ej: 987654321"
    maxlength="9">
  @if (isInvalid('campo')) {
    <div class="field-error">
      @if (hasError('campo', 'required')) { Este campo es obligatorio. }
      @else { Formato inválido. }
    </div>
  }
  @if (modoEdicion) {
    <div class="field-hint">
      <i class="bi bi-info-circle"></i>Descripción del formato esperado.
    </div>
  }
</div>
```

### Campo numérico

```html
<div class="col-md-6">
  <label class="form-label">Monto <span class="text-danger">(*)</span></label>
  <input type="number" class="form-control"
    [class.is-invalid]="isInvalid('monto')"
    formControlName="monto">
  @if (isInvalid('monto')) {
    <div class="field-error">Ingresa el monto.</div>
  }
</div>
```

### Email

```html
<div class="col-md-4">
  <label class="form-label">Email</label>
  <input type="text" class="form-control"
    [class.is-invalid]="isInvalid('email')"
    formControlName="email"
    placeholder="ejemplo@correo.com">
  @if (isInvalid('email')) {
    <div class="field-error">Correo electrónico inválido.</div>
  }
  @if (modoEdicion) {
    <div class="field-hint">
      <i class="bi bi-info-circle"></i>Formato: ejemplo&#64;correo.com
    </div>
  }
</div>
```

### FormArray con mínimo un elemento

```html
@if (referencias.length === 0) {
  <div class="empty-refs" [class.border-danger]="sinReferencias">
    <i class="bi bi-telephone-x"></i>
    <span>Sin referencias agregadas</span>
  </div>
  @if (sinReferencias) {
    <div class="field-error mt-1">
      <i class="bi bi-exclamation-circle me-1"></i>
      Debe agregar al menos una referencia de contacto.
    </div>
  }
}
```

### Checkbox requerido

```html
<div class="consent-check" [class.border-danger]="isInvalid('queryConsent')">
  <input class="form-check-input" type="checkbox" id="consent" formControlName="queryConsent">
  <label for="consent">
    Texto del consentimiento. <span class="text-danger">(*)</span>
  </label>
</div>
@if (isInvalid('queryConsent')) {
  <div class="field-error mt-1">
    <i class="bi bi-exclamation-circle me-1"></i>
    Debe aceptar la autorización para continuar.
  </div>
}
```

---

## 5. Checklist para aplicar en un formulario nuevo

- [ ] Agregar `submitIntentado = false` como propiedad
- [ ] Agregar `isInvalid()` y `hasError()` como helpers
- [ ] Resetear `submitIntentado = false` en `configurarModo()` / `ngOnChanges`
- [ ] Activar `submitIntentado = true` al inicio de `onSubmit()`
- [ ] Agregar `novalidate` al `<form>`
- [ ] Usar `type="text"` en campos email (no `type="email"`)
- [ ] Usar `null` como valor inicial en campos numéricos requeridos (nunca `0`)
- [ ] Usar `soloLetrasOpcional` en campos de texto opcionales con patrón
- [ ] Usar `SOLO_LETRAS` + `Validators.required` en campos de texto requeridos
- [ ] Reemplazar todo `invalid-feedback` por `@if (isInvalid(...)) { <div class="field-error"> }`
- [ ] Reemplazar hints grises por `@if (modoEdicion) { <div class="field-hint"> }`
- [ ] Agregar `.field-hint` y `.field-error` al CSS del componente

---

## 6. Formularios pendientes de aplicar este patrón

- [x] `ui-form-rgusuarios` — Registro / edición de usuarios
- [ ] Formulario de préstamos (cuando se implemente)
- [ ] Formulario de cobranza (cuando se implemente)
