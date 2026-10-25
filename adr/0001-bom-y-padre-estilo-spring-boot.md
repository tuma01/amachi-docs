# ADR-0001: BOM y POM padre separados, al estilo Spring Boot

- **Estado**: Aceptada
- **Fecha**: 2026-10-25
- **Decide**: Juan Amachi

## Contexto

- Un único artefacto `com.amachi.core:amachi-bom` hacía de BOM y de POM padre a la vez, y lo heredaban Vitalia y
  Amachi Platform con la misma versión `2.0.0-SNAPSHOT`: cualquier cambio en esa versión llegaba a las dos apps.
- La revisión del 2026-10-24 encontró versiones incoherentes: Spring Cloud de la línea de Boot 3.5, módulos de
  Testcontainers 1.21 sobre el núcleo 2.0, Mockito y otras librerías fijadas por debajo de lo que trae Spring Boot,
  JUnit 5 en los módulos de pruebas y JUnit 6 en el resto.
- Un proyecto Maven solo puede tener un padre; un BOM se puede importar en cualquier proyecto.

## Opciones consideradas

1. **Mantener un solo POM padre con versiones**: simple; pero un proyecto que deba heredar otro padre no puede usar
   las versiones, y versiones y build quedan mezclados.
2. **BOM puro + POM padre que lo hereda** (diseño de `spring-boot-dependencies` + `spring-boot-starter-parent`):
   una pieza más, pero cada una con una sola responsabilidad.

## Decisión

Opción 2, en el repositorio `amachi-bom`, groupId `com.amachi`:

- `amachi-dependencies`: solo versiones (importa JUnit, Spring Boot, Spring Cloud, Spring AI y Cucumber, más lo que
  Spring Boot no gestiona).
- `amachi-parent`: hereda el BOM y agrega Java 25, UTF-8, compilador, builds reproducibles y plugins configurados.
- `amachi-build`: agregador del repositorio, no se instala.

Reglas: una línea del BOM por línea menor de Spring Boot (`2.1.x` = Boot 4.1.x); una versión publicada no se edita;
nunca se fija una librería que Spring Boot ya gestiona.

## Consecuencias

- Toda app hereda `amachi-parent` y declara dependencias sin versión; una app con otro padre importa
  `amachi-dependencies`.
- `com.amachi.core:amachi-bom:2.0.0-SNAPSHOT` (Boot 4.0.0) queda congelada para la Vitalia actual (ADR-0004).
- JUnit 6.1 se importa antes que Spring Boot (que trae 6.0) para alinear la plataforma de pruebas con Cucumber 8;
  el override se retira cuando Spring Boot traiga JUnit 6.1.
- Validación de la 2.1.0: Vitalia compila y pasa 587 tests unitarios; la plataforma compila. Pendiente: Cucumber y
  arranque real con Docker.
- Detalle en `ARQUITECTURA.md` §3.4 y en el `README.md` del repositorio `amachi-bom`.
