# Registro de cambios de la arquitectura

> Conserva su número de sección (§15). Una fila por cambio de regla, modelo o convención, en la misma sesión en que se
> decide. La entrada más reciente arriba.

## 15. Registro de cambios

| Fecha | Cambio |
|---|---|
| 2026-10-26 | Regla del último administrador (§5.6.1) cerrada en la plataforma: SuperAdmin (solo nivel 1 activos, 409), baja propia vía `AccountDeactivationGuard` (Bridge en `core-auth`) y claves traducibles. Etapa 3 de la plataforma iniciada: SEG0 (además, avatar aislado por organización) y SEG1 (tokens de invitación solo como huella SHA-256). |
| 2026-10-26 | Etapa 2 de la plataforma terminada: sin vocabulario de salud en `core-*` (organización demo `demo-org`, roles `SUPER_ADMIN`/`ADMIN`/`USER`, i18n en/es completo, plantillas de correo con `platformName`, rutas públicas solo con controlador); suite verde: 177 tests unitarios y 69 escenarios Cucumber (`amachi-platform/CLAUDE.md`). |
| 2026-10-26 | La demo de Vitalia vive solo en la etiqueta `v1.0.0-demo` (se borró la rama `release/1.0` en `vitalia-api` y `vitalia-web`); se abre con `git switch --detach v1.0.0-demo` y un arreglo urgente crea su rama desde la etiqueta. Complementa el ADR-0005 sin cambiar su decisión. |
| 2026-10-26 | Una sola carpeta por app: se quitan los worktrees `vitalia-api-demo` y `vitalia-web-demo`. La demo sigue en la rama `release/1.0` (etiqueta `v1.0.0-demo`) y se abre en la misma carpeta con `git switch release/1.0`; se vuelve con `git switch main`. La decisión del ADR-0005 no cambia. |
| 2026-10-26 | Etapa D de la plataforma: hereda `com.amachi:amachi-parent:2.1.0-SNAPSHOT` (Boot 4.1.2, JUnit 6, Testcontainers 2), groupId `com.amachi.platform` y BOM propio `amachi-platform-dependencies` (ADR-0001, ADR-0002); desaparece la colisión de coordenadas con Vitalia en `~/.m2`. |
| 2026-10-26 | ADR-0005 reemplaza al ADR-0004: Vitalia evoluciona en su mismo repositorio (`main`) y la demo queda congelada en la rama `release/1.0` con la etiqueta `v1.0.0-demo` (worktrees `vitalia-api-demo` y `vitalia-web-demo`); no se crea `vitalia-legacy`. |
| 2026-10-26 | Regla del último administrador (§5.6.1) con el estado real del código; convención de ramas `<tipo>/<CLAVE>-NNN` y clave `DOC` para este repositorio (§10). Carpetas reorganizadas: plataforma, BOM y docs en `F:\JAVA-PROJET\amachi`; Vitalia en `F:\JAVA-PROJET\vitalia` (`vitalia-api`, `vitalia-web` y los worktrees de la demo `release/1.0`); material viejo en `F:\JAVA-PROJET\back-up`. |
| 2026-10-25 | `ARQUITECTURA-BASE.md` movido al repositorio `amachi-docs` y separado por tema (arquitectura, estándares, seguridad, plantillas) sin cambiar la numeración; lo propio de la plataforma (§12, §13, §14.2) pasa a `amachi-platform/docs/PLATAFORMA.md`; ADR 0001–0004 registran las decisiones del BOM, la plataforma como dependencia, Keycloak y Vitalia congelada. |
| 2026-10-25 | BOM 2.1.0 al estilo Spring Boot (`amachi-dependencies` + `amachi-parent`), Spring Boot 4.1.2, JUnit 6.1 alineado con Cucumber 8; la plataforma se consume como dependencia versionada; la Vitalia actual queda congelada en el BOM 2.0.0 (§2, §3.4, §14.2). |
| 2026-10-24 | Repositorio de la plataforma movido a `F:\JAVA-PROJET\inverpulse\workspace\amachi-platform`; ubicaciones del ecosistema anotadas en §12 y en `CLAUDE.md`. |
| 2026-10-24 | Etapa 2 (parcial): `Organization` como tenant cliente genérico, roles como datos (`SUPER_ADMIN`/`ADMIN`/`USER`), perfiles por rol con `DomainProfileRequest` (sin ThreadLocal), enums extensibles por interfaz `code()`, servicios contratables como catálogo; nuevos modelos en §12. |
| 2026-10-24 | Objetivo de cumplimiento (ISO 27001, OWASP ASVS L2, Ley 25) y decisiones: Keycloak, BFF, AES-GCM + HMAC, `@TenantId`, bitácora encadenada, `core-privacy`; plan por etapas (§14). |
| 2026-10-24 | Base de datos `platform_db`, trigramas por sector y rangos Flyway (§3.3); modernización recomendada (§13). |
| 2026-10-24 | Creado a partir de `ARCHITECTURE.md` de Vitalia (versión del 2026-10-23), sin lo propio del sector salud. Módulos de arranque y pruebas renombrados a `platform-boot` / `platform-test`. |
