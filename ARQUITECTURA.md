# Arquitectura del ecosistema Amachi (todas las aplicaciones, cualquier sector)

> **Qué es este documento.** La arquitectura común y recomendada para toda aplicación construida sobre
> **Amachi Platform**: una plataforma SaaS **multi-tenant** genérica (salud, finanzas/trading, educación, comercio…).
> Describe *cómo se construye* cualquier funcionalidad, con el nivel de una aplicación profesional en producción.
>
> **De dónde sale.** Se destiló de la arquitectura de **Vitalia** (aplicación hospitalaria construida sobre la misma
> plataforma), quitando todo lo propio del sector salud y conservando solo lo que se verificó en código real: reglas
> que nacieron de bugs encontrados y corregidos, no de teoría.
>
> **Cómo se combina con el documento de cada aplicación.** Cada aplicación de sector (ej. `ARQUITECTURA.md` de
> InverPulse) describe **solo su dominio** (sus entidades, flujos, reglas regulatorias). Todo lo demás —capas,
> multi-tenancy, seguridad, contrato, frontend, calidad— se rige por este documento. Si una aplicación necesita
> apartarse de una regla de aquí, lo dice explícitamente en su documento y por qué.
>
> **Creado**: 2026-10-24 (como `ARQUITECTURA-BASE.md`). **Movido a `amachi-docs`**: 2026-10-25; los estándares
> detallados, la seguridad, las decisiones (ADR) y lo propio de la plataforma están en archivos separados (índice en
> `README.md`). Cada sección conserva su número: una referencia «§N» se busca en el índice.

---

## 0. Cómo se usa y cómo se mantiene

1. **Antes de implementar** (pantalla, endpoint, entidad, refactor): leer la sección correspondiente y abrir el
   **archivo modelo** del caso (§12). Se replica el modelo; no se inventa un patrón nuevo.
2. **Si no hay modelo para el caso**: se propone uno, se valida en **un** ejemplo (con todas las pruebas de §11) y se
   agrega aquí **antes** de aplicarlo al resto.
3. **Mantenimiento obligatorio**: todo cambio que modifique una regla, un modelo o una convención se escribe en este
   repositorio (`amachi-docs`) **en la misma sesión de trabajo**, con su fila en `CHANGELOG.md` (§15); una decisión
   de arquitectura se registra como ADR (`adr/`). El roadmap registra *qué se hizo*; esta documentación, *cómo se
   hace*.
4. **Si el código y el documento no coinciden**: se verifica cuál está bien, se corrige el que esté mal y se anota en
   el registro de cambios (§15).
5. **Una sola fuente por nivel y por tema**: este repositorio (ecosistema), `amachi-platform/docs/PLATAFORMA.md`
   (plataforma) y el `docs/` de cada aplicación (dominio). Cada regla vive en un solo archivo; el `README.md` es el
   índice. Nunca dos documentos que traten el mismo tema (pasó en Vitalia: 21 documentos de distintas épocas,
   contradictorios entre sí y con el código; se consolidaron).

---

## 1. Principios

| Principio | En la práctica |
|---|---|
| **Nivel profesional siempre** | Nunca una versión "de prototipo". Se toma como referencia la aplicación líder del sector (Epic/Cerner en salud; Doctoralia/Zocdoc para reservas públicas; Bloomberg/TradingView/Interactive Brokers en finanzas) y se dice cuál se sigue y por qué. Se muestra el ejemplo concreto (URL, pantalla) **antes** de construir. |
| **Solución estructurada** | Ante la duda entre texto libre / booleano / enum fijo y una relación con catálogo, entidad propia o componente reutilizable: **la estructurada**. |
| **Escala real** | Nada que crezca con el uso se carga completo en memoria ni en el navegador (§9). |
| **Seguridad que falla cerrada** | Un endpoint sin regla explícita **no compila o no pasa el test de arquitectura** (§6). |
| **Una sola fuente de verdad** | Cada regla vive en un lugar (roles en el `@PreAuthorize`, rutas públicas en `PublicEndpoints`, transiciones de estado en su utility). El frontend tiene un *espejo* explícito, nunca una segunda regla inventada. |
| **Reutilización** | Todo lo nuevo se crea como pieza reutilizable (componente compartido, utility, estilo global). Nunca una presentación única por pantalla. |
| **Paridad entre roles análogos** | Lo que se hace para un rol aplica a los roles equivalentes salvo indicación contraria. |
| **Nunca pedir códigos técnicos al usuario** | Colores, íconos, organizaciones, lugares: se eligen con selectores visuales o búsqueda, nunca escribiendo un hexadecimal, un nombre de ícono o un código. |

---

## 2. Stack de referencia

| | Frontend | Backend |
|---|---|---|
| Lenguaje / framework | Angular **21** (standalone components, signals), TypeScript 5.9 | Java **25**, Spring Boot **4.1** / Spring Framework 7 (vía `com.amachi:amachi-parent` 2.1.x, ver §3.4) |
| UI | Angular Material, MtxGrid (`@ng-matero/extensions`), ApexCharts, FullCalendar si hay agenda | — |
| Estado | Signals + servicios (**sin** NgRx) | — |
| Persistencia | — | JPA/Hibernate 7, MySQL, Flyway, Hibernate Envers (auditoría) |
| Contrato | Cliente generado desde OpenAPI (`ng-openapi-gen`) | springdoc-openapi |
| Mapeo | — | MapStruct, Lombok |
| Pruebas | Vitest (`ng test`), Playwright (pantalla) | JUnit 5 + Mockito, Cucumber + Testcontainers (`platform-test`) |
| i18n | Un JSON por idioma (`src/assets/i18n/`) | `messages*.properties` y `validation*.properties` (en + idiomas de la app) |

- API bajo el context-path **`/api/v1`**. Desarrollo: backend 8088, frontend 4200.
- **Spring Boot 4 — configuración manual que no se borra** (`platform-boot`): `FlywayMigrationConfig` (Boot 4 ya no
  autoconfigura Flyway) y `JacksonLegacyConfig` (Boot 4 solo crea el `JsonMapper` de Jackson 3; el código usa el
  `ObjectMapper` de Jackson 2). En `platform-test`, `TestRestTemplate` requiere `spring-boot-resttestclient` /
  `spring-boot-restclient`.

---

## 3. Estructura de la plataforma y de una aplicación de sector

### 3.1 Módulos Maven

| Módulo | Responsabilidad | Prefijo de tablas | Flyway |
|---|---|---|---|
| `core-common` | Clases base (`BaseEntity`, `BaseService`, `BaseSpecification`, `BaseController`), `PageResponseDto`, excepciones, utilities (`TextSearch`), `AppConstants`, `GlobalCatalogAccess` | — | — |
| `core-geography` | Países y división política (catálogo global con niveles por país) | `GEO_` | `V2_*` |
| `core-domain` | `Person`, `Tenant`, `PersonTenant`, `AuditableTenantEntity`, auditoría Envers | `DMN_` | `V3_*` |
| `core-auth` | Tokens de Keycloak (resource server), cuentas en Keycloak (`IdentityAccounts`), `SecurityConfig`, `MultiTenantFilter`, roles, invitaciones, `AuthContextHolder`, notificaciones in-app, bitácora de auditoría | `AUT_` | `V4_*` |
| `core-management` | Gobierno de la plataforma: SuperAdmin, administradores de cada organización, configuración por organización, servicios contratados, facturación, temas, panel de plataforma | `MGT_` | `V8_*` |
| `core-privacy` | Derechos de la persona sobre sus datos (Ley 25 · GDPR · ISO 27701): acceso y portabilidad (reúne lo que cada módulo declara con `PersonalDataContributor`), rectificación (contacto al instante, identidad por solicitud que responde un administrador LEVEL_1 en 30 días) supresión (cada módulo borra lo suyo con `PersonalDataEraser`; la identidad compartida solo con la última organización de la persona) consentimientos (finalidades con versión; los módulos preguntan con `ConsentQuery`), responsable de privacidad, incidentes de confidencialidad (plazo de 72 h para avisar a la autoridad; se conservan siempre) y retención nocturna de lo ya respondido o retirado | `PRV_` | `V6_*` |
| `platform-boot` | Aplicación ejecutable: `application*.yml`, `GlobalExceptionHandler`, mensajes i18n, configuración (Flyway, Jackson, reloj), seeders de desarrollo genéricos | — | — |
| `platform-test` | Cucumber + Testcontainers de la plataforma y test de arquitectura de seguridad | — | — |
| `<app>-<dominio>` | Módulos del sector (ej. `inverpulse-market`, `inverpulse-signals`; en Vitalia: `vitalia-medical-core`, `vitalia-medical-catalog`) | prefijo propio de 3 letras (`MKT_`, `SIG_`…) | `V10_*` en adelante, un rango por módulo |

- Las migraciones viven en `src/main/resources/db/migration` **de cada módulo**. Cada módulo tiene su rango de versión
  para que nunca choquen.
- Un módulo de dominio depende de `core-*`; **nunca** al revés. Si `core-*` necesita algo del dominio, se usa un
  **Bridge** (interfaz en core, implementación en el dominio; modelo `TenantBridge`).
- Paquetes: `com.amachi.app.<módulo>.<feature>.<capa>` (ej. `com.amachi.app.inverpulse.signal.service.impl`).
- Clases, campos y métodos en **inglés y singular**. Comentarios y logs **en inglés**, cortos, explicando el *porqué*.

### 3.2 Estructura de un feature (backend)

```
<feature>/
  controller/   XApi (contrato OpenAPI) + XController (solo delega)
  dto/          XDto  · dto/search/XSearchDto
  entity/       X
  mapper/       XMapper (MapStruct)
  repository/   XRepository (+ projections)
  service/      XService · service/impl/XServiceImpl
  specification/XSpecification
  util/         reglas puras (XTransitions, XRules…) — utility classes
```

### 3.4 BOM y POM padre del ecosistema (repositorio `amachi-bom`)

- Mismo diseño que Spring Boot: `com.amachi:amachi-dependencies` (BOM puro, solo versiones, importable) +
  `com.amachi:amachi-parent` (padre: hereda el BOM y agrega Java 25, UTF-8, compilador, builds reproducibles y
  plugins configurados) + `amachi-build` (agregador, no se instala).
- Toda app hereda `amachi-parent`; una app que deba conservar otro padre importa `amachi-dependencies`.
- Una línea del BOM por línea menor de Spring Boot (`2.1.x` = Boot 4.1.x); una versión publicada no se edita.
  `com.amachi.core:amachi-bom:2.0.0-SNAPSHOT` (Boot 4.0.0) queda congelada para la Vitalia actual.
- Nunca fijar una librería que Spring Boot ya gestiona; las dependencias se declaran sin versión y solo en el módulo
  que las usa (el padre nunca fuerza dependencias en todos los módulos).
- La plataforma se consume como **dependencia versionada** (no se copia): publicará su propio BOM
  (`amachi-platform-dependencies`) con groupId `com.amachi.platform`; una app de sector nunca modifica `core-*`.

---

## 4. Multi-tenancy (Platform-as-a-Tenant)

- Un **tenant** es una organización cliente (hospital, firma de inversión, escuela…). La plataforma misma es el tenant
  **`GLOBAL`**, donde trabaja el SuperAdmin.
- **Identificadores**: `Tenant.id` (Long, PK interna, no se expone) · `Tenant.code` (identificador de negocio, en
  minúsculas, apto para subdominio y **único en toda la plataforma**).
- **Columna `TENANT_ID`**: guarda el `id` Long (no el código); `TENANT_CODE` se guarda al lado.
- **Resolución** (`MultiTenantFilter`): subdominio del host en producción (`{code}.dominio`); encabezado
  `X-Tenant-Code` en desarrollo y en rutas públicas donde el visitante elige la organización.
- **Contexto**: `TenantContext.getTenantId()` / `getTenantCode()` (ThreadLocal), fijado por los filtros de la
  petición (`MultiTenantFilter`, `JwtAuthenticationFilter`).
- **Aislamiento (SEG6, ISO 27001 A.8.3)**: `AuditableTenantEntity.tenantId` lleva **`@TenantId`** de Hibernate.
  `TenantIdentifierResolver` da a cada sesión su organización y Hibernate filtra **toda** consulta (derivada, `@Query`,
  Specification) y toda carga por id: el registro de otra organización **no existe** (404), sin código en el servicio.
  Escribir un registro de otra organización falla. Sin organización en el contexto la sesión no ve nada (nunca todo).
  No se agregan predicados de tenant ni métodos `...AndTenantId` que solo repitan la organización de la sesión.
- **SuperAdmin = alcance de plataforma**: su petición no se filtra (tenant *root*) y ve todas las organizaciones;
  **no** hay ramas `isSuperAdmin() ? findAll : …` en servicios ni Specifications. `isSuperAdmin()` queda solo para
  reglas de negocio (quién puede pedir otra organización, roles que no se muestran).
- **Cambiar de organización a mitad de una ejecución**: **solo** con `TenantScope` (sesión y transacción propias,
  restaura el contexto): `inTenant(id, code, …)` (enlace público con token, alta en otra organización),
  `asPlatform(…)` (jobs nocturnos, buscar un token antes de saber su organización), `inTenantContext(…)` (hilo
  `@Async` con una transacción por paso). Nunca `TenantContext.set…` a mano: la sesión ya abierta no se entera. Lo
  vigila `TenantIsolationArchitectureTest`.
- **Organización de un registro**: la toma de su relación `tenant` si la tiene (`owningTenant()`, columna `TENANT_ID`
  única y de solo lectura en la relación), si no del contexto. Una entidad nueva con `TENANT_ID` hereda de
  `AuditableTenantEntity` (lo exige el test de arquitectura).
- **SQL nativo y `JdbcTemplate`** no pasan por `@TenantId`: filtran por organización explícitamente o son de
  plataforma a propósito (numeración de facturas, bitácora, límite de tasa).
- **El `tenantCode` nunca viaja en el body** de un request ni en un DTO.
- **Datos compartidos entre tenants** (catálogos, datos de mercado, geografía): entidades **globales** sin
  `TENANT_ID`, leídas por todos, escritas solo por el SuperAdmin o por procesos de la plataforma.

---

## 6. Seguridad

> Objetivo de cumplimiento y decisiones de evolución (Keycloak, BFF, AES-GCM, `@TenantId`, bitácora encadenada,
> privacidad): §14. Esta sección describe las reglas vigentes.

- **Una sola fuente para los roles: el `@PreAuthorize` de cada endpoint.**
  - `SecurityConfig`: **solo** autenticación — rutas públicas, documentación de la API y
    `anyRequest().authenticated()`. Nunca una regla por rol de un controlador (dos capas con reglas distintas = la más
    estricta gana en silencio y cada cambio se hace dos veces; en Vitalia eso impidió crear usuarios a quien sí debía).
  - Un endpoint para cualquier usuario con sesión lo dice explícito (`CAN_SELF_SERVICE = "isAuthenticated()"`).
- **Test de arquitectura** (`platform-test/.../architecture/ControllerSecurityArchitectureTest`): recorre todos los
  `@RestController` por reflexión y **falla si un endpoint no público no tiene `@PreAuthorize`**; escribe el inventario
  completo (método, ruta, regla) para auditorías.
- **403 antes que 400**: `EarlyRoleAuthorizationInterceptor` evalúa el mismo `@PreAuthorize` antes de leer el cuerpo
  (si no, quien no tiene permiso recibe 400 y aprende las reglas de validación).
- **Identidad = Keycloak (ADR-0006)**: la plataforma **no tiene login ni contraseñas**. La API acepta solo tokens del
  realm `amachi` (firma por JWKS, emisor, audiencia `amachi-api`, vigencia en tiempo real); la organización sale del
  claim `organization` y debe ser la de la petición; los permisos se leen de la plataforma (`EffectiveAccess`,
  `UserTenantRole`), nunca del token. Toda alta (invitación, administrador creado, bootstrap) pasa por
  `IdentityAccounts`: vincula `AUT_USER.KEYCLOAK_ID`, agrega la membresía y Keycloak le envía a la persona el email
  para crear su contraseña. Reset de un administrador = email de Keycloak. Los inicios de sesión se auditan en los
  eventos de Keycloak. Lo vigila `IdentityArchitectureTest` (ninguna clase usa `PasswordEncoder` ni emite JWT).
- **Segundo factor, bloqueo y política (SEG3, realm como código `keycloak/amachi-realm-*.json`)**: los roles de
  `app.security.oidc.second-factor-roles` (por defecto `SUPER_ADMIN`, `ADMIN`; un sector agrega los suyos) reciben
  el rol de realm `mfa-required` al darse de alta y el flujo `amachi-browser` les exige OTP en cada inicio de
  sesión (lo configuran en el primero). 5 fallos bloquean la cuenta 1 minuto más cada vez, hasta 15. Contraseña:
  12 a 128 caracteres, ni el usuario ni el email, ninguna de las 5 anteriores, sin reglas de composición
  (NIST 800-63B). Ningún cliente de producción tiene *direct grant* (saltaría el flujo del navegador).
- **BFF (SEG4, `platform-bff`, paquete `com.amachi.bff`, sin dependencias de `core-*`)**: la única puerta del
  navegador. Inicia la sesión con Keycloak (código + PKCE, cliente confidencial `amachi-bff`) pidiendo
  `organization:<alias>` (`/oauth2/authorization/keycloak?organization=<alias>`); los tokens quedan en la sesión del
  servidor (Spring Session JDBC, tabla `BFF_SESSION`, cada atributo sellado con AES-256-GCM, `BFF_SEAL_KEY`) y el
  navegador solo recibe `AMACHI_SESSION` (`HttpOnly; Secure; SameSite=Strict`). La petición de autorización en curso
  va en una cookie sellada `Lax` de 5 minutos, solo hacia el callback (con `Strict` el regreso desde Keycloak no la
  traería). `/api/**` se reenvía a la API con el token (renovado con el refresh token) y `X-Tenant-Code` de la
  organización de la sesión; solo pasan las cabeceras del contrato (`If-Match`, `Idempotency-Key`, `ETag`,
  `Location`, `Retry-After`…), nunca cookies. Sin sesión: `401` (no redirige). CSRF para SPA (`XSRF-TOKEN` →
  `X-XSRF-TOKEN`). `POST /bff/logout` termina la sesión y responde `202` con la URL de cierre de Keycloak en
  `Location`. `GET /bff/user`: quién es y en qué organización entró. Sin CORS: el SPA y el BFF comparten origen.
- **Rutas públicas: una sola fuente** `PublicEndpoints` (leída por `SecurityConfig`, el filtro JWT y
  `MultiTenantFilter`). Dos conceptos distintos: *sin sesión* y *sin organización* (una búsqueda pública entre
  organizaciones no tiene ni una ni otra; el portal público de una organización no tiene sesión pero sí organización).
- Chequeo de rol dentro de la lógica: **solo** `AuthContextHolder.hasRole / hasAnyRole / isSuperAdmin /
  isTenantAdmin`, con nombres de `AppConstants.Roles`; nunca leer authorities a mano ni comparar un string suelto.
- **Recursos propios** (el cliente lee lo suyo): rutas `/…/me/**` que resuelven a la persona desde la sesión, o guard
  de pertenencia en el `@PreAuthorize` (`@xOwnershipGuard.isOwn(#id)`). **Un rol de cliente final nunca entra a un
  endpoint por id sin guard** (en Vitalia, un paciente leía la historia de otro cambiando el id).
- **Datos para todos = lista blanca**: lo que ve cualquier rol (directorio, perfil público) sale de un DTO propio
  con solo los campos públicos (`unmappedTargetPolicy = ERROR` en su mapper), nunca del DTO completo.
- Datos personales globales (direcciones, personas): nunca "autenticado" a secas; solo el rol que los administra.
- Firma electrónica de un documento (cuando la aplicación la requiera): reautenticación con contraseña, inmutable
  después de firmar (el mapper ignora los campos de firma), auditada. Patrón reutilizable `Signer`.
- Endpoints con efectos sobre terceros (enviar correos, publicar) con límites de frecuencia (`RateLimitFilter`).
- Sesión del frontend: escuchar el evento `storage` para detectar otro usuario o rol en otra pestaña.

---

## 7. Tiempo, estados y procesos

- **Reloj**: toda regla que dependa de "ahora" o de "hoy" usa un reloj inyectable de la organización
  (`TenantClock`: `now()`, `today()`, `zone()`), nunca `Instant.now()` / `LocalDate.now()`. El instante sale del bean
  `Clock` (`ClockConfig`, reemplazable en pruebas) y el **día en la zona de la organización** (`TenantConfig.timezone`
  o la de la plataforma) — nunca UTC por omisión. (En Vitalia: `HospitalClock`; en una app de mercados, además, la
  zona y el calendario de cada **bolsa**.)
- **Instante vs. fecha sin hora**: un instante viaja en UTC (`OffsetDateTime`, ISO con `Z`) y cada pantalla lo muestra
  en la zona del usuario; una fecha sin hora es `LocalDate` y viaja como `"yyyy-MM-dd"` — **nunca**
  `toISOString()` de una fecha local (al este de UTC guardaría el día anterior).
- **Ciclo de vida con estados**: las transiciones permitidas y sus reglas de tiempo viven **solo** en una utility
  `XTransitions` (+ su espejo en el frontend para decidir qué botones mostrar). El estado **nunca se edita** con el
  PUT: cambia con operaciones explícitas (`PATCH /{id}/status`, `POST /{id}/cancel` con motivo…). Lo cerrado no se
  reabre; ningún paso se salta; cada paso estampa su hora.
- **Tareas programadas por organización**: cada organización en su zona; el proceso pasa por el mismo servicio que una
  persona (mismas reglas) y se audita con el auditor `SYSTEM`; los disparos son configurables y en las pruebas se
  ejecutan explícitamente.
- **Avisos**: módulo de notificaciones in-app (campana) que guarda **la clave i18n y sus datos** (nunca el texto),
  una por origen y tipo (idempotente) y se resuelve sola cuando su origen ya no necesita a nadie.
- **Correos**: se envían **después del commit**; se registra el resultado real (enviado / falló + motivo) y lo que
  falló queda visible para que una persona actúe. En desarrollo, siempre a MailDev, nunca al servidor real.
- **Bitácora de auditoría**: todo cambio de gobierno se registra con `AuditService.registerEntityEvent(…)` después del
  commit, en su propia transacción; mensajes traducibles (`AuditMessage.of(clave, params)`); la organización ve lo que
  hizo su gente **y lo que la plataforma cambió en ella**. **Solo agregar y encadenada** (SEG11): cada evento entra por
  `AuditChain.append` con la huella HMAC del anterior de su organización; nunca se actualiza ni se borra (solo la
  retención, que deja ancla y constancia); `GET /audit-events/integrity` lo verifica. Una lectura de datos sensibles
  lleva `@AuditedRead("Tipo")` en el método del controlador (evento `SENSITIVE_DATA_READ`).
- **Datos personales de un módulo (Ley 25, `core-privacy`)**: todo módulo que guarda datos de una persona (de la
  plataforma o de un sector) implementa `PersonalDataContributor` (`core-common`): devuelve sus secciones como records
  propios, en la sesión de la petición (`@TenantId` ya limita a la organización), nunca secretos ni identificadores
  internos (id de Keycloak, huellas de token). Así la plataforma responde una solicitud completa (acceso, portabilidad)
  sin conocer el dominio del sector; cada respuesta queda en la bitácora (`PERSONAL_DATA_EXPORTED`).
  El mismo módulo implementa también `PersonalDataEraser`: borra o anonimiza sus datos de la persona en la
  organización (`Scope.ORGANIZATION`) y, cuando la plataforma se lo pide porque era su última organización, lo que
  dependa de la identidad compartida (`Scope.IDENTITY`). Idempotente; conserva solo lo que una ley obliga a guardar.
  Un tratamiento basado en el consentimiento (promociones, un estudio, compartir con un socio) pregunta a
  `ConsentQuery.hasConsent(usuario, código)` antes de cada uso, nunca una vez para siempre.

---

## 9. Escalabilidad y rendimiento (resumen de reglas)

1. Más de 50 registros posibles → paginación en el servidor (`Pageable` + `MatPaginator`).
2. Búsqueda de texto → consulta a la base con debounce de 400 ms; nunca filtrar en el navegador.
3. Cero N+1 (projections, `JOIN FETCH`, `@EntityGraph`, `@BatchSize`, `IN (:ids)`).
4. Paneles con conteos del servidor y un número fijo de consultas.
5. Catálogos globales con caché (Spring Cache + Caffeine en el servicio; `CatalogCacheService` en el frontend) —
   nunca la caché en las pantallas de edición del propio catálogo.
6. Datos compartidos entre tenants se calculan **una vez** y se sirven a todos (no por organización).
