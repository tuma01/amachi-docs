# ADR-0014: Entrar, registrarse y recuperar la contraseña en la pantalla de la aplicación, con Keycloak por dentro

- **Estado**: Aceptada (sustituye la parte de «entrar por la página de Keycloak» del ADR-0006 y del ADR-0003; el ADR-0013 sigue: el segundo factor es opcional salvo por rol)
- **Fecha**: 2026-11-19
- **Decide**: Juan Amachi
- **Controles ISO 27001**: A.5.15–5.18 (control de acceso y gestión de identidades), A.8.5 (autenticación segura), A.8.6 (capacidad), A.8.11 (protección de datos de prueba)
- **Resume**: `docs/ACCESO-INVENTARIO.md` y `docs/PLATAFORMA.md` (HAL3) de `amachi-platform`

## Contexto

Con el ADR-0006, entrar era una navegación a la página de Keycloak (correo y después contraseña, en dos pantallas) y el registro de
pacientes se había quitado. Medido con la pila real (HAL3): un paciente nuevo recorría 9 pasos y 2 correos, y la página de Keycloak pesaba
unos 2,4 MB. Los usuarios (Bolivia y Latinoamérica, internet lento, adultos mayores, muchos sin correo) no aceptan ese recorrido, y la
demo congelada (`v1.0.0-demo`) ya tenía la pantalla que sí funciona: roles, hospital, correo y contraseña en **una** pantalla.

## Decisión

1. **La pantalla de la demo vuelve** (`/login`, `/signup`, `/forgot-password`, y `/platform/login`), con su diseño, y Keycloak sigue
   siendo la identidad. El navegador envía el correo y la contraseña al BFF (`POST /bff/login`); el BFF las valida con Keycloak
   (concesión de contraseña del cliente `amachi-bff`, de servidor a servidor) y abre la sesión: el navegador solo recibe la cookie
   (`HttpOnly; Secure; SameSite=Strict`). La contraseña no se guarda ni se registra en ningún sitio.
2. **Keycloak conserva** la política de contraseña, el bloqueo progresivo, el segundo factor, la pertenencia a organizaciones, el correo
   y la recuperación. Nada de eso se reescribe.
3. **Lo que no cabe en la pantalla pasa por la página de Keycloak, una sola vez por cuenta**: el primer acceso de una cuenta nueva (su primera
   contraseña, confirmar el correo, configurar el segundo factor). El BFF lo detecta (`409 setup_required`) y la pantalla manda a Keycloak
   con `login_hint` (el correo ya escrito). Después, la entrada diaria es la pantalla de la aplicación.
4. **Registrarse son dos pasos**: el formulario de la demo y «Revise su correo» (`POST /public/registration`). La cuenta nace con el
   correo **sin confirmar**; hasta abrir el enlace no entra. Cada sector decide si lo ofrece y con qué rol
   (`app.registration.enabled` y `app.registration.role`; apagado por defecto: la plataforma es B2B). Vitalia: rol `PATIENT`.
5. **El código de seguridad, para el administrador**, es una pantalla propia («Código de seguridad») con el campo y el botón de la
   demo; el rol Administrador lo exige (`second-factor-roles: SUPER_ADMIN,ADMIN` en Vitalia). Quien lo activó por su cuenta tiene el enlace
   «Tengo un código de seguridad» en el mensaje de error.
6. **«Olvidé mi contraseña»** (`POST /public/password-reset`): responde siempre igual; si hay cuenta, Keycloak envía el enlace donde se
   elige la nueva contraseña.

## Lo que se midió con Keycloak 26.4 (y cambió el diseño)

| Hallazgo | Consecuencia |
|---|---|
| Contraseña mala, código malo, código ausente y cuenta bloqueada dan **la misma respuesta** | La pantalla no puede saber si falta el código: por eso el código es una pantalla por rol y un enlace en el error; no hay un detector de cuentas |
| Pedir una organización que **existe pero no es de la persona** no da error: el token sale **sin** la organización | El BFF comprueba que el token nombra la organización pedida; si no, rechaza **antes** de crear sesión alguna (un defecto real que la suite encontró) |
| Una cuenta con el **correo sin confirmar entra** por contraseña directa (solo el flujo del navegador lo pide) | El BFF exige `email_verified`; si no, `409 setup_required`. Sin esto, la confirmación del correo del registro se saltaba |
| El flujo de contraseña directa de Keycloak pide el código **solo a quien ya lo tiene configurado**: un administrador (rol `mfa-required`) sin código configurado **entraba solo con la contraseña** | Flujo propio `amachi-direct-grant` ligado al cliente del BFF (en los cuatro realms): exige el código por rol y, si falta configurarlo, no da token |
| Un administrador sin código configurado recibe de Keycloak `HTTP 500 unknown_error` (después de validar la contraseña) | El BFF lo trata como `setup_required`: va a Keycloak una vez a configurarlo |
| La contraseña directa **no trae `auth_time`** | La API falla cerrado (`RecentAuthentication`): el primer acto sensible pide reautenticar por Keycloak. Es lo seguro y se mantiene (decidido el 2026-11-20, opción A; quitar ese paso exigiría que la API confíe en la hora de inicio anotada por el BFF: ADR aparte) |
| El parámetro `lang` de `execute-actions-email` **no cambia el idioma del correo** (medido: llegaba en inglés) | El correo sale en el idioma guardado en la cuenta (atributo `locale`): la plataforma lo guarda antes de enviar |
| Una cuenta con acciones pendientes (contraseña temporal) responde `Account is not fully set up` | Mismo camino: primer acceso por Keycloak |

## Reglas de seguridad de lo nuevo

- **BFF** (`/bff/login`): CSRF activo (un inicio de sesión forzado metería a la víctima en la cuenta del atacante); sesión nueva antes de guardar
  nada (fijación de sesión, ASVS V3.2); límite por IP **y** por cuenta y minuto, persistente en la base del BFF (se mantiene entre
  instancias y reinicios; el correo se guarda como hash); `RFC 9700` desaconseja esta concesión para clientes públicos: aquí es un cliente
  **confidencial** que habla con Keycloak de servidor a servidor, y el límite de intentos y el bloqueo de Keycloak siguen vigentes.
- **Registro**: nunca se une a datos que ya existen. Un correo que ya es de una cuenta, de un miembro o de una persona que la organización
  registró se rechaza (`409 registration.email_taken`): nadie reclama la ficha de otra persona escribiendo su correo (esas personas entran por
  invitación o recuperan la contraseña). Un registro repetido sobre una cuenta que **nunca confirmó** la renueva (la contraseña más nueva gana
  y se reenvía el enlace): no bloquea al verdadero dueño del correo. Un correo recibe como mucho un mensaje por minuto desde cualquiera de las dos
  puertas (nadie llena el buzón de otra persona); la IP está limitada por el límite general.
- **Recuperar contraseña**: la respuesta es la misma exista o no la cuenta; solo se envía a quien tiene cuenta en Keycloak (un miembro que nunca tuvo acceso no recibe nada).
- Una contraseña que la política rechaza revierte el alta completa (y retira la cuenta de Keycloak si la acababa de crear).
- Auditoría: `USER_SELF_REGISTERED`, `PASSWORD_RESET_REQUESTED`.

## Consecuencias

- Riesgos **aceptados**: revelar «este correo ya tiene cuenta» en el registro (el mismo tipo que HAL1; limitado por IP y la persona necesita saberlo
  para entrar o recuperar); un atacante que registra el correo ajeno y la víctima confirma sin querer queda con una cuenta **vacía** (no se une a
  ningún dato), y la víctima lo arregla con «olvidé mi contraseña».
- La traducción al aimara de los textos nuevos **no está hecha** (caen al español); requiere una persona que hable el idioma.
- El flujo `amachi-direct-grant` y el binding del cliente son parte del realm: un realm ya creado no los relee (aplicar con la consola o recrearlo).
- **No probado**: la velocidad con conexión lenta; dos instancias del BFF a la vez (el límite persistente está diseñado para ello, no ejercitado);
  la carga con muchos usuarios; la entrega real de correo del registro (solo MailDev).
