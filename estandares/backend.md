# Estándar de backend (Java · Spring Boot)

> Parte de la arquitectura del ecosistema (`../ARQUITECTURA.md`). Conserva su numeración (§5).

## 5. Backend por capas

### 5.1 Entidad
- Jerarquía: `BaseEntity` (id) → `Auditable<String>` (auditoría + `@Version` + `EXTERNAL_ID`) →
  `AuditableTenantEntity` (`TENANT_ID` + `TENANT_CODE` + `IS_DELETED` + filtro de tenant).
  - **De una organización**: `extends AuditableTenantEntity`.
  - **Global** (catálogos, `Person`, geografía, datos compartidos): `extends Auditable<String>`.
- Borrado lógico (`IS_DELETED`) con `@SQLRestriction("IS_DELETED = false")`; `BaseService.delete()` decide lógico o
  físico según la entidad. Una restricción `UNIQUE` que incluye `IS_DELETED` obliga a **reactivar** el registro borrado
  en vez de crear otro (si no, el duplicado ya no se puede borrar nunca).
- `@Audited` en entidades de negocio; FK a un catálogo global → `@Audited(targetAuditMode = NOT_AUDITED)`.
- Los roles de una persona (cliente, empleado, analista…) **componen** `Person` (`@ManyToOne`); nunca heredan de ella.
  Unicidad `(FK_ID_PERSON, TENANT_ID, IS_DELETED)`.
- **Datos sensibles cifrados (SEG5)**: `@Convert(converter = EncryptedStringConverter.class)` = AES-256-GCM con nonce
  aleatorio y versión de clave (`v<n>:…`, columna `VARCHAR(600)` para 100 caracteres). La columna cifrada **nunca** es
  `UNIQUE` ni se compara: a su lado va su **índice ciego** `@BlindIndex(source, purpose)` (`CHAR(64)`, HMAC-SHA256 con su
  propia clave, valor normalizado), que lleva el `UNIQUE` y las búsquedas exactas
  (`findByNationalIdIndex(blindIndexer.of(Person.NATIONAL_ID_INDEX, valor))`); la entidad declara
  `@EntityListeners(BlindIndexListener.class)` y el índice es `@NotAudited`. Nunca `LIKE` sobre un dato cifrado. Lo
  vigila `DataProtectionArchitectureTest`. Claves: `app.data-protection.*`, por variable de entorno, sin valor por
  defecto. **Ningún dato personal en claro**, tampoco en una invitación (documento) ni en la bitácora (email del actor, que
  se cifra y deja de ser buscable por texto: se filtra por `userId`; la cadena firma el valor en claro, así que sigue
  verificando). **Rotación**: se añade la clave nueva, `active-key` apunta a ella y `FieldReEncryptionService` (cada noche, por
  lotes, segura entre instancias) re-cifra lo que cifró la anterior en **todas** las columnas con `EncryptedStringConverter`
  —las encuentra en las entidades— y en su historial `_AUD`; la clave vieja se retira cuando ya no quedan valores con ella
  (lo registra el log). Un campo cifrado necesita `@Column(name)` explícito. El índice ciego no rota. Nunca un dato personal
  en los logs.
- Converters y entity listeners de JPA son beans de Spring (`SpringBeanContainer`): reciben sus dependencias por
  constructor, sin holders estáticos. Solo `AuditRevisionListener` (Envers) sigue leyendo un holder.
- `@Builder.Default` de Lombok **no aplica** con MapStruct: todo campo `NOT NULL` con valor por defecto se asigna en el
  servicio.
- Hibernate solo ejecuta el `@PrePersist` de la clase más específica: una subclase que declare el suyo conserva el
  comportamiento del padre (tenant, `externalId`).

### 5.2 DTO y SearchDto
- DTO: FK como `Long` (`accountId`), campos calculados de solo lectura (`accountName`), `@Schema` con descripción y
  ejemplo en cada campo, Bean Validation. **Nunca** `tenantId`, `version`, `externalId` ni auditoría.
- Preferir tipar con el **enum** en el DTO. Si un `String` se convierte a enum, lleva `@EnumValue` (sin él, un valor
  desconocido es un 500 dentro del mapper).
- SearchDto: `extends BaseSearchDto`, solo filtros de negocio, texto libre en `query`. **Nunca `@Hidden`** (el contrato
  debe publicar los filtros).
- El parámetro del paginado se llama **`searchDto`** en Api y controlador (con otro nombre, el cliente generado no lo
  envía y la búsqueda no viaja, sin error).

### 5.3 Mapper
- MapStruct `@Mapper(config = BaseMapperConfig.class)` que extiende `EntityDtoMapper<E, D>`, con las anotaciones de
  campos ignorados (`@IgnorePureAuditableFields`, `@IgnoreTenantAuditableFields`, `@IgnoreSoftDelete`…).
- **FK a otra entidad: ignorarla en el mapper y resolverla en el servicio** (`entityManager.getReference` o `find`).
  Nunca `@Mapping(target = "x.id", source = "xId")`: crea un objeto "cáscara" que falla al guardar (409/500) o, sobre
  una relación ya cargada, cambia su id ("identifier was altered"). Fue el bug más repetido de Vitalia.
- Si la respuesta lee datos de la relación (nombre, código) **después** de la transacción, la relación debe estar
  cargada: `em.find` (no `getReference`) o `@EntityGraph` en el repositorio. Si no: `LazyInitializationException` → 500.
- Prohibido el round-trip `Entity → DTO → Entity`.

### 5.4 Repository
- `extends TenantCommonRepository<E, Long>` (de una organización) o `CommonRepository<E, Long>` (global).
- Listados masivos, conteos y agregados → **Interface Projection** con alias JPQL. **Prohibido** `List<Object[]>` y
  `SELECT new paquete.Dto(...)`.
- **Cero N+1**: nunca una consulta dentro de un bucle; `JOIN FETCH`, `IN (:ids)`, projections.
- Una lista paginada que muestra datos de relaciones `@ManyToOne` sobrescribe `findAll(Specification, Pageable)` (y
  `findById` si la respuesta las lee) con `@EntityGraph(attributePaths = {…})`. Colecciones leídas por fila:
  `@BatchSize(size = 50)`. **Nunca** `@EntityGraph` sobre colecciones con paginado (Hibernate paginaría en memoria).

### 5.5 Specification
- `extends BaseSpecification<E>` (aplica tenant y borrado lógico).
- Texto libre (`query`): `TextSearch.everyWordInAnyField(cb, texto, campos…)` — cada palabra en algún campo, en
  cualquier orden, sin distinguir tildes ni mayúsculas. Para coincidir con una colección: subconsulta `EXISTS`
  correlacionada (sin JOIN, que duplicaría filas de la página).
- Los comodines que escribe el usuario (`%`, `_`) no son comodines.
- Orden explícito por naturaleza de la lista (catálogos por nombre/código, personas por apellido y nombre);
  `BaseService` ordena por `id DESC` si no se indica otro.

### 5.6 Service
- Interfaz `XService` + `XServiceImpl extends BaseService<E, D, F>`, con los hooks `getRepository()`,
  `buildSpecification()`, `mapToEntity()` y `mergeEntities()` (+ `publishCreated/UpdatedEvent` si la entidad no emite
  eventos de dominio).
- `@Transactional(readOnly = true)` en la clase y `@Transactional` en **cada** método de escritura, sin excepción (un
  método que "solo" inserta un registro auxiliar hereda el `readOnly` y falla con "Connection is read-only").
- `BaseService` está **congelado**: todo cambio se consulta y se justifica antes; se sobrescribe en el servicio
  concreto.
- Orquestación en el método público + métodos privados con nombre; Java moderno (records, streams, `switch` con
  patrones, `Optional` de retorno, text blocks); cero valores mágicos.
- **Lógica pura sin estado → utility class** (`final`, constructor privado, métodos `static`), nunca un `@Service`.
  Ejemplos: transiciones de estado, reglas de un formulario, ventanas de tiempo.
- Patrones establecidos (reusar antes de inventar): Bridge, Strategy/Provider, Template Method (`BaseService`),
  Builder, Observer (listener de auditoría).

### 5.6.1 Último administrador (Last Man Standing)

Ninguna operación puede dejar a la plataforma sin un superadministrador de nivel 1, ni a una organización sin un
administrador de nivel 1. Sin esa cuenta nadie puede crear otra y el acceso solo se recupera tocando la base de datos.

- **Qué operaciones protege:** borrar, desactivar y bajar de nivel. Las tres sacan a la cuenta del grupo de
  administradores de nivel 1, así que se validan igual.
- **Qué se cuenta:** solo cuentas **activas** (usuario habilitado) del **nivel 1** y del mismo ámbito (la plataforma o
  la organización), excluyendo la cuenta que se modifica.
- **Dónde se valida:** en el servicio, dentro de la misma transacción que la escritura. Nunca solo en el frontend.
- **Error:** `BusinessException` con clave i18n (409, §5.7), por ejemplo `admin.error.last_level1`, con mensaje en
  todos los idiomas. Nunca `IllegalStateException` (termina en 500) ni texto fijo en inglés.
- **Frontend:** la acción se muestra deshabilitada con el motivo cuando la cuenta es la última de nivel 1; el 409 del
  servidor sigue siendo la protección real.
- **Pruebas:** borrar, desactivar y bajar de nivel a la última cuenta devuelven 409 y no cambian nada; las mismas
  operaciones con otra cuenta de nivel 1 activa se permiten. Probar con datos `PRUEBA`, nunca con el seed.

**Estado del código (plataforma, cerrado el 2026-10-26 en la etapa 3, SEG0):**

| Caso | Implementación |
|---|---|
| Admin. de organización: borrar o bajar de nivel | `TenantAdminServiceImpl`, otros nivel 1 activos; clave `tenant_admin.error.last_level1` (409) |
| Superadmin: borrar, bajar de nivel o desactivar | `SuperAdminServiceImpl`, solo nivel 1 activos (`countByLevelAndUserEnabledTrueAndIdNot`); clave `super_admin.error.last_level1` (409) |
| Darse de baja uno mismo (`DELETE /account`) | `core-auth` pregunta a cada `AccountDeactivationGuard` (Bridge) antes de desactivar; `core-management` implementa `LastAdministratorGuard` |

Pruebas: unitarias de los tres casos; Cucumber del administrador de organización con una organización creada en el
escenario. El último SuperAdmin se prueba solo con unitarias: el único de la suite es el de seed.

### 5.7 Errores

**Formato de toda respuesta de error (RFC 9457, desde 2026-10-26):** `application/problem+json` con `type`
(`urn:amachi:problem:<código>`, ej. `urn:amachi:problem:BUS_006`), `title` (motivo HTTP), `status`, `detail` (el mensaje
en el idioma de quien lee), `instance` (la ruta) y las extensiones `timestamp` y `errors` (lista con `code`, `field`,
`details.invalidFields`, `details.suggestion`…). Los errores que responde Spring MVC (405, 415, ruta inexistente) usan el
mismo documento. **Nunca** se devuelve una traza, una clase de excepción ni el mensaje interno de un parser: van al log
con el `traceId` que recibe el cliente.

**Claves de idempotencia:** los POST que crean algo crítico (organizaciones, invitaciones, cargas masivas, facturas,
servicios contratados, administradores) aceptan `Idempotency-Key`. Misma clave y mismo pedido → la primera respuesta con
`Idempotent-Replayed: true`; misma clave y otro pedido → 422 (`BUS_009`); mientras el primero corre → 409 (`BUS_010`).
Una ruta nueva que crea algo que no debe duplicarse se agrega a `IdempotencyProperties.PATHS`. La reserva de una clave
solo dura mientras corre el pedido (`inProgressTimeout`, 10 min por defecto): si la JVM cae a medias, la clave vuelve a
estar libre en minutos y no 24 h después; la respuesta guardada sí vive la retención. El cuerpo se lee con tope
(`maxBodyBytes`, 1 MiB; más → 413) y las cargas multipart se resumen en flujo, sin cargarlas en memoria.

**Tamaño de página:** el que pide el cliente nunca se confía. `BaseService.getAll(search, pageIndex, pageSize)` lo acota
(`MAX_PAGE_SIZE` 200; sin valor, `DEFAULT_PAGE_SIZE` 20; índice negativo → 0; tamaño menor que 1 → 1). Una lista con
otro tope lo documenta en su propio servicio.

**Concurrencia (`ETag` / `If-Match`):** la respuesta con un registro versionado lleva `ETag: "<version>"` y el DTO
expone `version` de solo lectura (el mapper **nunca** la copia del cliente a la entidad). Quien edita o borra la reenvía
en `If-Match`: si el registro cambió → 412 (`BUS_011`) y no se aplica nada; una `version` vieja en el cuerpo → 409
(`BUS_008`). Sin precondición funciona como antes. Un DTO nuevo de entidad implementa `Versioned`; la comparación vive
en `BaseService` (`ConcurrencyCheck`), no en cada servicio.

| Situación | Excepción | HTTP |
|---|---|---|
| Regla de negocio sobre el **estado** (duplicado, transición inválida, conflicto) | `BusinessException("modulo.error.clave", details?, args…)` | **409** |
| Dato inválido detectado por una regla del servicio (usa el reloj, otro campo o la base) | `FieldValidationException(Map campo→clave)` | **400 por campo** (misma forma que `@Valid`, `details.invalidFields`) |
| Bean Validation | `@Valid` | 400 por campo |
| No encontrado / ajeno | `ResourceNotFoundException` | 404 (sin revelar si existe o es de otro) |
| Demasiados pedidos (límite de tasa, SEG9) | — (filtro) | 429 + `Retry-After` |
| El registro cambió desde que se leyó (`If-Match`) / `version` vieja | — (`BaseService`) | 412 / 409 |
| Sin permiso | `@PreAuthorize` | 403 (**antes** que 400, §6) |

- **Un error explica y propone**: los `args` llenan `{0}`, `{1}` del mensaje; una propuesta aplicable va en
  `details.suggestion` con los nombres de campo del DTO y `details.suggestionLabel`. El frontend ofrece "Usar …" y
  completa el formulario, **nunca guarda solo**.
- Toda clave nueva tiene mensaje en **todos** los `messages*.properties` (si no, el usuario ve la clave cruda).
- Fechas y horas en un mensaje: en la zona de la organización y como las escriben las pantallas.

### 5.8 Api + Controller

**Recurso de entidad: los 6 métodos estándar**

| Método | HTTP | Respuestas |
|---|---|---|
| `getXById(id)` | `GET /recurso/{id}` | 200 · 400 · 403 · 404 · 500 |
| `createX(dto)` | `POST /recurso` | **201 + `Location`** · 400 · 403 · 409 · 500 |
| `updateX(id, dto)` | `PUT /recurso/{id}` | 200 · 400 · 403 · 404 · 409 · 500 |
| `deleteX(id)` | `DELETE /recurso/{id}` | 204 · 400 · 403 · 404 · 409 · 500 |
| `getAllXs()` | `GET /recurso/all` | solo catálogos cerrados y chicos |
| `getPaginatedXs(searchDto, pageIndex, pageSize)` | `GET /recurso` | 200 · 400 · 403 · 500 |

- Más las operaciones propias del dominio (`checkIn`, `sign`, `cancel`…). Una entidad que **crece con el uso no expone
  `getAll`** (solo el paginado). Excepciones (sub-recursos, configuraciones únicas, registros que no se editan por
  trazabilidad) llevan un comentario de una línea en la Api.
- **Recurso sin entidad** (resumen, cálculo, panel): solo sus métodos reales.

**Api** (interfaz; todo el contrato OpenAPI vive aquí): Javadoc; `@Tag(name, description)` — el `name` define el
nombre del servicio generado en el frontend y **no se cambia** en un endpoint existente; `NAME_API` / `NAME_API_PLURAL`;
`@Operation` con todas sus respuestas; `@Parameter` en **cada** parámetro; rutas con las constantes `ID`/`ALL`; el
SearchDto como **objeto** (`@Parameter @NonNull XSearchDto searchDto`, nunca `@ParameterObject`); `@NonNull` de
`org.jspecify` en ids, bodies y SearchDto; `@Valid @RequestBody`; nunca `@NonNull` en `pageIndex`/`pageSize`.

**Controller** (solo delega): `@RestController` + `@RequestMapping("/recurso")` (sin `/api/v1`) +
`@RequiredArgsConstructor` + `@Slf4j` + `extends BaseController implements XApi`; inyecta la **interfaz** del
servicio y el mapper; `@PreAuthorize` en **cada** método con constantes `CAN_*` con nombre por su **significado**
(`CAN_READ`, `CAN_WRITE`, `CAN_SIGN`, `CAN_APPROVE`…; nunca una regla en línea); `create` →
`ResponseEntity.created(location).body(dto)`; paginado → `PageResponseDto.of(page, mapper::toDto)`; `log.info` corto
en las escrituras, sin datos personales. Parámetros con el **mismo nombre** que en la Api.

- **Datos maestros globales** (leídos por cualquier usuario con sesión, escritos solo por el SuperAdmin): usan la
  política compartida `GlobalCatalogAccess.CAN_READ/CAN_WRITE` con import estático explícito. **No** se declara en
  `BaseController`: heredada por todos, un controlador que olvidara su regla quedaría abierto sin aviso.
- El usuario de la sesión: `AuthContextHolder.getUserId()` (el principal es `AuthContext`: `userId`, `tenantId`,
  `tenantCode`). El "yo" (cliente, empleado de la sesión) se resuelve **siempre** desde la sesión, nunca del request.
- Test del controlador (modelo `CountryControllerTest`): página completa y, por reflexión, **toda operación de la Api
  con `@PreAuthorize`**.

### 5.9 Contrato OpenAPI → frontend
1. Cambio de DTO/endpoint → compilar y levantar.
2. Bajar `/api/v1/v3/api-docs` al `openapi.json` del frontend (formateado igual que el commiteado).
3. Comparar: **solo** debe cambiar lo esperado (paths y esquemas).
4. `npm run gen-api`. **Nunca** editar a mano el código generado.
- Swagger agrupado por áreas (un controlador nuevo agrega su prefijo de ruta al área que corresponde) + un grupo
  "Completo" que usa el generador.

### 5.10 Hilos, tareas asíncronas y build
- **Hilos virtuales** (`spring.threads.virtual.enabled: true`): cada petición y cada `@Scheduled` corre en uno. Nada
  depende de un número de hilos; el recurso escaso es el pool de conexiones (`hikari.maximum-pool-size`): más peticiones
  que conexiones esperan, no se rechazan. Los `ThreadLocal` (`TenantContext`, sesión) los limpian sus filtros; un hilo
  virtual nunca se reutiliza, así que nada pasa a la petición siguiente. Un código nuevo no usa `synchronized` alrededor
  de E/S ni un pool propio para trabajo de E/S.
- **`@Async`**: el executor por defecto es explícito (`applicationTaskExecutor`, hilos virtuales con tope de tareas
  simultáneas): al declarar cualquier otro `Executor`, Spring Boot no crea el suyo y el `@Async` sin nombre cae en un hilo de
  sistema nuevo por tarea sin tope. Un trabajo que debe ir en serie o acotado lleva su propio executor con nombre y
  `@Async("nombre")` (cargas masivas: un solo hilo). Un `@Async` no hereda la organización: usa `TenantScope`.
- **Maven Wrapper**: se usa `./mvnw` (misma versión de Maven en cada máquina y en el pipeline). `.mvn/jvm.config` da a
  Maven los `--add-exports jdk.compiler/…` que necesita google-java-format con Java 25.
- **Estilo**: Spotless (google-java-format AOSP) con `ratchetFrom: origin/main`: `./mvnw spotless:apply` formatea solo los
  archivos cambiados respecto de `main` y `spotless:check` es lo que corre el pipeline. No está enlazado a una fase: un
  build sin historial git debe compilar igual.

### 5.11 Observabilidad
- **Trazas y métricas por OTLP** (Micrometer Tracing + OpenTelemetry), por empuje: **ningún** endpoint de Actuator se expone
  para esto (la regla de SEG7 sigue: solo `health`). Salen del proceso solo con `OBSERVABILITY_EXPORT_ENABLED=true` y las
  direcciones del colector; sin eso la traza existe igual (el `traceId` va en el log y en los errores) pero no sale.
- **`traceId` de la petición** en cada línea del log (`[trace:…]`) y en `errors[0].traceId` de toda respuesta de error
  (`ErrorDetail` lo toma del contexto de log; sin traza, un UUID): quien reclama con ese identificador, el soporte encuentra
  la petición en las trazas y en el log. Nunca se inventa otro identificador para el mismo fin.
- **La organización es un atributo de la traza** (`tenant.code`, clave de **alta cardinalidad**: Micrometer la manda solo a
  la traza) y **nunca una etiqueta de métrica**: una serie por cliente multiplica el coste. Se lee del atributo de la petición
  que fija `MultiTenantFilter` (al terminar la observación el contexto del hilo ya se limpió). Una petición sin organización
  no lleva el atributo, no uno supuesto. Ningún dato personal (email, documento, nombre) en atributos de traza ni en etiquetas.
- Las sondas de salud (`/actuator/**`) no se trazan. El BFF abre la traza y la API la continúa con `traceparent`.
- Prueba: un colector OTLP propio de la suite que decodifica lo recibido (`FakeOtlpCollector`): se comprueba lo que **llegó**,
  no lo que la aplicación dice haber enviado.

### 5.12 La aplicación y la plataforma: una librería sin `main` y un `main` por aplicación
- La plataforma se entrega como una **librería sin clase `main`** (`platform-starter`): todos los `core-*` más la configuración
  transversal (errores RFC 9457, executors con hilos virtuales, observabilidad, reloj y fechas UTC, OpenAPI, Flyway, mensajes,
  logging) y los valores por defecto. **Cada aplicación tiene su propio `main`** (la plataforma sola, un sector): nunca hay dos
  en el mismo programa, y ninguna aplicación de sector depende del módulo ejecutable de la plataforma.
- **Los valores por defecto de la librería tienen la prioridad más baja** (`EnvironmentPostProcessor` que los añade al final):
  lo que la aplicación define (su `application.yml`, sus perfiles, las variables de entorno) los sobrescribe clave a clave. No se
  usa `spring.config.import` para esto: Spring Boot da prioridad a lo importado sobre el archivo que lo importa.
- **La aplicación pone lo suyo**: `spring.application.name`, el puerto, el perfil activo, su bootstrap y sus mensajes. Los
  mensajes se amplían sin copiar los de la plataforma: `spring.messages.basename: ${platform.messages.basenames},i18n/<sector>/messages`.
- Lo que la plataforma arregla en esa configuración llega a todas las aplicaciones al actualizar la versión; ninguna la copia.
- Prueba: un test fija que lo definido por la aplicación gana al valor por defecto y que el valor por defecto solo rellena lo que falta.

### 5.13 Una aplicación de sector sobre la plataforma
- **Qué aporta el sector y qué la plataforma.** La plataforma da identidad (Keycloak), organizaciones, roles, auditoría
  encadenada, cifrado, privacidad, errores y observabilidad; el sector da su dominio. Un sector **no copia** el `core`: depende de
  `platform-starter` (BOM `amachi-platform-dependencies`) y de `platform-test-support` en sus pruebas.
- **Su organización es una extensión de `Organization`** (JOINED, con su discriminador): los datos comunes (razón social,
  identificador fiscal, contactos) son de la plataforma y la tabla del sector solo lleva lo propio. Un sector crea la primera
  organización del bootstrap con una `TenantBootstrapFactory` de su tipo y lo configura en `tenantLocal.type`.
- **Roles = datos.** El sector agrega los suyos al catálogo `AUT_ROLE` en su propia migración (ids desde 101; los de la
  plataforma son 1-3) y el perfil de cada rol lo crea un `DomainEntityProvider` (el rol llega como código y sus datos como
  atributos; nunca por `ThreadLocal`).
- **Eventos y notificaciones propios**: enums del sector que implementan `AuditEventType` y `NotificationType`.
- **El aislamiento por organización es del framework** (`@TenantId`): no existe "buscar por id y organización";
  `findById` de un registro ajeno ya no encuentra nada. Leer entre organizaciones es un acto de plataforma
  (`TenantScope.asPlatform`) y exige revisión de privacidad.
- **El `main` del sector** llama a `DefaultTimeZoneGuard.useUtc()` (la aplicación se niega a arrancar si la zona de la JVM no es
  UTC); las migraciones del sector van desde `V10` (el catálogo en `V10_0_*`); las pruebas traen su propio realm de Keycloak.
- **Rutas públicas de un sector** (portal de reservas, directorio): las declara con un bean `PublicRoutesContributor`, no
  tocando la lista de la plataforma. La plataforma **valida cada patrón al arrancar y se niega a arrancar** si no cuelga de
  `/public/<nombre>/…` (nunca `/public/**`, nunca una ruta protegida de la plataforma): abrir una ruta sin sesión es una decisión
  de seguridad y un sector no puede abrir las de la plataforma. Lo que una ruta pública hace es del sector: expone solo lo que la
  organización publica, valida todo lo que recibe y **limita por IP** las rutas que se pueden martillar; el contador es por
  patrón (un código de confirmación no se prueba "uno a uno" con cuota nueva cada vez). La organización de una ruta pública sale
  del subdominio, o de la cabecera cuando el visitante la eligió en un directorio (`visitorChosenTenant`).
- **Trabajo de fondo y datos iniciales de un sector**: entran a una organización con `TenantScope` (`inTenant` abre su sesión y
  transacción; `inTenantContext` para tareas que conservan sus propias transacciones; `asPlatform` para lo que lee varias
  organizaciones a propósito), nunca con `TenantContext.set…/clear`. Sin ámbito, el filtro de `@TenantId` falla cerrado o
  devuelve vacío en silencio.
- **El tiempo**: lo ocurrido es un instante en UTC (bean `Clock`, que los tests mueven); el día de calendario de la organización sale
  de un reloj propio del sector con su zona; una entidad, que no puede inyectar, usa `now(ZoneOffset.UTC)`. El sector copia el
  `TimeArchitectureTest` de la plataforma para que ningún `now()` sin zona vuelva a entrar.
- **Eventos de auditoría propios**: el enum del sector implementa `AuditEventType` (se guarda por su `code()` en la cadena de la
  organización, igual que los de la plataforma). El mensaje es una clave `AuditMessage.of("clave", "param", valor…)`: no se escribe
  una frase en el backend; la traduce el frontend con `audit_message.<clave>` en cada idioma, así que toda clave nueva se agrega al
  diccionario del frontend del sector. La retención (`app.audit.retention.days`) del sector no se acorta sin una decisión
  explícita: un registro clínico se conserva por ley.
- **Lecturas de datos personales**: cada `GET` que devuelve datos de una persona lleva `@AuditedRead("Tipo")` en el método del
  controlador (la lectura que salió bien deja un evento con tipo e id; la rechazada no). El sector fija con una prueba de
  arquitectura la lista de controladores y exige la anotación en cada `GET`; lo que ya registra un evento propio más específico
  (p. ej. el acceso entre organizaciones) queda fuera.
- **Actos sensibles** (firmar un documento clínico, aprobar un pago): el sector llama a `RecentAuthentication.require()` al
  empezar el acto; sin una autenticación de hace segundos (`auth_time`) responde `401` con
  `WWW-Authenticate: Bearer error="insufficient_user_authentication"` y el cliente entra de nuevo por el BFF
  (`reauthenticate=true`). Ninguna contraseña viaja a la API (ADR-0007).
