# ADR-0011: Regla de tiempo: instantes en UTC, el día lo pone la organización

- **Estado**: Aceptada
- **Fecha**: 2026-11-07
- **Decide**: Juan Amachi (decisión D3 de la adopción de Vitalia, aplicada desde el corte 2 y confirmada aquí)
- **Controles ISO 27001**: A.8.15 (registros con hora fiable y comparable), A.8.17 (sincronización de relojes), A.8.9 (configuración)
- **Resume**: `ARQUITECTURA.md` §7 (que sigue siendo el texto de la regla)

## Contexto

Una aplicación guarda dos cosas distintas que se llaman «fecha»: **lo que ocurrió** (un inicio de sesión, una firma, una dosis: un
instante) y **el día de calendario de una organización** (la cita del lunes, el turno, «hoy» para un hospital de La Paz). Mezclarlas
da errores silenciosos: un diagnóstico guardado a las 21:00 de La Paz es del día siguiente en UTC, una cita «a las 9:00» no es un
instante, y una comparación entre el «hoy» del servidor y el del hospital falla solo entre las 00:00 y las 04:00 UTC (lo vivimos: un
escenario de la línea de tiempo clínica falló en el CI por eso).

## Decisión

1. **Todo instante guardado es UTC** y la JVM corre en UTC (`DefaultTimeZoneGuard.useUtc()` en cada `main`; no arranca en otra zona).
   El instante sale del bean `Clock`; las entidades, que no pueden inyectar, usan `now(ZoneOffset.UTC)`. Un instante viaja con `Z`.
2. **El día de calendario es de la organización**: se pregunta a su reloj (`HospitalClock.today()` / `zone()` en Vitalia; en general un
   `TenantClock`), nunca a `LocalDate.now()` del servidor. Una fecha sin hora es `LocalDate` y viaja como `yyyy-MM-dd`.
3. **Lo que no es un instante no se guarda como instante**: la hora de una cita se guarda con su zona de organización, no como UTC
   a ciegas.
4. **Se vigila**: `TimeArchitectureTest` (plataforma y cada sector) prohíbe `now()` sin reloj ni zona.
5. **Las pruebas siguen la regla**: un escenario que compara con «hoy» usa el día **de la organización**
   (`HospitalClock.zoneOf(código)`), nunca el del servidor.

## Consecuencias

- Los valores por defecto de fecha-sin-hora de tres entidades de Vitalia (`DoctorHospitalAssignment.startDate`,
  `MedicalHistory.recordDate`, `Referral.referralDate`) usan UTC porque una entidad no inyecta el reloj: es un límite conocido, y el
  servicio que las crea puede fijar la fecha del hospital.
- Toda fecha nueva se clasifica al diseñarla (¿instante o día de la organización?) antes de elegir el tipo.
