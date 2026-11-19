# ADR-0003: Registro, login y JWT del usuario final (módulo `user`)

Fecha: 2026-11-18
Estado: aceptada (los límites de validación son un supuesto a confirmar con la cátedra, ver «Pendiente»)

## Contexto
E §3.2 pide que cada usuario final se registre e inicie sesión contra este backend, que el login devuelva un JWT, y que el
modelo y el contrato sean **compatibles con el usuario generado por JHipster**. E §3.2 y E §9 fijan:
- El registro contempla `login`, `password`, `firstName`, `lastName`, `email`, `imageUrl` (opcional) y `langKey`,
  «respetando las validaciones definidas por la cátedra».
- Los identificadores, el estado de activación, las autoridades y la auditoría los administra el backend; el cliente no los
  elige.
- El usuario queda activo sin verificar el correo. La contraseña se guarda con un mecanismo criptográfico adecuado, nunca en
  texto plano, y no se expone en respuestas ni en logs.
- No se confía en identificadores de usuario enviados por el cliente.

**Lo que ni E ni I dicen:** las validaciones de la cátedra, las rutas y la forma del token. Todo lo que sigue sobre JHipster
(rutas, límites, claims) se toma de cómo se genera ese proyecto y **es un supuesto**, no una cita de E o I.

## Opciones consideradas

### Contrato (rutas y validaciones)
1. **Compatible con JHipster:** `POST /api/register`, `POST /api/authenticate` con respuesta `{ "id_token": "..." }` y las
   validaciones habituales de JHipster.
   + Cumple «mismos datos y comportamientos observables» (E §3.2); quien use JHipster obtiene el mismo contrato.
   − Depende de un supuesto; si la cátedra define otros límites, se ajustan.
2. **Contrato propio** (`/api/users`, `/api/auth/login`).
   + Más libre. − Se aparta de lo que E §3.2 pide como compatibilidad.

### Hash de contraseña
1. **BCrypt** (`spring-security-crypto`). + Es el de JHipster; sin dependencias extra. − Ninguna relevante.
2. **Argon2.** + Más moderno. − Exige otra dependencia (BouncyCastle) sin necesidad para este alcance.

### Firma del JWT del usuario
1. **HS512, secreto compartido** en una variable de entorno.
   + Solo este servicio emite y valida el token: no hace falta clave pública. Es lo que usa JHipster.
   − Quien tenga el secreto puede fabricar tokens: hay que tratarlo como secreto (`.env`).
2. **RS256 con otro par de claves.** + Permite validar sin poder emitir. − Complejidad sin beneficio: nadie más valida este
   token. (El RS256 ya se usa en el token técnico, que sí valida otro servicio: ADR-0037 del catálogo.)

### Cuándo entra Spring Security
1. **En T2, con una cadena mínima.** + BCrypt y el encoder del JWT ya lo necesitan; un único lugar donde declarar qué es
   público. − T2 toca algo de seguridad que T3 completa.
2. **Todo en T3.** − T2 no podría emitir el JWT ni hashear con las herramientas del framework.

## Decisión
- **Contrato compatible con JHipster** (opción 1), con las siguientes validaciones, **supuestas**:

  | Campo | Regla |
  |---|---|
  | `login` | obligatorio, 1 a 50 caracteres, patrón de JHipster (`^[a-zA-Z0-9!$&*_=^`{\|}~.-]+@?[a-zA-Z0-9-]+(?:\.[a-zA-Z0-9-]+)*$\|^[_.@A-Za-z0-9-]+$`); se guarda en minúsculas |
  | `password` | obligatoria, 4 a 72 **bytes** en UTF-8 (ver la actualización del 2026-11-19) |
  | `firstName`, `lastName` | opcionales, hasta 50 caracteres |
  | `email` | obligatorio, formato válido, 5 a 254 caracteres; se guarda en minúsculas |
  | `imageUrl` | opcional, hasta 256 caracteres |
  | `langKey` | obligatoria, 2 a 10 caracteres |

  `login` y `email` son únicos (sin distinguir mayúsculas).
- **Rutas:** `POST /api/register` (201, sin cuerpo útil) y `POST /api/authenticate` (200, `{ "id_token": "<jwt>" }`). Ambas
  públicas. El cuerpo del login lleva `username` y `password`, como el `LoginVM` de JHipster (a diferencia del registro,
  que usa `login`); un `rememberMe` que mande el cliente se ignora.
- **Modelo:** tabla `app_user` (`user` es palabra reservada de PostgreSQL) con `id` numérico autogenerado (`bigint`),
  `login`, `password_hash` (60 caracteres), `first_name`, `last_name`, `email`, `image_url`, `activated` (siempre `true` al
  registrar), `lang_key`, `created_date` y `last_modified_date`. Autoridades en `user_authority(user_id, authority)`; al
  registrar se asigna `ROLE_USER`. El `id` es el identificador estable e inmutable del usuario; será la base del
  `externalPatientId` (I §2, decisión de T7).
- **Hash:** BCrypt. Puerto de salida `PasswordHasher`; adaptador en `infrastructure`.
- **JWT:** HS512 con el secreto en la variable `JWT_SECRET` (Base64 de al menos 64 bytes; `.env`, nunca en el repo). Claims:
  `sub` = login, `auth` = autoridades, `uid` = id del usuario, `iat` y `exp` (24 horas). `uid` no es de JHipster: se agrega
  para identificar al usuario sin depender de datos que mande el cliente (E §9). La emisión es de T2; la **validación en los
  endpoints protegidos** es de T3.
- **Spring Security en T2:** cadena mínima, sin estado, con CSRF desactivado (la autenticación va en un header, no en
  cookie), `register` y `authenticate` públicos y el resto autenticado. En el perfil `dev` también se permiten
  `/v3/api-docs/**` y `/swagger-ui/**`. T3 la completa con el servidor de recursos.
- **Errores** (`application/problem+json`, con un `code` estable): `400` datos inválidos (`VALIDATION_ERROR`, con la lista
  `errors` de `{field, message}`, nunca el valor recibido), `400` cuerpo que no es un JSON válido (`MALFORMED_REQUEST`), `409`
  login o email ya usados (`LOGIN_ALREADY_USED`, `EMAIL_ALREADY_USED`), `401` credenciales inválidas (`BAD_CREDENTIALS`). El login
  responde igual si el usuario no existe o la clave es incorrecta, para no revelar cuál falló. JHipster usa `400` para el
  duplicado; se elige `409` por ser el código que describe el conflicto, y se registra como desvío.

## Justificación
El contrato compatible con JHipster es lo que pide E §3.2 y es la forma de no inventar un contrato propio donde E no define
uno. BCrypt y HS512 son los mecanismos estándar de ese contrato, sin dependencias ni claves adicionales, y alcanzan porque
este servicio es el único que emite y valida el token. Declarar en T2 qué es público evita que Spring Security bloquee las dos
rutas apenas entra al classpath.

## Consecuencias
- Se agregan al `pom.xml` Spring Security y `spring-security-oauth2-jose` (emisión del JWT con Nimbus). Se agrega
  `JWT_SECRET` al `.env` y al `.env.example`.
- Primera migración propia: `V1__usuarios.sql` (`app_user` y `user_authority`).
- Primer módulo real (`user`): desde acá las reglas de ArchUnit controlan código de verdad.
- El `id` numérico es secuencial; si más adelante se expone en URLs, hay que revisar la enumeración. Hoy solo viaja dentro del
  token y se usa como identificador interno.
- Seguridad: sin rate limiting ni bloqueo por intentos fallidos. Queda como limitación a documentar (E §9).
- Si la cátedra publica otras validaciones, se cambian las constantes de validación y este ADR se actualiza; el resto no se
  toca.
- Los pedidos (`RegisterRequest`, `AuthenticateRequest`) redefinen `toString` para no imprimir la contraseña si alguien los
  loguea. El comando de dominio `RegisterUserCommand` también la lleva y conviene hacer lo mismo.

## Actualización (2026-11-19): lo que cambió al implementar
1. **Contraseña de 4 a 72 bytes, no de 4 a 100 caracteres.** BCrypt solo usa los primeros 72 bytes. Las versiones viejas de
   Spring Security truncaban en silencio; la actual rechaza la clave con
   `IllegalArgumentException: password cannot be more than 72 bytes`, que con el límite original habría sido un `500`. Se
   comprobó con una prueba. Es un **desvío de JHipster** (que permite 100) y se registra como tal. `@Size` cuenta
   caracteres y una «ñ» ocupa 2 bytes, por eso el máximo se valida con una restricción propia (`@MaxUtf8Bytes(72)`), también
   en el login para que una clave más larga devuelva `400` y no un error interno.
   - Descartadas: pre-hashear con SHA-256 antes de BCrypt (más código y más superficie para permitir 28 caracteres más) y
     cambiar a Argon2 (otra dependencia, contra lo decidido arriba).
2. **El login usa `username`** (como JHipster) y no `login`.
3. **Se agrega el código `MALFORMED_REQUEST`** para que un JSON roto responda en `problem+json` igual que el resto de los
   errores, y no con el formato por defecto de Spring.

## Pendiente
- **Confirmar con la cátedra** los límites de validación y las rutas de registro y login (gap ya registrado en
  `DECISIONES.md`). Si la cátedra exige hasta 100 caracteres de contraseña, habría que reabrir el punto 1 de la
  actualización.
