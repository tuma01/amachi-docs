# ADR-0010: La reserva pública no vincula a quien no probó ser la persona de la ficha

- **Estado**: Aceptada
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

**C ahora; D como mejora posterior.**

1. **Identidad verificada** (`identityVerified`) = nadie con ese documento es conocido (persona nueva), o la fecha de nacimiento
   que trae la reserva es **la de la ficha**. Una persona conocida cuya fecha **no coincide o no consta** queda **no verificada**.
2. **Nunca se rechaza por esto**: la respuesta es una reserva pendiente, como cualquier otra. No hay error que revele nada.
3. **Una reserva no verificada no se vincula a la ficha** y **no se confirma sola** (ni en una agenda de confirmación automática ni
   por el reloj): permanece pendiente (y vence si nadie la atiende).
4. **Admisión la confirma comprobando el documento**: `POST /booking/requests/{id}/confirm?identityChecked=true`. Sin ese
   parámetro, `409` (`booking.request.identity_review_required`). Queda **quién lo comprobó** (`identityVerifiedBy`) y, al confirmar,
   la reserva se vincula a la ficha como cualquier otra.
5. **Lo que ve quien reserva no cambia con esto**: la vista pública no trae `identityVerified`, ni el vínculo (`patientId`).
6. **Pendiente (D)**: una prueba de posesión automática. Mientras no esté, la **diferencia observable** que queda es que, en una agenda
   de confirmación automática, una reserva no verificada queda pendiente y una verificada se confirma al instante: quien pruebe
   documentos y fechas puede deducir que «existe una ficha con otra fecha». Lo limita el control de peticiones por IP de las rutas
   públicas y la regla de una reserva activa por persona y día; **no lo elimina**. D lo elimina.

## Consecuencias

- Columnas nuevas en `BKG_REQUEST` (`IDENTITY_VERIFIED`, `IDENTITY_VERIFIED_BY`); la base de desarrollo se recrea (se editó el CREATE).
- Contrato nuevo para el frontend (etapa 5): `identityVerified` en la bandeja de Admisión y el parámetro `identityChecked` al confirmar.
- Un paciente que se equivoca de fecha ya no recibe un error: queda pendiente y Admisión lo resuelve (mejor experiencia y sin oráculo).
- La plataforma no cambia; el principio sirve para InverPulse (un cliente que «reserva» una llamada con un asesor).
