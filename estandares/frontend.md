# Estándar de frontend (Angular)

> Parte de la arquitectura del ecosistema (`../ARQUITECTURA.md`). Conserva su numeración (§8).

## 8. Frontend

### 8.1 Estructura (`src/app/`)
- `api/` (generado) · `core/` (servicios, guards, interceptores, auth, i18n) · `platform/` (SuperAdmin) · `tenant/`
  (roles de la organización) · `public/` (páginas sin sesión) · `shared/` (componentes, directivas, utils, estilos).
- Todo componente reutilizable en `shared/components/` (exportado en su `index.ts`); utilidades puras en
  `shared/utils/` con su `.spec.ts`.

### 8.2 Motor CRUD (lista / alta / edición)
- 5 archivos en una carpeta en plural: `{x}.routes.ts`, `{x}-crud.config.ts`, `{x}-list|add|edit.component.ts`.
- Piezas: `CrudTemplateComponent`, `CrudBaseComponent`, `CrudBaseAddEditComponent`, `OpenApiCrudAdapter`,
  `getOperationColumn`.
- Formularios declarativos en el config (`required`, `min/max`, `pattern`, `hint`, `icon`, `unit`, `optionsLoader`,
  `emptyText`, `remote`); los campos anidados (`person.firstName`) se reconstruyen solos al guardar.
- **Guardar nunca se desactiva por un formulario inválido**: al pulsarlo se marca cada campo con qué falta. Solo se
  desactiva mientras se envía o cuando la acción no aplica (y entonces se explica por qué).
- **Errores del servidor en su campo** (`markServerFieldErrors`) y **un error se muestra una sola vez**
  (`ownErrorMessage()` cuando la pantalla muestra el suyo).
- **Desactivar, no solo avisar**: una pantalla no ofrece una acción que el servidor rechazará (espejo de la regla).
- Unidad **dentro** del campo (`unit`), nunca en la etiqueta.
- Un select vacío dice por qué (`emptyText`).
- `<form>` sin `[formGroup]`: `(ngSubmit)` no se emite y el navegador **recarga la página** → `(submit)` con
  `preventDefault()` o un `FormGroup`.
- Datos asíncronos que lee una plantilla "trasplantada" (`preGridTemplate`) → `signal` (si no, NG0100).
- Fechas: todo `matDatepicker` con la directiva de máscara del idioma.

### 8.3 Datos grandes — siempre en el servidor

| Caso | Template |
|---|---|
| Lista | `OpenApiCrudAdapter.getPaginated` + `table.serverSide` + `searchField: 'query'` (+ `baseSearchDto` para un filtro fijo, `sortBy`) |
| Autocompletado de catálogo | `serverAutocomplete()`: debounce 400 ms, mínimo 2 letras, la consulta nueva cancela la anterior |
| Elegir un registro en un formulario | `RemoteAutocompleteComponent` o campo `type: 'remote'` con `pagedRemoteSource(...)` |
| Cascada (país → región → ciudad) | `parentCascade(...)`: pide solo los hijos del valor elegido |
| Panel / KPIs | Un endpoint de resumen con conteos en el servidor (número fijo de consultas) |
| Varias secciones en una pantalla | `PagedSection<T>`: cada sección es su propia página del servidor |
| Calendario | Solo el rango visible, con tope y aviso si se supera |

- Una entidad que crece: su adaptador declara **solo** `getPaginated`; sin `table.serverSide`, falla explícitamente en
  vez de descargar todo.
- Catálogos cerrados y chicos (< 50 por naturaleza) pueden filtrarse en el navegador.

### 8.4 i18n
- Un JSON por idioma, **siempre sincronizados** (validador `npm run validate:i18n` en CI). Ninguna clave sin traducción
  en todos los idiomas; ningún texto visible sin clave (tooltips incluidos).
- Raíz `menu` solo para la navegación (el sidebar agrega el prefijo `menu.`; en los `*-menu.json` el `name` nunca lo
  lleva). Singular/plural con dos claves (`x`, `x_one`).
- Menús por rol en `src/assets/menus/*-menu.json` (`link` | `sub` | `group`; entrada de edición con
  `"visible": false`; rol aditivo con `"requiredRole"`).

### 8.5 UI y navegación
- Contenedor raíz sin padding propio; **3 tipos de encabezado** (lista, detalle, panel) de `_page-layout.scss`; todas
  las páginas empiezan en la misma posición con el mismo ancho.
- Estilos globales en `src/styles/custom/` (formularios, diálogos `.vt-dialog`, avisos en línea `.vt-inline-notice`
  con niveles, celdas `.tc-*`, paleta de colores con nombre). Prohibido `:global()` y estilos ad-hoc por componente.
- Responsivo con breakpoints únicos (móvil ≤ 600, tablet ≤ 1024); ninguna página con desplazamiento horizontal.
- Feedback completo: cursor y hover en lo clicable, spinner mientras carga, estado vacío, tooltips traducidos.
- **Navegación por rol**: cada rol tiene su área (`/admin`, `/<rol>`…); un componente compartido se monta en una ruta
  espejo con `backRoute`; nunca navegar al área de otro rol.
- **Sitio público por país o mercado** (cuando aplica): el país/mercado en la URL (`/reservar/bo`, `/mercados/us`),
  un país por defecto configurado por el SuperAdmin, sin listados masivos hasta que el visitante busca.
