# ADR-0004: Servidor de recursos y validación del JWT del usuario

Fecha: 2026-11-19
Estado: aceptada

## Contexto
E §9 exige «exigir y validar el JWT en las APIs protegidas», «proteger los endpoints que no sean expresamente públicos»,
«validar firma, vigencia y permisos del token recibido» y «evitar confiar en identificadores de usuario enviados libremente
por el cliente». El ADR-0003 resolvió la **emisión** del token (HS512 con `JWT_SECRET`, claims `sub`, `auth`, `uid`, `iat`,
`exp`) y dejó la **validación** para T3. Hoy, con solo la cadena mínima de T2, cualquier ruta protegida responde 403 aunque
el token sea válido.

Este servicio es el único que emite y valida este token. El JWT técnico RS256 que le manda al catálogo (ADR-0037 del
catálogo) es otro token, con otra clave y otro destinatario: no debe aceptarse acá.

## Opciones consideradas

### Cómo se valida
1. **Un bean `JwtDecoder` propio** (Nimbus, HS512, el mismo `JWT_SECRET`) y el servidor de recursos de Spring Security.
   + Poco código, fija el algoritmo y reutiliza el mecanismo estándar. − Emisor y decodificador leen el secreto por separado.
2. **Configurar el decodificador solo con propiedades de Spring Boot.** Boot ofrece propiedades para clave pública, JWK Set
   o `issuer-uri`, pero ninguna para una clave simétrica (HMAC): no es viable para HS512.
3. **Un filtro propio que parsee el JWT.** + Control total. − Reinventa lo que Spring ya prueba y mantiene.

### Qué autorización exigen las rutas protegidas
1. **Exigir además el rol `ROLE_USER`** (leyendo el claim `auth`). + Un token válido sin ese rol daría 403 y queda preparado
   el rol administrador opcional de E §9. − Más configuración sin un uso real hoy.
2. **Solo exigir un token válido.** + Más simple; hoy todo usuario registrado tiene el mismo rol. − El 403 no tiene ningún
   camino que lo dispare hasta que existan reglas de propiedad (T13) o un segundo rol.

### Tolerancia de reloj en la vigencia
1. **Ninguna.** + Quien emite y quien valida son el mismo servicio y el mismo reloj; un token vencido se rechaza en el
   segundo exacto.
2. **La de Spring por defecto (60 s).** + Cero código. − Acepta tokens vencidos hasta un minuto sin motivo.

### Endpoint protegido de ejemplo
1. **No agregar ninguno.** + No amplía el alcance del backlog; los tests usan un controller de prueba.
2. **Un `GET /api/account` al estilo JHipster.** + Permite una prueba manual más vistosa. − Endpoint no pedido; habría que
   decidir qué datos expone.

## Decisión
- **Decodificador HS512 propio** (opción 1) con el mismo `JWT_SECRET` que usa el emisor. Fijar el algoritmo rechaza un token
  con otro (HS256, `none`) y el JWT técnico RS256. Se agrega la dependencia `spring-boot-starter-oauth2-resource-server` (que
  incluye `jose` y reemplaza a `spring-security-oauth2-jose` en el `pom.xml`).
- **Solo se exige un token válido** (opción 2 de autorización). No se lee el claim `auth` para autorizar todavía.
- **Sin tolerancia de reloj** (opción 1).
- **Sin endpoint de ejemplo** (opción 1). Para probar a mano: un token válido en una ruta inexistente da 404 (la seguridad lo
  dejó pasar) y sin token da 401.
- **Rutas:** son públicas `POST /api/register` y `POST /api/authenticate`; todo lo demás exige token. En las rutas públicas el
  header `Authorization` se **ignora**: sin esta regla, Spring valida el token aunque la ruta sea pública y un token vencido
  que la app siga enviando impediría registrarse o iniciar sesión (se comprobó: respondía 401). La excepción es solo para
  esos dos métodos y rutas.
- **Errores** (`application/problem+json` con `code` estable, los mismos nombres que el catálogo): `401` `UNAUTHENTICATED`
  (token ausente, inválido, vencido o de otro esquema) y `403` `FORBIDDEN`. El `401` **no dice por qué falló**: el cuerpo es
  siempre el mismo y la cabecera `WWW-Authenticate` es solo `Bearer`. La cabecera que arma Spring por defecto agrega el
  motivo («Jwt expired at …», «Malformed token»); se reemplaza para no revelarlo.
- **Dónde vive:** el decodificador, la cadena de seguridad y los manejadores de 401 y 403 están en `config/`, porque aplican a
  todos los módulos. Los rechazos de Spring Security ocurren en un filtro, antes de los controllers, así que el
  `@RestControllerAdvice` no los ve: por eso los manejadores se escriben aparte.
- El permiso para el error interno de Spring (`DispatcherType.ERROR`) se deja pasar para que un error real no se enmascare
  como 401.

## Justificación
El decodificador estándar de Spring con el algoritmo fijado cubre lo que pide E §9 (firma y vigencia) con muy poco código, y
la única alternativa viable por propiedades no existe para claves simétricas. La tolerancia cero y el 401 uniforme siguen la
misma idea: este servicio controla ambos extremos del token, así que no hay motivo para ser permisivo ni para explicar de
más. No exigir un rol evita configuración sin uso; el 403 queda con su manejador listo para cuando haya reglas que lo
disparen (propiedad de reservas en T13, o un rol administrador).

## Consecuencias
- Desde acá un controller obtiene la identidad **del token**, nunca del cuerpo ni de la URL: `@AuthenticationPrincipal Jwt`
  y el claim `uid`. T7 (el `externalPatientId`) y T10 (filtrar reservas por usuario) se apoyan en eso.
- El emisor (`JwtTokenIssuer`) y el decodificador leen `JWT_SECRET` por separado. Si alguna vez divergen, los tokens que emite
  el login dejarían de ser válidos: lo detecta el test de punta a punta que usa el token real del login contra la seguridad.
- **Limitación (E §9):** el token vale 24 horas **sin refresco ni revocación**. Si un usuario se desactiva o cambia su clave,
  su token anterior sigue siendo válido hasta que venza. Consultar la base en cada pedido lo evitaría, pero deja de ser sin
  estado; no se hace en T3.
- El `403` no tiene hoy ningún camino que lo dispare en las rutas reales; está probado solo a nivel del manejador.
- Los tests de la capa web que importan `SecurityConfig` ahora necesitan el manejador y la propiedad
  `app.security.jwt.secret`. El test que afirmaba 403 para una ruta no declarada pasa a afirmar 401.
- Sin rate limiting ni bloqueo por intentos fallidos (ya registrado en el ADR-0003).
