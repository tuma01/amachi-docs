# Estándar de base de datos

> Parte de la arquitectura del ecosistema (`../ARQUITECTURA.md`). Conserva su número de sección (§3.3).

### 3.3 Base de datos

- **Una base por despliegue**, MySQL 8.4 + Flyway. Nombre en desarrollo: **`platform_db`** (cada aplicación puede
  usar el suyo en producción: `inverpulse_db`). Credenciales **solo por variables de entorno** (`DB_USERNAME`,
  `DB_PASSWORD`); ninguna contraseña en los archivos del repositorio.
- **Prefijo de tabla = trigrama del módulo** (`<TRI>_<ENTIDAD>`, mayúsculas, singular): plataforma `GEO_`, `DMN_`,
  `AUT_`, `MGT_`; cada sector reserva los suyos y los registra en la tabla de §3.1 (ej. InverPulse: `MKT_` datos de
  mercado, `SIG_` señales, `AGT_` agentes, `PTF_` carteras/simulador). Dos módulos nunca comparten trigrama.
- **Versiones Flyway por rango** (no chocan entre módulos): `V2_*` geografía, `V3_*` dominio, `V4_*` autenticación,
  `V8_*` gobierno; **`V10_*` en adelante** para los sectores, un rango por módulo (ej. `V20_*` mercado, `V21_*`
  señales). Archivo: `V<rango>_<n>__create_<tri>_<entidad>.sql`. Cada módulo guarda sus migraciones en
  `src/main/resources/db/migration`; Flyway las encuentra todas en el classpath.
- Claves foráneas `FK_ID_<ENTIDAD>`; restricciones con nombre (`UK_…`, `FK_…`, `IDX_…`); columnas de auditoría y de
  tenant las aportan las clases base (`AuditableTenantEntity`).
- **Fase de desarrollo: sin `ALTER TABLE`** — se edita el `CREATE TABLE` original y se recrea la base.
- Toda columna nueva en una entidad `@Audited` va también en su tabla `_AUD` (si no, Hibernate no arranca).
- Nunca una columna cuyo nombre termine en una palabra reservada que el driver pueda leer como cláusula (`..._INTO`:
  MySQL Connector/J toma `SELECT ... x_into` por `SELECT ... INTO` y rompe toda lectura de la tabla).
- Las pruebas manuales y visuales **nunca** corren sobre la base de desarrollo del usuario: se usa un esquema de
  prueba (`<app>_sN`) que se borra al terminar.
