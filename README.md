# Amachi Docs

Arquitectura, estándares y decisiones comunes a **todas** las aplicaciones del ecosistema Amachi (Amachi Platform y
las apps de cada sector: Vitalia, InverPulse…). Describe *cómo se construye* cualquier funcionalidad con nivel de
aplicación profesional en producción.

## Tres niveles de documentación

| Nivel | Dónde | Qué contiene |
|---|---|---|
| Ecosistema | este repositorio | Arquitectura general, estándares de código, seguridad y cumplimiento, decisiones (ADR), plantillas |
| Plataforma | `amachi-platform/docs/PLATAFORMA.md` | Archivos modelo de cada caso, modernización con su estado, plan de etapas |
| Aplicación | `docs/` dentro del repositorio de cada app (ej. `vitalia-api/docs/`) | Dominio, reglas del sector, regulación propia, roadmap |

Un documento de app describe **solo su dominio**; todo lo demás se rige por este repositorio. Si una app necesita
apartarse de una regla, lo dice explícitamente en su documento y por qué.

## Orden de lectura

1. `ARQUITECTURA.md` — principios, stack, estructura, multi-tenancy, seguridad, tiempo y estados, escalabilidad.
2. El estándar del área en la que se va a trabajar (`estandares/`).
3. `plantillas/checklist-feature.md` antes de empezar un feature.
4. La ADR relacionada si se va a tocar una decisión tomada (`adr/`).

## Contenido

| Archivo | Secciones |
|---|---|
| `ARQUITECTURA.md` | §0 Uso y mantenimiento · §1 Principios · §2 Stack · §3 Estructura (3.1 módulos, 3.2 feature, 3.4 BOM) · §4 Multi-tenancy · §6 Seguridad · §7 Tiempo, estados y procesos · §9 Escalabilidad |
| `estandares/base-de-datos.md` | §3.3 Base de datos |
| `estandares/backend.md` | §5 Backend por capas (entidad, DTO, mapper, repository, specification, service, último administrador, errores, API, contrato) |
| `estandares/frontend.md` | §8 Frontend (estructura, motor CRUD, datos grandes, i18n, UI) |
| `estandares/pruebas.md` | §10 Pruebas y calidad |
| `plantillas/checklist-feature.md` | §11 Checklist de un feature nuevo |
| `seguridad/cumplimiento.md` | §14 Objetivo de cumplimiento (ISO 27001, OWASP ASVS L2, Ley 25) · §14.1 Decisiones |
| `amachi-platform/docs/PLATAFORMA.md` | §12 Caso → archivo modelo · §13 Modernización · §14.2 Plan de etapas |
| `CHANGELOG.md` | §15 Registro de cambios |
| `adr/` | Decisiones de arquitectura, una por archivo |
| `plantillas/` | ADR, checklist de feature, plan de pruebas manuales |

Las secciones conservan su número original: una referencia «§N» en cualquier documento se busca en esta tabla.

## Cómo se mantiene

- **Una regla vive en un solo archivo.** Ningún tema se trata en dos archivos (la causa de los 21 documentos
  contradictorios de Vitalia fue el solapamiento, no la cantidad de archivos).
- **Mismo día**: todo cambio de regla, modelo o convención se escribe aquí en la misma sesión en que se decide, con
  una fila en `CHANGELOG.md`.
- **Decisiones con contexto**: una decisión de arquitectura nueva o que reemplaza otra se registra como ADR
  (`plantillas/adr.md`). Una ADR aceptada no se edita: se escribe otra que la reemplaza.
- **Código y documento no coinciden**: se verifica cuál está bien, se corrige el que esté mal y se anota en el
  `CHANGELOG.md`.
- Cambios por pull request; los textos en español, los identificadores de código como están en el código.
