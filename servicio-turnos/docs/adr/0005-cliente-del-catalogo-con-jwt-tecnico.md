# ADR-0005: Cliente del catálogo con JWT técnico (módulo `professional`)

Fecha: 2026-11-20
Estado: aceptada

## Contexto
E §3 y E §4.2 hacen que el catálogo sea de otro servicio: turnos lo consulta por HTTP y no lo guarda ni lo replica. E §9 exige
JWT validado entre los dos servicios. El ADR-0037 del catálogo ya fijó el token (técnico, RS256, vida de 5 minutos) y el
contrato `catalogo-para-turnos.md` fija las rutas, los modelos y los errores. El ADR-0002 dejó este trabajo en el módulo
`professional`: un único cliente del catálogo, un único emisor del token y una sola traducción de errores.

T4 construye ese lado de salida: el emisor del token, el puerto de salida con su adaptador `RestClient` y la traducción de
errores. Los casos de uso y los endpoints hacia la app son de T5.

**Lo que ya estaba fijado** (contrato del catálogo): claims `iss=servicio-turnos`, `aud=servicio-catalogo`,
`sub=servicio-turnos`, `scope=catalog:read`, `jti`, `iat` y `exp = iat + 5 min`; `userId` opcional e informativo; reutilizar
el token hasta unos 30 segundos antes de que venza; ante un `401`, emitir uno nuevo y reintentar **una sola vez**; no usar
nunca el JWT de la cátedra. Las rutas son `/api/...` sin el prefijo `/api/internal` (que el ADR-0037 dejaba pendiente).

## Opciones consideradas

### ¿El token lleva `userId`?
1. **No incluirlo.** + Un único token compartido y cacheado 5 minutos; los puertos no reciben el usuario. − El catálogo no
   puede ligar una llamada a un usuario (turnos sí, en sus propios registros).
2. **Incluirlo.** + Más trazabilidad en el catálogo. − El token pasa a ser por usuario: no se comparte el caché y cada puerto
   necesita un parámetro `userId`.

### ¿Se reintenta algo además del `401`?
1. **Solo el `401`.** + Ante un catálogo caído o lento falla rápido; quien espera una búsqueda no queda colgado; la
   recuperación en segundo plano es de T12.
2. **Un reintento también ante timeouts y `5xx`.** + Tapa fallos breves. − Suma latencia y esconde un catálogo caído.

### ¿Cómo avisa el puerto de salida cuando algo falla?
1. **Excepciones para las fallas y `Optional` para «no existe».** + Es el mismo criterio que usa el catálogo con la cátedra.
2. **Un tipo resultado cerrado (`Ok`, `NotFound`, `Unavailable`...).** + Más explícito. − Es la decisión de «catálogo caído como
   valor» que sigue pendiente para T6 y T7 y no corresponde cerrar acá.

### ¿Cómo llega la clave privada a la aplicación?
1. **Ruta a un archivo `.pem` por variable de entorno.** + Una clave de varias líneas no viaja dentro del `.env`; en Docker se
   monta de solo lectura; `.gitignore` y `.dockerignore` ya excluyen `*.pem`.
2. **El contenido del PEM en una variable de entorno.** − Incómodo en el `.env` y más expuesto en registros y en
   `docker inspect`.

### ¿Con qué se prueba el adaptador HTTP?
1. **WireMock.** + Un servidor HTTP real: prueba tiempos de espera, conexión rechazada y la secuencia `401` → `200`.
   − Agrega una dependencia de test.
2. **`MockRestServiceServer`.** + Sin dependencia nueva. − No ejercita HTTP real: no puede probar tiempos de espera ni
   conexión rechazada.

## Decisión
- **Sin `userId`.** Un único token firmado con RS256 se reutiliza entre todas las llamadas hasta 30 segundos antes de su
  vencimiento (`CatalogTokenProvider`, seguro entre hilos). Si el catálogo lo rechaza, se descarta solo si sigue siendo el
  vigente, para no tirar uno recién emitido por otro hilo. `userId` es opcional en el contrato: se puede agregar después.
- **Reintento único del `401` y nada más.** Tiempos de espera de **2 s para conectar y 5 s para leer** (configurables). El
  reintento queda registrado con un aviso (`WARN`) sin el token.
- **Traducción de errores** (`application/exception`, en el módulo `professional`):

  | Respuesta del catálogo | En turnos |
  |---|---|
  | `401` | emite un token nuevo y reintenta una vez; si vuelve a dar `401`, `CatalogIntegrationException` |
  | `403`, `400`, otro `4xx`, `404` sin `code`, respuesta ilegible o cuerpo vacío | `CatalogIntegrationException` (algo de la integración está mal de nuestro lado, no del usuario) |
  | `404` con `PROFESSIONAL_NOT_FOUND` (solo en la agenda) | `Optional.empty()` |
  | `5xx`, sin conexión, conexión rechazada o tiempo de espera | `CatalogUnavailableException` |

  Un `404` sin `code` no es «el profesional no existe»: es una ruta inexistente, o sea un error de integración. Esta decisión
  solo fija el puerto de salida; cómo informa un módulo a otro («catálogo caído», «no reservable») sigue pendiente para
  T6 y T7.
- **Puertos de salida** en `professional/domain/ports/out`: `ProfessionalSearchPort`, `ProfessionalCategoryPort` y
  `ProfessionalAgendaPort`. Un `CatalogRestClient` (HTTP, token y traducción de errores) y tres adaptadores finos con un
  `CatalogMapper`, como hace el catálogo con la cátedra.
- **Clave privada por archivo:** `CATALOGO_PRIVATE_KEY_FILE` apunta a un PEM en formato PKCS#8 (el que genera
  `openssl genpkey`) de RSA de al menos 2048 bits. La aplicación **no arranca** si falta, si no se puede leer o si no sirve,
  y el mensaje nunca incluye el contenido. En Docker el Compose monta el archivo de solo lectura.
- **Configuración** (`app.catalog.*`): `CATALOGO_BASE_URL` (por defecto `http://localhost:8080`, el puerto por defecto del catálogo), `CATALOGO_CONNECT_TIMEOUT`,
  `CATALOGO_READ_TIMEOUT`, `CATALOGO_PRIVATE_KEY_FILE`; el emisor y la audiencia son `servicio-turnos` y
  `servicio-catalogo`.
- **WireMock** (`wiremock-standalone` 3.13.1, solo para tests). Se comprobó que funciona con Spring Boot 4.1.1. Las claves de
  los tests se generan en memoria en cada ejecución: **nunca se versiona una clave privada**, ni siquiera de prueba (los
  repositorios rechazan o marcan los archivos con claves privadas).
- **Prueba contra el catálogo real:** como todavía no hay endpoints (llegan en T5), hay un test opcional que solo corre si se
  define `CATALOGO_REAL_URL`; un `./mvnw test` normal lo omite. El catálogo conserva su puerto por defecto (8080): es un servicio terminado y no se le cambia nada. Turnos, que está en desarrollo, usa el **8081** (`SERVER_PORT`; en Docker, `APP_PORT`), así que ninguno pisa al otro. Para esta prueba turnos se ejecuta fuera de Docker. La red compartida entre los dos Compose es de T14.

## Justificación
Un token único y compartido es la forma más simple de cumplir «reutilizar hasta unos 30 segundos antes de que venza», y el
`userId` es opcional y solo informativo, así que su costo (token por usuario y un parámetro en cada puerto) no se justifica.
Fallar rápido ante un catálogo caído coincide con E §8 («no iniciar operaciones si el catálogo no responde») y deja los
reintentos en segundo plano para T12, donde sí corresponden. Separar `CatalogUnavailableException` (no es culpa de nadie en
particular) de `CatalogIntegrationException` (error nuestro) permite que T5 responda distinto en cada caso.

## Consecuencias
- **Dependencias:** `wiremock-standalone` (test). No se agrega ninguna de ejecución: el emisor usa Nimbus, que ya viene con
  `spring-boot-starter-oauth2-resource-server` (ADR-0004).
- Se usa `RestClient.builder()` y no el `RestClient.Builder` de Spring Boot: Boot 4 no lo autoconfigura sin un starter
  adicional, y el test de arranque completo lo detectó. El `RestClient` no se expone como bean para no confundirlo con otros
  clientes HTTP futuros.
- **Para T5:** `CatalogUnavailableException` debería traducirse a `503` y `CatalogIntegrationException` a un error del lado
  del servidor (`502` o `500`) con un `code` estable. Antes de llamar al catálogo hay que validar lo que escribe el usuario
  (por ejemplo, un nombre de más de 100 caracteres): un `400` del catálogo se trata como error de integración.
- **Todos los tests que levantan la aplicación completa** necesitan una clave: usan la propiedad
  `app.catalog.jwt.private-key-file` con un PEM temporal generado en el test.
- **Seguridad:** la clave privada solo existe en turnos y nunca se versiona, ni se copia a la imagen; el token nunca se
  escribe en mensajes ni en registros (hay tests que lo comprueban). Rotar la clave implica entregar la nueva pública al
  catálogo y reiniciarlo. En Docker Desktop el usuario no-root del contenedor puede leer el archivo montado aunque tenga
  permisos `600` (se comprobó); con Docker Engine nativo la propiedad del archivo no se remapea y podría hacer falta ajustar
  permisos.
- **Limitación:** el catálogo no puede ligar una llamada a un usuario concreto; la trazabilidad por usuario queda en los
  registros de turnos. Sin revocación: la vida corta del token (5 minutos) es la mitigación.

## Pendiente
- Entregar la clave **pública** al catálogo (`CATALOGO_JWT_PUBLIC_KEY_LOCATION`) y verificar a mano contra el catálogo real
  (último ítem de T4).
- Cómo se informan «catálogo caído» y «no reservable» entre módulos (T6 y T7).
