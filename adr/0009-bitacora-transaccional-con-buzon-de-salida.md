# ADR-0009: La bitácora se escribe con el cambio (buzón de salida transaccional) y un acto sin registro no se hace

- **Estado**: Aceptada
- **Fecha**: 2026-11-06
- **Decide**: Juan Amachi (decisión abierta del code review de la adopción de Vitalia: «la bitácora no reintenta»)
- **Controles ISO 27001**: A.8.15 (registro), A.5.28 (recogida de evidencias), A.5.33 (protección de registros), A.8.16
  (monitoreo) · Ley 25 / RGPD / ISO 27701: quién accedió a datos personales debe poder probarse
- **Completa**: SEG11 (bitácora encadenada por HMAC) y `seguridad/iso27001.md`

## Contexto

La bitácora (`AUT_AUDIT_EVENTS`, encadenada por organización) se escribía **después del commit** del cambio, en una transacción
propia, y **un error al escribirla se tragaba** («una falla de auditoría no rompe el negocio»). Dos huecos:

- El cambio se confirmaba y su registro **se perdía** si la base fallaba justo entonces (o la JVM caía entre el commit y la escritura):
  un cambio sin rastro. Una bitácora que pierde hechos no sirve de prueba ante un incidente ni ante una solicitud de acceso.
- Una **lectura** de datos personales (`@AuditedRead`) se registraba después de haber respondido: si el registro fallaba, el dato ya
  había salido.

## Opciones consideradas

1. **A. Dejarlo como estaba.** Sin trabajo; los dos huecos siguen.
2. **B. Reintentar en memoria.** Se pierde con la JVM; no cubre la caída.
3. **C. Buzón de salida transaccional (*transactional outbox*).** El evento se escribe en una tabla **en la misma transacción que el
   cambio**; un relevo lo entrega a la cadena con reintentos. Es el patrón de los sistemas que no pueden perder un hecho.

## Decisión

**C.**

1. **Se escribe con el cambio.** `AuditService.registerEntityEvent` inserta el evento en `AUT_AUDIT_OUTBOX` **en la transacción del
   cambio**: o se guardan los dos o ninguno. Si el cambio se revierte, su registro también. Si no se puede escribir el registro,
   **el cambio falla** (un cambio sin registro no se permite).
2. **Se entrega a la cadena** (`AuditOutboxRelay`): justo después del commit y, si falla, **cada 30 s** con espera creciente
   (2, 4, 8… hasta 60 min). Cada evento entra a la cadena y sale del buzón en **una sola transacción**, y la entrega es idempotente
   (el id del evento es el `EXTERNAL_ID` único de la cadena): no se duplica ni se pierde. Con varias instancias, `FOR UPDATE SKIP
   LOCKED`: dos no entregan el mismo evento.
3. **Una lectura sin registro no se responde.** Fuera de una transacción de escritura (las lecturas), el evento se guarda en una
   transacción propia **antes** de responder; si ni eso se puede, la petición falla con **503** (`audit.error.unavailable`) y el dato
   no sale (falla cerrado).
4. **Los eventos que ya registran un rechazo** (`registerEvent`: un inicio de sesión fallido…) siguen sin romper el flujo, pero si la
   cadena no los toma van al buzón en vez de perderse.
5. **El buzón no guarda datos en claro.** El evento viaja como JSON **cifrado con AES-256-GCM** (lleva el correo de quien actuó).
6. **Monitoreo.** Cada pasada del relevo registra un **error** si hay eventos con más de 15 minutos de espera: la bitácora va atrasada
   y alguien debe mirar. Una fila vive en el buzón solo hasta entregarse.

## Consecuencias

- Una tabla nueva (`AUT_AUDIT_OUTBOX`, `V4_21`) y tres clases (`AuditOutbox`, `AuditOutboxRelay`, `AuditUnavailableException`).
- Un poco más de escritura por acto auditado (una fila que se borra al entregarse).
- Una caída de la base o de la cadena **detiene los actos auditados** en vez de dejarlos sin rastro: es la intención.
- Los sectores no cambian nada: usan el mismo `AuditService`. Sus pruebas esperan que el evento aparezca (llega justo tras el commit).
- **Brecha que queda**: el buzón vive en la misma base que la bitácora; si se pierde la base se pierden ambos (las copias de seguridad
  son del operador, `seguridad/iso27001.md` §4). Un registro en un almacén aparte (WORM) sería una decisión posterior.
