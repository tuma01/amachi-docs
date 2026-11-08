# ADR-0012: Frontend general para cualquier sector: se extrae de `vitalia-web` cuando haya un segundo consumidor

- **Estado**: Aceptada
- **Fecha**: 2026-11-07
- **Decide**: Juan Amachi
- **Controles ISO 27001**: A.8.9 (gestión de la configuración), A.8.25 (ciclo de vida de desarrollo seguro: la sesión, la reautenticación y el CSRF se escriben una vez)
- **Resume**: `docs/PLATAFORMA.md` §13.2 y §14.2 (etapa 5) de `amachi-platform`

## Contexto

El backend es una plataforma general (`platform-starter`) sobre la que cada sector construye su API. El frontend de Vitalia (`vitalia-web`,
Angular 21) ya contiene, mezclado con pantallas de salud, todo lo que cualquier sector necesita: entrada por BFF/Keycloak sin contraseñas ni
tokens en el navegador, reautenticación al firmar, motor CRUD, cascarón con menú por rol, administración de la plataforma, invitaciones,
privacidad, i18n en cuatro idiomas y los componentes comunes. InverPulse (trading) será el segundo sector y necesitará lo mismo.

## Decisión

1. **Habrá un frontend general**, reutilizable por cualquier sector: una **librería Angular** (workspace) que ofrece el cascarón, la
   sesión, la reautenticación, el motor CRUD, la administración de la plataforma, la privacidad y los componentes comunes. Cada sector es
   una aplicación Angular que la consume y añade **solo** sus rutas, pantallas, menú y textos. No se copia el código entre aplicaciones.
2. **No se extrae todavía.** Con un único consumidor, una abstracción sale de suposiciones. Se extrae **cuando InverPulse (o cualquier
   otro sector) sea el segundo consumidor**, con dos usos reales a la vista; igual que el backend se estabilizó primero con Vitalia y
   después se publicó el `platform-starter`.
3. **Mientras tanto, `vitalia-web` se mantiene separable**: lo general y lo de salud no se mezclan.
   - Lo general vive en `core/`, `platform/`, `shared/` (sin vocabulario clínico) y en el motor CRUD.
   - Lo de salud vive en `tenant/<rol>/` y en los componentes `shared/components` con dominio clínico (inventario en
     `vitalia-web/docs/architecture/GENERAL-VS-SECTOR.md`).
   - Una pieza general **no importa** nada de lo clínico; se comprueba con una regla de dependencias (eslint) cuando se extraiga.
4. **El contrato con la plataforma es el de siempre**: sesión del BFF (`/bff/user`, `/oauth2/authorization/keycloak`, `/bff/logout`),
   cliente OpenAPI generado, errores RFC 9457 y los puntos de extensión del backend (`PublicRoutesContributor`, perfiles por rol,
   catálogo de roles). El frontend de un sector no inventa otra forma de entrar.

## Consecuencias

- Hoy no cambia ninguna dependencia: el trabajo es dejar `vitalia-web` limpio y validado (Playwright por rol) y registrar el inventario.
- Al extraer: el criterio de éxito es que InverPulse arranque con la librería y **solo** pantallas propias; lo que tenga que copiar de
  `vitalia-web` es lo que faltó en la librería.
- Riesgo aceptado: lo que hoy parece general y es de salud se descubre al extraer; el inventario y la regla de dependencias lo acotan.
