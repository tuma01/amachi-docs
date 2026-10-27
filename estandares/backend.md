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
- Datos personales sensibles cifrados con `@Convert(EncryptedStringConverter)` (AES-256, clave en variable de entorno,
  nunca en el repositorio). Con IV determinístico se conservan `UNIQUE` y la búsqueda exacta; en columnas cifradas
  solo se busca por igualdad (nunca `LIKE` ni `lower()`).
- Clases que instancia JPA/Envers **fuera de Spring** (`EncryptedStringConverter`, `AuditRevisionListener`) no admiten
  `@Autowired`/`@Value`: leen de holders estáticos.
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
Una ruta nueva que crea algo que no debe duplicarse se agrega a `IdempotencyProperties.PATHS`.

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
