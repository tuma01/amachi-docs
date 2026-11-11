# ADR-0013: Segundo factor opcional para todos, con indicador visible y activación en un clic

- **Estado**: Aceptada (sustituye la parte de «OTP obligatorio para administradores» de AMA-010 y del ADR-0006)
- **Fecha**: 2026-11-10
- **Decide**: Juan Amachi
- **Controles ISO 27001**: A.8.5 (autenticación segura), A.8.2 (derechos de acceso privilegiado)
- **Resume**: `keycloak/README.md` y `docs/PLATAFORMA.md` (SEG3) de `amachi-platform`

## Contexto

Desde AMA-010, el rol `mfa-required` hacía que `SUPER_ADMIN` y `ADMIN` entraran siempre con un código OTP de una aplicación de
autenticación. Probado con el navegador real, el efecto en la práctica fue otro:

- el flujo `amachi-browser` tenía **dos** disparadores de OTP seguidos (`Conditional 2FA`, para quien tiene OTP configurado, y
  `MFA required`, para quien tiene el rol); un administrador veía el formulario del código **dos veces**, y Keycloak rechaza reutilizar
  un código en la misma ventana de 30 s, así que parecía un bucle;
- la primera entrada obliga a instalar una aplicación, escanear un QR y escribir un código: para el público de Bolivia (y para quien
  administra un hospital pequeño) es una barrera que aleja de la plataforma, y el 2F obligatorio no lo pide ningún otro servicio al
  usuario común;
- el enlace «Mi cuenta» (donde se gestiona la contraseña y el segundo factor) **no funcionaba**: el realm no daba a nadie los roles del
  cliente `account` (`view-profile`, `manage-account`), y la consola de cuenta respondía 401.

## Decisión

1. **El segundo factor es opcional para todos, también para los administradores.** `app.security.oidc.second-factor-roles` queda **vacío
   por defecto**: nadie recibe `mfa-required`. El mecanismo sigue existiendo: un despliegue o un sector que lo necesite obligatorio
   (p. ej. InverPulse, o quien lee historias clínicas) lista los roles en esa propiedad.
2. **Se invita, no se obliga.** El administrador de plataforma y el del hospital (`ROLE_SUPER_ADMIN`, `ROLE_ADMIN`,
   `ROLE_TENANT_ADMIN`) ven, y nadie más:
   - una **opción en el menú del usuario** (junto a «Mi cuenta» y «Cerrar sesión»), **no en la barra superior**: no es una urgencia de la
     atención al paciente y no debe competir con ella. Dice «Activar verificación en 2 pasos» (clic → se activa) o «Verificación en 2
     pasos: activa» (confirmación);
   - un **aviso al iniciar sesión** mientras no lo tengan activo, con «Activar ahora», «Cómo funciona» (guía de 4 pasos desde instalar la
     aplicación hasta el primer código) y «Recordármelo después» (lo oculta hasta el siguiente inicio de sesión).
3. **Activación en un clic, con la pantalla del proveedor de identidad.** `GET /oauth2/authorization/keycloak?action=enable-second-factor`
   pide a Keycloak la acción iniciada por la aplicación `kc_action=CONFIGURE_TOTP`: tras confirmar quién es, la persona ve la página con el
   QR y el código, y vuelve a la aplicación con el 2F activo. La aplicación nunca maneja el secreto ni el código. Cualquier otro valor de
   `action` se ignora.
4. **El estado lo dice el proveedor de identidad.** El BFF consulta el servicio de cuenta de Keycloak
   (`<issuer>/account/credentials`) con el token de la propia persona y lo expone como `secondFactorEnabled` en `/bff/user`
   (`null` si no pudo saberlo: se trata como «aún no», nunca como «activo»).
5. **El realm (dev, test y prod, idénticos) cambia en dos cosas:**
   - `Conditional 2FA` no se aplica a quien tiene `mfa-required` (condición `negate`), de modo que **un administrador con OTP pide el
     código una sola vez**, sea el 2F opcional o impuesto;
   - se declara el **rol por defecto** (`default-roles-amachi`, con `offline_access`, `uma_authorization` y los roles `view-profile` y
     `manage-account` del cliente `account`): la consola de cuenta funciona para todos y es donde cada persona gestiona su contraseña y su
     segundo factor.

## Riesgo aceptado y compensaciones

Una cuenta de administrador, sobre todo la de plataforma (ve todas las organizaciones), puede quedar solo con contraseña. Se acepta
**de forma consciente** para no cerrar la puerta a quien no usa aplicaciones de autenticación, y se compensa con: contraseña de 12 a 128
caracteres sin las 5 anteriores, bloqueo progresivo tras 5 fallos, bitácora encadenada de la actividad de los administradores, la
opción del menú y el aviso en cada inicio de sesión, y la posibilidad de volverlo obligatorio por rol sin tocar código. **Recomendación para
producción**: empezar con `SUPER_ADMIN` obligatorio cuando la operación de la plataforma tenga más de una persona, y revisar esta
decisión en la primera auditoría de ISO 27001.

## Consecuencias

- Las pruebas de Cucumber de la plataforma fuerzan `second-factor-roles: SUPER_ADMIN,ADMIN` (perfil de pruebas) para seguir
  verificando el mecanismo y el flujo del realm; el valor por defecto se prueba en `IdentityAccountsTest`.
- Un realm ya creado no relee el archivo: en una instancia existente hay que asignar a mano los roles de cuenta (o recrearla).
- **Pendiente** (no bloquea): «recordar este dispositivo» (no existe en Keycloak 26.4; exigiría una extensión propia), tema visual de
  la página de inicio de sesión con la marca del producto, y alinear los realms propios de cada sector (`vitalia-realm-*.json`).
