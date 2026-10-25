# Seguridad y cumplimiento: objetivo y decisiones

> Las reglas de seguridad vigentes del código están en `../ARQUITECTURA.md` §6. Aquí: objetivo de cumplimiento y
> decisiones de evolución (§14, §14.1). El estado de implementación en la plataforma está en
> `amachi-platform/docs/PLATAFORMA.md` (§14.2).

## 14. Seguridad y cumplimiento — objetivo y decisiones (2026-10-24)

**Objetivo.** La plataforma cumple y deja **evidencia** de los controles técnicos de **ISO/IEC 27001:2022
(Anexo A)**, **OWASP ASVS 5.0 nivel 2**, **OWASP API Security Top 10** y de privacidad **Ley 25 (Québec) / GDPR /
ISO 27701**. (La certificación ISO es de la organización y su SGSI, no del software: la plataforma aporta los controles
y la evidencia para la auditoría.)

### 14.1 Decisiones

| Tema | Decisión | Control |
|---|---|---|
| Identidad | **Keycloak** (OIDC) como proveedor de identidad: login, MFA (TOTP y passkeys/WebAuthn, obligatorio para administradores), bloqueo progresivo, política de contraseñas, rotación de claves de firma, SSO, una organización Keycloak por tenant. La plataforma es **resource server** (valida tokens con JWKS) y conserva su autorización (`@PreAuthorize`, roles por tenant) | A.5.15–5.18, A.8.5 · ASVS V2/V3 |
| Sesión en el navegador | Patrón **BFF**: cookie `HttpOnly; Secure; SameSite=Strict`, ningún token en `localStorage` | ASVS V3 |
| Roles de la plataforma | `SUPER_ADMIN`, `ADMIN` (de la organización), `USER` (miembro). Cada sector agrega los suyos desde su módulo | A.5.15 |
| Cifrado de datos sensibles | **AES-256-GCM** con versión de clave (rotación sin perder datos) + **índice ciego HMAC** para búsquedas exactas; claves fuera del repositorio (variables de entorno / gestor de secretos) | A.8.24 |
| Multi-tenancy | `@TenantId` nativo de Hibernate (aplicado en toda lectura y escritura) + test de aislamiento en ambos sentidos | A.8.3 |
| Bitácora | Solo agregar, **encadenada por hash** (detecta manipulación), retención configurable, también lecturas de datos sensibles | A.8.15, A.8.16 |
| Privacidad | Módulo **`core-privacy`**: consentimientos, solicitudes del titular (acceso, rectificación, supresión), retención y borrado | A.5.34 · Ley 25 · ISO 27701 |
| Exposición | Actuator solo para el rol de operación o puerto interno; cabeceras CSP, HSTS, `X-Content-Type-Options`, `Referrer-Policy`; límite de tasa distribuido | A.8.9, A.8.20 |
| API | Errores RFC 9457; **claves de idempotencia** en POST; `ETag`/`If-Match` con `@Version`; validación de entrada y salida | ASVS V4/V5 · API Top 10 |
| Cadena de suministro | CI con SBOM (CycloneDX), OWASP Dependency-Check, CodeQL, gitleaks, OWASP ZAP; ninguna versión se publica con un hallazgo alto | A.8.25–8.29 |
