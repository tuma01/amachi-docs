# ADR-0007: Los actos sensibles exigen una reautenticación reciente (`auth_time`, RFC 9470)

- **Estado**: Aceptada
- **Fecha**: 2026-11-05
- **Decide**: Juan Amachi (opción (a) de la decisión D2 de la adopción de Vitalia)
- **Completa**: ADR-0006 (Keycloak, BFF; la contraseña solo existe en Keycloak)

## Contexto

La firma electrónica de un documento clínico (receta, nota de evolución, epicrisis, alta) pedía al médico su contraseña
y la comprobaba contra el hash del usuario. Con la ADR-0006 la plataforma no tiene contraseñas: la API es un *resource
server* que solo ve un token. Hace falta otra prueba de que **quien firma es quien tiene la sesión ahora**, no alguien que
heredó una sesión abierta, un equipo desatendido o un token robado. No es solo de la salud: cualquier sector tiene actos
así (aprobar un pago, cambiar un límite de riesgo en InverPulse).

## Opciones consideradas

- **(a) Reautenticación reciente** (step-up, RFC 9470): la API exige que el token diga que la persona escribió sus
  credenciales en Keycloak hace segundos (claim `auth_time`); si no, responde `401` con
  `WWW-Authenticate: Bearer error="insufficient_user_authentication", max_age="…"` y el cliente la manda a iniciar sesión
  de nuevo (`prompt=login`) y repite la llamada.
- **(b) Segundo factor propio de la firma** (PIN de firma, TOTP aparte): la plataforma guardaría un secreto más y volvería
  a tener un flujo de credenciales propio, que la ADR-0006 quitó.

## Decisión

**(a).** La prueba es el `auth_time` del token, que Keycloak solo renueva cuando la persona se autentica de nuevo.

- **API (`core-auth`)**: `AuthContext.authenticatedAt` (de `auth_time`) y el componente `RecentAuthentication`
  (`require()` / `require(Duration)`). Falla cerrado: sin `auth_time` (p. ej. un token de concesión directa) o más viejo que
  `app.security.reauthentication.max-age` (2 minutos por defecto) lanza `ReauthenticationRequiredException`
  (`SEC_017`) y el manejador global responde el `401` de arriba. El límite lo mide el `Clock` de la aplicación.
- **BFF**: `/oauth2/authorization/keycloak?organization=…&reauthenticate=true` añade `prompt=login`; Keycloak muestra su
  pantalla de credenciales aunque su sesión siga viva, el token nuevo trae un `auth_time` nuevo y la sesión del BFF lo
  reemplaza. El BFF deja pasar `WWW-Authenticate` a la SPA para que distinga este `401` de un "sesión terminada".
- **El sector** solo llama a `RecentAuthentication.require()` al principio de cada acto sensible. Ninguna contraseña viaja
  a la API: los contratos de firma pierden el campo `password`.

## Consecuencias

- La SPA (etapa 5) trata el `401` con `insufficient_user_authentication` como "pide las credenciales de nuevo" (navega a
  `reauthenticate=true`, conserva el borrador, vuelve y repite la firma), distinto del `401` común que reinicia la sesión.
- Probado con Keycloak real: un token de navegador trae `auth_time`; el de concesión directa no (por eso falla cerrado).
  `auth_time` no cambia al refrescar el token: renovar sin credenciales no sirve de prueba.
- El límite corto exige que la SPA pida la reautenticación **antes** de firmar si la última fue hace más de ese tiempo, o al
  recibir el `401`; no es un fallo del servidor.
- No sustituye al MFA del inicio de sesión (ADR-0006): la reautenticación pasa por el mismo flujo de Keycloak, MFA incluido
  para quien lo tenga.
- Un actor con el token en la mano durante `max-age` aún puede firmar; es el mismo riesgo que acepta la reautenticación
  por contraseña de cualquier sistema clínico. Recortar `max-age` lo reduce.
