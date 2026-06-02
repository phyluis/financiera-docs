# White-label — Marca por cliente

El sistema soporta **múltiples marcas** (clientes) sobre el mismo código base.
La marca NO vive en una rama: es una **configuración de build**. Un solo mainline,
cero divergencia.

| Build | Marca |
|---|---|
| `ng build --configuration production` | **Reelige** (producto base / default) |
| `ng build --configuration credidar` | **CREDIDAR** (cliente) |

> ⚠️ Para desplegar a un cliente hay que usar SU configuración.
> CREDIDAR ya **no** se construye con `production` (eso es Reelige).

---

## Estructura

```
src/brand/
  brand.model.ts        → interface BrandConfig + helpers (brandTitle, applyBrand)
  brand.ts              → marca DEFAULT (Reelige) — la que se reemplaza
  brand.credidar.ts     → marca CREDIDAR
  brand-theme.css       → CSS vars por defecto (fallback Reelige, SSR/primer paint)

public/assets/brands/
  reelige/  { logo.png, mark.png, favicon.png }
  credidar/ { logo.png, mark.png, favicon.png }
```

Cada marca define **texto, assets, estilo de login y colores** en un único
archivo `brand.<cliente>.ts`:

```ts
export const BRAND: BrandConfig = {
  name: 'CREDI',                 // parte principal del nombre
  nameAccent: 'DAR',             // parte resaltada (color de acento)
  tagline: 'Sistema Financiero',
  panelTheme: 'light',           // estilo de la portada del login (ver abajo)
  logo:  'assets/brands/credidar/logo.png',   // lockup login (tema 'light')
  mark:  'assets/brands/credidar/mark.png',   // ícono sidebar
  favicon: 'assets/brands/credidar/favicon.png',
  colors: {
    primary: '#1B4F9C', accent: '#27A844', navy: '#16243B',
    panelBg: '#fdfbee',          // fondo del panel del login (color o gradiente)
    panelText: '#16243B',
  },
};
```

### `panelTheme` — estilo de la portada del login
El login soporta **dos estéticas**, elegidas por marca:

| `panelTheme` | Portada | Marca muestra | Usado por |
|---|---|---|---|
| `'dark'`  | Panel oscuro/degradado, features translúcidas | **ícono (`iconClass`) + nombre + tagline** | Reelige (clásico) |
| `'light'` | Panel claro, features tipo tarjeta | **logo imagen (`logo`)** | CREDIDAR |

- Tema `'dark'`: definir `iconClass` (ej. `'bi-bank2'`) y `colors.panelBg` como
  gradiente; `colors.panelText` claro (`#ffffff`).
- Tema `'light'`: definir `logo` (lockup horizontal) y `colors.panelBg` igual al
  fondo del logo para que no se vea costura; `panelText` oscuro.
- El template (`login.html`) hace `@if (brand.logo)` → imagen, si no → ícono+texto.
  El CSS aplica `.panel-dark` / `.panel-light` según el tema.

### Cómo se aplica
- **Colores** → `applyBrand()` (en `app.ts`) los setea como CSS vars en `:root`
  en runtime (`--brand-primary`, `--brand-accent`, `--brand-navy`,
  `--brand-panel-bg`, `--brand-panel-text`). El login y el sidebar consumen esas vars.
- **Logo / mark / nombre / tagline** → los componentes (`login`, `admin-layout`)
  bindean a `brand.*`.
- **Favicon y título de pestaña** → `app.ts` los aplica en runtime desde `BRAND`.

El reemplazo por cliente se hace con `fileReplacements` en `angular.json`
(solo `.ts`/`.json`, por eso los colores van en el `.ts`, no en un `.css`):

```jsonc
"credidar": {
  "fileReplacements": [
    { "replace": "src/environments/environment.ts", "with": "src/environments/environment.prod.ts" },
    { "replace": "src/brand/brand.ts",              "with": "src/brand/brand.credidar.ts" }
  ],
  "outputHashing": "all"
}
```

---

## Agregar un cliente nuevo (≈10 min)

1. **Assets**: crear `public/assets/brands/<cliente>/` con `logo.png` (lockup
   horizontal), `mark.png` (ícono cuadrado) y `favicon.png` (64×64).
   > Si el logo viene con fondo (jpeg), recortar y dejar el `panelBg` igual a ese
   > fondo para que no se vea costura en el login.
2. **Config**: copiar `brand.credidar.ts` → `brand.<cliente>.ts` y ajustar
   nombre, tagline, rutas de assets y colores.
3. **angular.json**: agregar una configuración `<cliente>` con los
   `fileReplacements` (environment.prod + `brand.<cliente>.ts`).
4. **Build**: `ng build --configuration <cliente>`.

No se toca ningún componente: login, sidebar, favicon y título se adaptan solos.

---

## Despliegue (recordatorio)

```bash
ng build --configuration credidar          # genera dist/financiera-app/browser/
scp -i KEY.pem -r dist/financiera-app/browser/* ubuntu@IP:/home/ubuntu/app/frontend/
# en el servidor:
/home/ubuntu/app/scripts/deploy-frontend.sh
```
(El backend es agnóstico de marca — no cambia entre clientes.)
