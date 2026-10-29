# ADR-0006: Keycloak con un realm y Organizations, contraseña solo en Keycloak, BFF aparte

- **Estado**: Aceptada
- **Fecha**: 2026-10-28
- **Decide**: Juan Amachi
- **Completa**: ADR-0003 (Keycloak como proveedor de identidad)

## Contexto

La ADR-0003 decidió usar Keycloak y el patrón BFF. Antes de implementarlo (etapa 3 de la plataforma, SEG2–4) quedan
cuatro decisiones de diseño:

- cómo se modelan las organizaciones (tenants) en Keycloak;
- por dónde pasa la contraseña de quien acepta una invitación;
- si se conserva el autorregistro público `/auth/register`;
- dónde vive el BFF.

Hoy la plataforma autentica por su cuenta: login propio, JWT HS256, refresh en la cookie `amachi_rt`, lista negra de
tokens, activación, reset y cambio de contraseña, y política de contraseñas. La autorización (roles por organización
en `UserTenantRole`, nivel de administrador, `@PreAuthorize`, `@TenantId`) es propia y está bien resuelta.

## Opciones consideradas

1. **Organizaciones**: (a) un realm `amachi` con una *Organization* de Keycloak 26 por tenant; (b) un realm por
   tenant: aislamiento y configuración propia por cliente, pero una persona en dos organizaciones tiene dos cuentas,
   hay un issuer por realm y cada alta crea un realm.
2. **Contraseña al aceptar una invitación**: (a) la pone en Keycloak (acción requerida); (b) la recibe la pantalla
   de la plataforma y la envía a Keycloak por la Admin API.
3. **Autorregistro**: (a) quitarlo; (b) mantenerlo delegado en la pantalla de registro de Keycloak.
4. **BFF**: (a) módulo aparte `platform-bff`; (b) dentro de la API.

## Decisión

1. **Un realm `amachi` + Organizations** (1a). Una persona es una sola cuenta aunque trabaje en varias
   organizaciones, como hoy en la plataforma (`User` y `Person` globales). Un solo issuer para la API.
2. **La contraseña se crea en Keycloak** (2a). La plataforma acepta la invitación (datos de perfil), crea el usuario
   en Keycloak sin contraseña y Keycloak le pide crearla, y configurar MFA si es administrador. La contraseña nunca
   pasa por la plataforma.
3. **Sin autorregistro** (3a). La plataforma es B2B: se entra por invitación de una organización. Si un sector
   necesita alta pública, se habilita en Keycloak para su caso.
4. **BFF en un módulo aparte** (4a): `platform-bff`, en el mismo repositorio, desplegable propio (Spring Cloud Gateway
   + `oauth2Login` + Spring Session en la base). La API queda como *resource server* puro, reutilizable por otros
   clientes.

**Fuente única de la autorización**: la plataforma. El token de Keycloak dice **quién es** (`sub`, email) y **en qué
organización entró**; los roles se leen de `UserTenantRole` en cada petición (un rol revocado deja de valer al
instante y no se duplica en Keycloak).

## Consecuencias

- Se eliminan de la plataforma: login, emisión y refresco de JWT, lista negra, activación, reset y cambio de
  contraseña, política de contraseñas, `/auth/register` y sus tablas (`AUT_REFRESH_TOKEN`, `AUT_BLACKLISTED_TOKEN`,
  `AUT_PASSWORD_RESET_TOKEN`, la columna de contraseña de `AUT_USER`), y el secreto `JWT_SECRET`.
- `AUT_USER` guarda el id de Keycloak (`sub`). La baja de una cuenta también desactiva al usuario en Keycloak.
- El realm se versiona como código en el repositorio; Keycloak corre en Testcontainers para la suite y en el
  `docker compose` de desarrollo. La suite obtiene tokens con un cliente que solo existe en el realm de test.
- Implementación por pasos (`amachi-platform/docs/PLATAFORMA.md` §14.3, SEG2–4): Keycloak en paralelo con el login
  actual; después invitaciones y cuentas vía Keycloak y borrado del login propio; MFA, bloqueo y política (SEG3); BFF
  (SEG4).
- El frontend de la etapa 5 no maneja tokens: usa la cookie del BFF.
- Vitalia adopta lo mismo al migrar a la plataforma; sus contraseñas BCrypt necesitan una estrategia de migración
  (Keycloak no las lee sin una extensión), a decidir en ese momento.
