# ADR-0002: Amachi Platform se consume como dependencia versionada, no se copia

- **Estado**: Aceptada
- **Fecha**: 2026-10-25
- **Decide**: Juan Amachi

## Contexto

- Amachi Platform nació como copia del backend de Vitalia. Desde entonces los dos tienen los mismos hallazgos de
  seguridad (tokens en claro, JWT HS256, AES-CBC con IV fijo, Actuator público…) y cada corrección hay que hacerla
  dos veces.
- La plataforma será la base de varias apps de sector (InverPulse, una app ganadera, Vitalia reconstruida…). Con la
  copia, el costo de cada corrección crece con cada app.
- Hoy la plataforma y Vitalia publican artefactos con las mismas coordenadas (`com.amachi.app.platform:core-*`
  `1.0.0-SNAPSHOT`), lo que obliga a no instalar la plataforma en `~/.m2`.

## Opciones consideradas

1. **Copiar los módulos `core-*` en cada app (template)**: libertad total por app; pero cada app es un fork y las
   correcciones de seguridad no llegan solas.
2. **Dependencia versionada**: la app declara `core-*` como dependencias con una versión; exige puntos de extensión
   bien diseñados y versionado semántico.

## Decisión

Opción 2:

- La plataforma usa el groupId `com.amachi.platform` y publica su propio BOM (`amachi-platform-dependencies`) con
  una sola versión para todos los `core-*`.
- Versionado semántico: parche corrige, menor agrega sin romper, mayor rompe. Cada app elige cuándo sube.
- La configuración común (seguridad, reloj, errores, Flyway, Jackson) pasa a un starter `core-boot` con
  autoconfiguración de Spring Boot.
- **Una app de sector nunca modifica `core-*`**: si necesita algo que la plataforma no ofrece, se agrega a la
  plataforma como punto de extensión genérico (Organization, roles como datos, `DomainEntityProvider`, enums
  extensibles, catálogo de servicios, Bridge).
- El template (`amachi-app-template`) solo contiene el esqueleto de una app de sector, no el código de `core-*`.
- El frontend sigue el mismo modelo: librería Angular común + app del sector.

## Consecuencias

- Una corrección de la plataforma llega a todas las apps con un cambio de versión.
- Al cambiar el groupId desaparece la colisión con Vitalia y la plataforma se puede instalar y publicar.
- Antes de declarar la plataforma terminada se valida con un caso real (portar un módulo de Vitalia al cerrar la
  etapa de seguridad).
- Publicación: `mvn install` local al principio; GitHub Packages con la CI.
