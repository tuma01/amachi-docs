# Plantilla: checklist de un feature nuevo

> Copiar en el PR o en el plan del feature y marcar cada punto. Conserva su número de sección (§11).

## 11. Checklist de un feature nuevo

1. Referencia profesional elegida y ejemplo concreto mostrado (pantalla/URL) **antes** de codificar.
2. Entidad (tenant o global) + migración (`CREATE`, tabla `_AUD`) + DTO/SearchDto + mapper (FK resueltas en el servicio)
   + repository (projections/`@EntityGraph`) + specification (`TextSearch`) + service + utility de reglas.
3. Api + Controller con `CAN_*`, 201 + `Location`, `PageResponseDto.of`; mensajes en todos los idiomas.
4. JUnit (servicio, reglas, controlador) → contrato OpenAPI → `gen-api`.
5. Frontend con el motor CRUD o el template del caso; i18n en todos los idiomas; menú.
6. Vitest por regla; Cucumber por caso real y 403; Playwright con acciones reales por rol; plan manual recorrido.
7. Este documento (o el de la aplicación) actualizado en la misma sesión.
