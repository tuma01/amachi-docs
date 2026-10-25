# ADR-0003: Keycloak como proveedor de identidad

- **Estado**: Aceptada
- **Fecha**: 2026-10-24
- **Decide**: Juan Amachi

## Contexto

- La autenticación actual es propia: JWT firmado con HS256 (jjwt), tokens de refresh e invitación guardados en
  claro, sin MFA, sin bloqueo de cuenta, sesión del frontend en `localStorage`.
- El objetivo de cumplimiento (ISO 27001 A.5.15–5.18 y A.8.5, OWASP ASVS L2 V2/V3) exige MFA para administradores,
  bloqueo progresivo, política de contraseñas, rotación de claves de firma y SSO.
- Implementar y mantener todo eso dentro de la plataforma es costoso y riesgoso.

## Opciones consideradas

1. **Endurecer la autenticación propia**: control total; pero hay que construir y auditar MFA, passkeys, rotación de
   claves y bloqueo.
2. **Keycloak (OIDC)**: proveedor de identidad maduro y de código abierto; la plataforma pasa a ser *resource
   server*.
3. **Servicio gestionado (Auth0, Cognito…)**: sin operación propia; costo por usuario y dependencia de un tercero
   para datos de identidad.

## Decisión

Opción 2. Keycloak gestiona login, MFA (TOTP y passkeys/WebAuthn, obligatorio para administradores), bloqueo,
política de contraseñas, rotación de claves y SSO, con una organización Keycloak por tenant. La plataforma valida
tokens con JWKS y **conserva su autorización** (`@PreAuthorize`, roles por tenant). En el navegador, patrón BFF con
cookie `HttpOnly; Secure; SameSite=Strict`; ningún token en `localStorage`.

## Consecuencias

- Se elimina el código propio de emisión de JWT y de contraseñas en la etapa de seguridad de la plataforma.
- Keycloak se agrega al `docker compose` de desarrollo y a las pruebas de integración.
- Lo implementado primero en una app se replica en la otra con el mismo diseño.
- Detalle en `seguridad/cumplimiento.md` §14.1.
