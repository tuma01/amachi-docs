# Estándar de pruebas y calidad

> Parte de la arquitectura del ecosistema (`../ARQUITECTURA.md`). Conserva su número de sección (§10).
> La lista de verificación de un feature está en `../plantillas/checklist-feature.md` (§11).

## 10. Pruebas y calidad

**Nada se declara "validado" sin los 4 puntos, en cada rol que usa la funcionalidad:**

1. **En pantalla (Playwright), con las acciones reales**: crear **y** editar; mover fechas/horas hacia adelante y
   hacia atrás; obligatorios vacíos; texto sin elegir de una lista; valores fuera de rango; acciones de otro estado.
   Cada mensaje dice **qué hacer**.
2. **Un test unitario por cada regla** (JUnit en backend con `Clock.fixed`; Vitest por cada regla de formulario).
3. **Un escenario Cucumber por cada caso de la vida real**, con el cuerpo que **envía de verdad** la pantalla, con cada
   rol que lo usa y los que no deben poder (403).
4. **Plan de pruebas manuales recorrido antes de entregarlo**: cada paso con *rol · menú › submenú · botón · qué debe
   verse*.

- **Verificar, nunca deducir**: todo dato que se afirma (estado de git y de las ramas, qué archivos cambiaron, qué compila, qué
  pasa en una prueba, qué hace una dependencia, si un método se usa) se **comprueba con el comando o la lectura** en el momento de
  afirmarlo. Si no se pudo comprobar, se dice «no verificado» y por qué; una deducción nunca se presenta como hecho. En concreto:
  antes de dar comandos de `git` se mira `git status`, `git branch` y `git log`; antes de borrar o dar por «sin uso» un
  método se busca quién lo usa; un refactor masivo se compila **en cada repositorio afectado** antes de entregarlo; y un
  resultado de otra ejecución (otra rama, otro día) no vale para ésta.
- **El paso del tiempo es parte de la validación**: días pasados, hoy antes y después de la hora, mañana, cierre del
  día, en la zona de la organización (Cucumber con un reloj movible y pasos "la hora es…", "pasan N minutos").
- **Contrato**: el `openapi.json` nuevo difiere del anterior solo en lo esperado.
- Datos de prueba nombrados "PRUEBA…" y borrados al terminar; nunca pruebas destructivas sobre datos sembrados.
- **Una prueba que no se ejecuta no prueba nada**: el `pom` del módulo de integración lista qué clases corre (`includes`); una clase
  JUnit nueva fuera de esos patrones pasa en el IDE y nunca en la suite. Al agregar una, confirmar que aparece en
  `Running …` de la salida del build completo.
- **Pipeline** (GitHub Actions, un `ci.yml` por aplicación): `./mvnw -B install` con Java 25 y Docker (la suite completa, sin ninguna variable de
  entorno propia de una máquina), `spotless:check`, SBOM CycloneDX, gitleaks en cada pull request, Dependency-Check periódico y
  `main` protegida exigiendo el build. Toda acción de terceros fijada por commit (versión en un comentario); un secreto, solo
  de lectura y con un único propósito. Un workflow que nadie ejecutó no está validado: decir que no se probó en GitHub.
- **Arquitectura verificada con ArchUnit**, en cada build: dirección de los módulos sin ciclos, capas (un controlador no usa
  repositorios, la persistencia no sube a servicios) y nombres (`*Controller`, `*Repository`, `*Service`). Una regla que falla
  se corrige en el código, no se silencia; una regla que no encuentra clases falla (no queda vacía en silencio).
- **Escaneo dinámico (OWASP ZAP)** de la API levantada de verdad y autenticada, periódico y a mano, no en cada pull request. Se
  empieza solo informando; las reglas que deban fallar el build se añaden tras el primer informe. Para pasar el token a ZAP, la
  variable `ZAP_AUTH_HEADER_VALUE` (la sintaxis `replacer` con espacios no llegó a las peticiones: comprobado).
- **Infraestructura de prueba compartida**: contenedores, cliente de Keycloak, pasos HTTP y reloj movible viven en un módulo
  publicable (`platform-test-support`) que cada aplicación importa con `<scope>test</scope>`; nadie la copia. Cada suite trae su
  realm de Keycloak (las cuentas de la plataforma más las suyas) y lo señala con `-Dplatform.test.keycloak-realm`. Lo propio
  de una aplicación (sus pasos, sus datos, sus reglas de arquitectura) se queda en su módulo de pruebas.
- **Actos que exigen autenticación reciente** (firmar un documento clínico, ADR-0007): el inicio de sesión directo con contraseña de las
  pruebas **no trae `auth_time`** y la API lo rechaza (`401` `SEC_017`, falla cerrada). La suite usa el paso compartido
  `el usuario inició sesión hace un momento como …` (inicio de sesión real por las páginas de Keycloak con PKCE); el cliente de navegador
  del realm de pruebas lleva la audiencia de la API. Una cuenta con segundo factor obligatorio (administradores) no puede usarlo: los
  escenarios donde un administrador *no* debe firmar usan el inicio de sesión directo. Probar también el rechazo: firmar sin sesión reciente
  da `401`.
- **Una aplicación que adopta la plataforma porta su suite completa antes de darla por adoptada.** La compilación y los tests unitarios no
  vieron nada de lo que destapó la suite de Vitalia: búsquedas que comparaban columnas cifradas, lecturas entre organizaciones sin el
  ámbito de plataforma, hilos asíncronos y *schedulers* sin organización activa, y datos y permisos que la plataforma cambió. Los
  escenarios que duplican a los de la plataforma (login, invitaciones, gobernanza) se borran en la aplicación; los propios del sector se portan.
- **No compilar con Maven mientras el backend corre en el IDE** (rompe sus clases en caliente): validar en una copia
  aislada del repositorio.
- Método para cambios que se repiten: template → **un** ejemplo validado → el resto, abriendo cada archivo.
- Lint sin advertencias nuevas; `tsc` de la app y de los specs.
- Git: commits del usuario, mensajes de una línea `<CLAVE>-NNN tipo(scope): descripción`.
- Ramas: `main` estable y solo por pull request; una rama por cambio, `<tipo>/<CLAVE>-NNN-descripcion-corta`
  (tipos `feature`, `fix`, `docs`, `refactor`, `chore`, `test`, `hotfix`); `release/x.y` solo para versiones
  congeladas. Claves: `DOC` en `amachi-docs`. Propuesta pendiente de confirmar: `AMA` plataforma, `VIT` Vitalia,
  `INV` InverPulse, con un número compartido entre backend y frontend de la misma tarea.
