# Plan de pruebas manuales: <feature>

> Punto 4 de la validación (`estandares/pruebas.md` §10): se recorre completo **antes** de entregar. Cada paso con
> *rol · menú › submenú · botón · qué debe verse*. Datos de prueba con nombre `PRUEBA…`, borrados al terminar;
> nunca sobre datos sembrados ni sobre la base de desarrollo del usuario.

- **App / versión**:
- **Entorno y organización (tenant)**:
- **Fecha y responsable**:

## Casos

| # | Rol | Ruta (menú › submenú) | Acción | Resultado esperado | OK |
|---|---|---|---|---|---|
| 1 | ADMIN | Organización › Usuarios | Botón «Invitar», correo válido | Aviso de éxito; la invitación aparece en la lista con estado «Pendiente» | ☐ |
| 2 | USER | (URL directa del caso 1) | Abrir la pantalla | 403: la pantalla no se ofrece en su menú y el servidor la rechaza | ☐ |

Cubrir, en cada rol que usa la funcionalidad y en los que no deben poder (403):

- Crear **y** editar; obligatorios vacíos; valores fuera de rango; texto sin elegir de una lista.
- Fechas y horas hacia adelante y hacia atrás, en la zona de la organización (hoy antes y después de la hora,
  mañana, cierre del día).
- Acciones no permitidas en el estado actual (deben estar desactivadas y explicar por qué).
- Cada mensaje de error dice **qué hacer**.

## Hallazgos

| # | Caso | Descripción | Gravedad | Estado |
|---|---|---|---|---|
