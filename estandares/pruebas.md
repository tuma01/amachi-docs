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
- **No compilar con Maven mientras el backend corre en el IDE** (rompe sus clases en caliente): validar en una copia
  aislada del repositorio.
- Método para cambios que se repiten: template → **un** ejemplo validado → el resto, abriendo cada archivo.
- Lint sin advertencias nuevas; `tsc` de la app y de los specs.
- Git: commits del usuario, mensajes de una línea `<CLAVE>-NNN tipo(scope): descripción`.
- Ramas: `main` estable y solo por pull request; una rama por cambio, `<tipo>/<CLAVE>-NNN-descripcion-corta`
  (tipos `feature`, `fix`, `docs`, `refactor`, `chore`, `test`, `hotfix`); `release/x.y` solo para versiones
  congeladas. Claves: `DOC` en `amachi-docs`. Propuesta pendiente de confirmar: `AMA` plataforma, `VIT` Vitalia,
  `INV` InverPulse, con un número compartido entre backend y frontend de la misma tarea.
