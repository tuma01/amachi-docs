# ADR-0005: Vitalia evoluciona en su mismo repositorio; la demo queda en una rama de versión

- **Estado**: Aceptada
- **Fecha**: 2026-10-26
- **Decide**: Juan Amachi
- **Reemplaza**: ADR-0004

## Contexto

- El ADR-0004 decidió congelar la Vitalia actual, renombrar su repositorio `vitalia-legacy`, archivarlo y crear una
  `vitalia-api` nueva desde el template de la plataforma.
- Al reorganizar el ecosistema (2026-10-26) el repositorio del backend ya se llama `vitalia-api` (antes
  `amachi-platform`) y conserva toda la historia de Vitalia: commits, pull requests (#1 a #87), ramas `feature/AMA-NNN`
  y el código de referencia de los patrones que la plataforma replica (`PLATAFORMA.md`). El frontend `vitalia-web`
  guarda además toda la documentación del producto en `docs/`.
- Un repositorio nuevo partiría esa historia en dos y obligaría a buscar en el archivado el origen de cada decisión.
- Git permite congelar una versión sin otro repositorio: una rama de versión con su etiqueta, abierta en paralelo con
  un worktree.

## Opciones consideradas

1. **Repositorio nuevo desde el template y el actual archivado (ADR-0004)**: empieza limpio; parte la historia, duplica
   la configuración de GitHub y un arreglo de la demo no se puede llevar a la versión nueva con un merge.
2. **Mismo repositorio: la demo en `release/1.0` y la evolución en `main`**: conserva la historia y los enlaces entre
   commits, pull requests y documentos; un arreglo de la demo se lleva a `main` con un merge o un cherry-pick. En
   contra: `main` se aleja mucho de `release/1.0` (borra su copia de `core-*`) y la rama de la demo hay que protegerla.

## Decisión

Opción 2, en los dos repositorios de Vitalia (`vitalia-api` y `vitalia-web`):

- **Demo congelada**: rama `release/1.0` con la etiqueta `v1.0.0-demo` (mismo commit que `main` el 2026-10-26),
  abierta en los worktrees `F:\JAVA-PROJET\vitalia\vitalia-api-demo` y `vitalia-web-demo`. Sigue con
  `com.amachi.core:amachi-bom:2.0.0-SNAPSHOT`, solo recibe arreglos que bloqueen una demo y solo usa datos ficticios.
  Un arreglo va en una rama `hotfix/1.0.x-descripcion` desde `release/1.0` y se etiqueta `v1.0.x-demo`.
- **Evolución**: `main` se migra a la plataforma cuando esta esté lista (al cerrar su etapa de seguridad): depende de
  ella como librería versionada (ADR-0002), hereda `com.amachi:amachi-parent` (ADR-0001), borra su copia de los módulos
  `core-*` y conserva solo los módulos médicos (`vitalia-medical-core`, `vitalia-medical-catalog`,
  `vitalia-clinical-ai`), el arranque y las pruebas.
- No se crea `vitalia-legacy` ni se archiva ningún repositorio.

## Consecuencias

- La historia de Vitalia queda en un solo lugar; la demo sigue funcionando sin trabajo nuevo.
- Se mantiene lo dicho en el ADR-0004 sobre la demo: después del 2026-12-31 usa Spring Boot 4.0 sin soporte gratuito y
  conserva los hallazgos de seguridad abiertos. Aceptable para una demo con datos de prueba, **no** para pacientes
  reales.
- Se mantiene la validación de la plataforma con un módulo médico de Vitalia al cerrar la etapa de seguridad.
- Hay que proteger `release/1.0` en GitHub (solo por pull request) y avisar en el `CLAUDE.md` de los worktrees de la
  demo que está congelada.
- Mientras `main` y la plataforma tengan los dos su copia de `core-*` con el mismo groupId y versión, no se ejecuta
  `mvn install` en la plataforma. El choque desaparece cuando la plataforma usa el groupId `com.amachi.platform`
  (etapa D de `PLATAFORMA.md` §14.2).
