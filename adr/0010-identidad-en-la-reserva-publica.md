# ADR-0010: La reserva pública no vincula a quien no probó ser la persona de la ficha

- **Estado**: Aceptada (C el 2026-11-06; D el 2026-11-07)
- **Fecha**: 2026-11-06
- **Decide**: Juan Amachi (decisión abierta del code review de la adopción de Vitalia: «reserva con ficha sin fecha de nacimiento»)
- **Controles ISO 27001**: A.5.15 (control de acceso), A.8.3 (restricción de acceso a la información), A.5.34 (privacidad) · Ley 25 /
  RGPD: datos de salud son sensibles; el acceso a la ficha requiere probar quién es la persona
- **Aplica a**: toda aplicación de sector con un portal público que reserva o solicita algo a nombre de una persona ya registrada
  (hoy, `vitalia-medical-core/booking`)

## Contexto

En una reserva sin cuenta, el número de documento lo escribe quien reserva: **nadie probó que sea suyo**. El código comparaba la fecha de
nacimiento con la de la ficha y, si no coincidía, **respondía con un error**; si la ficha no tenía fecha, **dejaba pasar** y vinculaba
la reserva a la ficha. Tres problemas:

1. **Oráculo de existencia.** El error aparece solo si el documento es de un paciente conocido: cualquiera averigua qué documentos
   son de pacientes del hospital (dato de salud: «esta persona se atiende aquí»).
2. **Vínculo sin prueba.** Quien conoce un documento (que no es un secreto) ponía una cita, con sus propios datos de contacto, en la ficha
   de otra persona.
3. **Ficha sin fecha de nacimiento**: no había nada que comparar y se aceptaba.

Cómo lo resuelven los portales profesionales de salud y banca (por lo que conocemos, no verificado aquí con sus manuales): **no
revelan si la persona existe**, y no la vinculan a su historial hasta una **prueba de posesión** (un código de un solo uso al contacto
que ya consta, o la revisión del documento en persona).

## Opciones consideradas

1. **A. Bloquear** la reserva cuando la ficha no tiene fecha o no coincide. Evita el vínculo, pero mantiene el oráculo y deja a esos
   pacientes sin reserva en línea.
2. **B. Dejar pasar** (lo que había para la ficha sin fecha). Sin trabajo; el vínculo sin prueba sigue.
3. **C. No rechazar y no vincular.** La reserva se acepta siempre; si la persona no se pudo verificar, queda **sin vincular** y
   **pendiente de revisión**; Admisión la confirma después de **revisar el documento**.
4. **D. C más prueba de posesión automática** (código de un solo uso al correo o teléfono de la ficha), sin intervención de Admisión.

## Decisión

**C el 2026-11-06 y D el 2026-11-07.**

1. **Identidad verificada** (`identityVerified`) = nadie con ese documento es conocido (persona nueva), o la fecha de nacimiento
   que trae la reserva es **la de la ficha**. Una persona conocida cuya fecha **no coincide o no consta** queda **no verificada**.
2. **Nunca se rechaza por esto**: la respuesta es una reserva pendiente, como cualquier otra. No hay error que revele nada.
3. **Una reserva no verificada no se vincula a la ficha** y **no se confirma sola** (ni en una agenda de confirmación automática ni
   por el reloj): permanece pendiente (y vence si nadie la atiende).
4. **Admisión la confirma comprobando el documento**: `POST /booking/requests/{id}/confirm?identityChecked=true`. Sin ese
   parámetro, `409` (`booking.request.identity_review_required`). Queda **quién lo comprobó** (`identityVerifiedBy`) y, al confirmar,
   la reserva se vincula a la ficha como cualquier otra.
5. **Lo que ve quien reserva no cambia con esto**: la vista pública no trae `identityVerified`, ni el vínculo (`patientId`).
6. **D, hecha (2026-11-07): prueba de posesión con un código de un solo uso.** Cuando la persona es conocida y no probó su fecha de
   nacimiento, se envía un código de 6 dígitos **al correo de la ficha, nunca al del formulario**; la persona que reservó lo escribe en
   `POST /public/booking/request/{código}/verify-identity` y la reserva queda verificada y vinculada, sin Admisión (en una agenda de
   confirmación inmediata se confirma en ese momento; si el turno ya se ocupó, queda pendiente y verificada). Como los portales de
   salud y banca: se guarda solo la huella SHA-256 del código, vale 15 minutos, sirve una vez, tiene 5 intentos, como mucho 3 códigos por
   documento y hora (nadie puede inundar la bandeja de otra persona) y la ruta tiene límite por IP. **Todo fallo** (código malo, vencido,
   sin intentos, sin código, reserva inexistente o ya resuelta) responde con **el mismo 409**: la verificación no revela nada. El correo
   del código no lleva datos de quien reservó (ni nombre ni teléfono), solo el turno pedido, y dice que se ignore si no fue la persona. Sin
   correo en la ficha, o con el tope de códigos alcanzado, sigue la revisión de Admisión de la opción C.
7. **Riesgo residual aceptado** (ninguna combinación lo elimina del todo): quien reserva con un documento ajeno y **su propio correo**
   recibe un código solo si el documento no tiene ficha (se le trata como persona nueva); con ficha, el código va a otro correo. Esa
   diferencia entre «reserva nueva» y «reserva a la espera de un código» sigue siendo observable, y solo se evitaría revisando a mano
   toda reserva pública, lo que mata la reserva inmediata. Lo limitan el control de peticiones por IP, una reserva activa por persona y
   día, y que lo único que se averigua es «ese documento tiene o no ficha» sin ningún dato clínico. Decisión de Juan Amachi (2026-11-07):
   código solo para conocidos y riesgo residual documentado.

## Consecuencias

- Columnas nuevas en `BKG_REQUEST` (`IDENTITY_VERIFIED`, `IDENTITY_VERIFIED_BY`); la base de desarrollo se recrea (se editó el CREATE).
- Contrato nuevo para el frontend (etapa 5): `identityVerified` en la bandeja de Admisión y el parámetro `identityChecked` al confirmar.
- Un paciente que se equivoca de fecha ya no recibe un error: queda pendiente y Admisión lo resuelve (mejor experiencia y sin oráculo).
- Contrato nuevo para el frontend: `POST /public/booking/request/{código}/verify-identity` con `{ "code": "…" }` (200 verificada; 409 con el mismo mensaje en cualquier fallo). El portal ofrece siempre «¿Recibió un código?» para no revelar quién tiene ficha.
- Columnas nuevas en `BKG_REQUEST`: `IDENTITY_CODE_HASH`, `IDENTITY_CODE_EXPIRES_AT`, `IDENTITY_CODE_ATTEMPTS` (no auditadas en Envers).
- La plataforma no cambia; el principio sirve para InverPulse (un cliente que «reserva» una llamada con un asesor).
