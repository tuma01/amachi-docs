# ADR-0008: Las solicitudes de derechos de la persona pueden registrarlas la organización en su nombre

- **Estado**: Aceptada
- **Fecha**: 2026-11-06
- **Decide**: Juan Amachi (opción B de la decisión abierta del corte 5 de la adopción de Vitalia)
- **Completa**: ADR-0006 (identidad en Keycloak) y el módulo `core-privacy` (PRIV1–PRIV4)

## Contexto

`core-privacy` atiende el derecho de supresión (Ley 25 · RGPD art. 17), pero solo acepta la solicitud de **la propia persona,
con sesión**, y la identifica por **cuenta de usuario** (`FK_ID_USER NOT NULL`). Eso deja fuera lo que la ley también obliga a
atender:

- La solicitud que llega **por otro canal**: carta, correo, en ventanilla. El RGPD (art. 12) y la Ley 25 no piden cuenta ni portal.
- La que presenta **un representante** (un familiar, un tutor) o ordena **una autoridad** (orden judicial).
- La de una persona **sin cuenta**. Un hospital registra pacientes en recepción que nunca tendrán usuario; una firma de corretaje
  tiene clientes que escriben por correo. Hoy la plataforma no puede atenderlos.

Para suplirlo, Vitalia tiene su propio registro (`DataErasureRequest`, `/data-erasure-requests`) con tipo de solicitante y retención
legal, y su propia anonimización de identidad y cuenta, que duplica la de la plataforma y puede divergir (no cierra la cuenta en
Keycloak igual, no deja la misma prueba). La plataforma servirá a varias aplicaciones de sector (salud, trading…): cada una
inventaría su propio registro.

Cómo lo resuelven las plataformas de privacidad y los grandes SaaS multi-organización (por lo que conocemos, no verificado aquí con
sus manuales): **un solo objeto «solicitud»** con **canal**, **tipo de solicitante** y **verificación de identidad**, creado por la
persona o por un administrador en su nombre, con el mismo flujo de respuesta; la ejecución la hacen conectores de cada módulo.

## Opciones consideradas

1. **A. Cada sector con su registro** (lo que hay hoy). Sin trabajo en la plataforma; N registros divergentes, y el hueco de la
   persona sin cuenta sigue en los demás.
2. **B. Registro general en `core-privacy`.** La solicitud pertenece a la **persona** (no a la cuenta), tiene canal, tipo de
   solicitante, verificación de identidad y retención legal, y puede registrarla un administrador. Un solo flujo y una sola
   ejecución (`PersonalDataEraser`). Cuesta tocar el modelo de `core-privacy`.
3. **C. Quitar el registro de Vitalia sin reemplazo.** La más simple; la organización pierde cómo atender familiares y órdenes
   judiciales.

## Decisión

**B.**

1. **La solicitud es de la persona.** `FK_ID_PERSON` es la identidad; `FK_ID_USER` pasa a ser opcional (la cuenta, si la tiene).
   El límite de «una abierta a la vez» es por persona y organización.
2. **Datos de la solicitud, comunes a los derechos** (`DataSubjectRequest`): `channel` (`PORTAL`, `EMAIL`, `LETTER`, `IN_PERSON`),
   `requesterType` (`DATA_SUBJECT`, `REPRESENTATIVE`, `LEGAL_ORDER`), `registeredBy` (el administrador que la registró; vacío en
   `PORTAL`), `identityVerified` (el administrador atestigua que comprobó el documento de quien la presenta; obligatorio fuera de
   `PORTAL`, donde la identidad la prueba Keycloak), `proofReference` (referencia del documento que la respalda, p. ej. «oficio
   N.º 123/2026»; se conserva como prueba) y `requesterName` (quien la presenta si no es la persona; dato personal: cifrado y
   borrado al responder, igual que el motivo).
3. **Retención legal**: `legalHoldUntil` (fecha). Hasta esa fecha **no se puede aprobar**: la organización conserva por una
   obligación que debe poder probar. Se fija al registrar y un administrador puede ponerla o quitarla después, con su motivo.
4. **API**: `POST /privacy/erasure-requests/on-behalf` (administrador con acceso total —nivel 1— o SuperAdmin) con la persona, el
   canal, el tipo de solicitante, la referencia y la atestación. La persona debe pertenecer o haber pertenecido a la organización
   del administrador; si no, 404 (no se revela quién existe). La propia persona sigue usando `POST /privacy/erasure-requests`.
   Aprobar y rechazar no cambian: otro paso del mismo flujo, con el mismo plazo de 30 días y el mismo `outcome`.
5. **Persona sin cuenta**: la ejecución usa la persona, no la cuenta, y las reglas que impiden irse todavía (una internación en curso) pasan a preguntarse por la **persona** con un punto de extensión nuevo, `ErasureGuard` (`core-common`); las de la cuenta (`AccountDeactivationGuard`, p. ej. el último administrador) se siguen preguntando si hay cuenta. Los módulos que borran por cuenta (`core-auth`,
   `core-management`) no hacen nada si no hay; el guardián del último administrador solo se pregunta si hay cuenta; los avisos
   en la aplicación solo van a los administradores (a la persona se le contesta por el canal por el que escribió).
6. **Sin cuatro ojos obligatorio.** El administrador que registra puede ser el que aprueba, porque una organización pequeña puede
   tener un solo administrador de nivel 1. Queda **auditado** quién registró y quién respondió, y el canal, sin datos personales.
   Endurecerlo es una opción de configuración futura.
7. **El sector deja de tener registro propio.** Vitalia elimina `DataErasureRequest` y `/data-erasure-requests`, y la
   anonimización de persona y cuenta de `PatientAnonymizationService`; conserva `PatientPersonalDataEraser` (expediente) y
   `PatientErasureGuard`, y su consulta de elegibilidad (informativa: la pantalla muestra los bloqueos antes de registrar).
8. **Esquema**: se edita el `CREATE` de `PRV_ERASURE_REQUEST` (y de la solicitud de rectificación, que comparte la clase base)
   en el mismo archivo, sin `ALTER` (la base se recrea en desarrollo; el PR lo avisa).

## Consecuencias

- Se gana: un solo registro y un solo flujo para todos los sectores; se cubre a quien no tiene cuenta; la prueba de la
  solicitud (canal, quién la presentó, quién verificó, quién respondió) queda en un solo lugar y sin datos personales.
- Cuesta: cambia el modelo y el contrato de `core-privacy` (campos nuevos, `userId` opcional) y hay que revisar cada
  `PersonalDataEraser`/`PersonalDataContributor` ante una cuenta ausente. Vitalia pierde su pantalla de solicitudes y usa la de la
  plataforma, y su API de `/data-erasure-requests` desaparece (el frontend, etapa 5, usa la nueva).
- Riesgo: un administrador de nivel 1 puede registrar la supresión de cualquier persona de su organización. Es el mismo poder que ya
  tiene al aprobarla; se mitiga con la atestación, la referencia obligatoria, el límite a su organización y la auditoría.
- Fuera de alcance, queda abierto: aplicar lo mismo al **acceso/portabilidad** y a la **rectificación** (la clase base ya trae los
  campos; falta exponer el registro en su nombre); exigir reautenticación reciente (ADR-0007) a la persona que pide su propia
  supresión; el cuatro ojos configurable.
- Se registra en `amachi-platform/docs/PLATAFORMA.md` (PRIV5), `amachi-docs/estandares/backend.md` §5.13 y `CHANGELOG.md`.
