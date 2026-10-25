# ADR-0004: La Vitalia actual queda congelada y se reconstruye sobre la plataforma

- **Estado**: Aceptada
- **Fecha**: 2026-10-25
- **Decide**: Juan Amachi

## Contexto

- Vitalia (gestión hospitalaria) está en etapa de demo y contiene su propia copia de los módulos `core-*`.
- La plataforma todavía va a cambiar mucho (etapa 2 en curso, seguridad y modernización pendientes). Migrar Vitalia
  ahora obligaría a adaptarla dos veces.
- Spring Boot 4.0, la línea que usa Vitalia, pierde el soporte gratuito el 2026-12-31.

## Opciones consideradas

1. **Migrar la Vitalia actual a cada versión de la plataforma**: Vitalia siempre al día; trabajo repetido y riesgo de
   romper la demo.
2. **Congelarla y reconstruirla cuando la plataforma esté terminada**: un solo esfuerzo de migración, sobre una base
   estable.

## Decisión

Opción 2:

- La Vitalia actual se queda en `com.amachi.core:amachi-bom:2.0.0-SNAPSHOT` (etiqueta `v2.0.0-SNAPSHOT`), solo con
  arreglos que bloqueen una demo y solo con datos ficticios.
- Su repositorio se renombra `vitalia-legacy` y se archiva cuando la nueva funcione.
- La nueva `vitalia-api` se crea desde el template, depende de la plataforma (ADR-0002) y solo migra los módulos
  médicos (`vitalia-medical-core`, `vitalia-medical-catalog`, `vitalia-clinical-ai`).

## Consecuencias

- La demo sigue funcionando sin trabajo nuevo.
- Después del 2026-12-31 la Vitalia actual usa una versión de Spring Boot sin soporte gratuito y conserva los
  hallazgos de seguridad abiertos: aceptable para una demo con datos de prueba, **no** para pacientes reales.
- La plataforma se valida con un módulo médico de Vitalia al cerrar la etapa de seguridad, sin esperar al final.
