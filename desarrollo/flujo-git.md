# Flujo Git

## Trabajo diario

Todo el desarrollo nuevo se hace en la rama `desarrollo`:

```bash
# Clonar (primera vez)
git clone -b desarrollo https://github.com/phyluis/financiera-backend.git
git clone -b desarrollo https://github.com/phyluis/financiera-frontend.git

# Flujo normal
git add .
git commit -m "feat(modulo): descripcion del cambio"
git push origin desarrollo
```

## Convención de commits

```
tipo(alcance): descripción corta

feat(back): nuevo endpoint de pagos
feat(front): componente de cobranza
feat(back+front): módulo completo de garantías
fix(back): corregir cálculo de mora
fix(front): validación de formulario
chore(config): actualizar propiedades de entorno
docs: actualizar guía de despliegue
```

## Pase de cambios entre ramas

**Regla fundamental:** Nunca merge automático. Siempre PR con revisión línea por línea.

```
desarrollo → calidad:     revisión de integración y pruebas
calidad    → produccion:  revisión con criterios de negocio
produccion → main:        revisión de certificación final
```

### Pasos para promover cambios

1. Crear Pull Request en GitHub desde la rama origen a la rama destino
2. Revisar cada archivo cambiado línea por línea
3. Aprobar y hacer merge solo si todo está correcto
4. Actualizar documentación si hay cambios de negocio
