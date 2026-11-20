# Decisiones — servicio de turnos y reservas

Registro de decisiones de diseño. Cada decisión nueva lleva su ADR en `docs/adr/` (plantilla: `docs/adr/ADR-template.md`).
Las decisiones heredadas viven como ADR en el repo hermano `../../servicio-catalogo/docs/adr/`.

## Decididas (heredadas del servicio de catálogo)

| Decisión | Resumen | Fuente |
|---|---|---|
| El registro y el login del usuario final viven **en este servicio** | Turnos emite y valida los JWT de usuario; el catálogo nunca los ve | ADR-0035 (catálogo) |
| La app KMP habla **solo con este servicio** | Turnos expone a KMP la búsqueda, las categorías y la disponibilidad, consultando al catálogo. KMP no llama al catálogo ni a la cátedra | ADR-0036 (catálogo), I §17 |
| Turnos llama al catálogo con un **JWT técnico RS256** | `iss=servicio-turnos`, `aud=servicio-catalogo`, `sub=servicio-turnos`, `scope=catalog:read`, vigencia 5 min, `jti`, `userId` opcional e informativo. La clave **privada** solo existe acá; el catálogo guarda la pública | ADR-0037 (catálogo) |
| Contrato catálogo → turnos | Operaciones, modelos, errores y reglas de uso (ignorar horarios y profesionales con `enabled=false`; no cachear el catálogo como vigente) | `../../servicio-catalogo/docs/contratos/catalogo-para-turnos.md` |
| "Disponibilidad" en el filtro del catálogo es **un prefiltro** | `availableOn` = atiende ese día de la semana. La disponibilidad real (slots libres) la construye turnos con las ocupaciones de la cátedra | ADR-0034 (catálogo), E §7 |
| **CORS no se implementa** | Ningún navegador ni app llega a los backends sin pasar por turnos; según el profesor (verbal, a confirmar por escrito) su mención en E §9 es un error del documento | DECISIONES del catálogo |
| Arquitectura hexagonal con ArchUnit | Dominio sin framework; adaptadores en infraestructura; los drivers entran por la fachada `@Service`. La skill gana ante una duda de diseño | skill `hexagonal-catalogo`, `docs/` del catálogo |
| Un issue por milestone, una rama por issue, un PR por rama | Commits Conventional Commits; código, tests y docs en commits separados | `docs/FLUJO-DE-TRABAJO.md` |

## Decididas en este servicio

Aceptadas tras la confirmación del alumno (ADR-0001 y ADR-0002: 2026-11-16; ADR-0003: 2026-11-18, actualizado 2026-11-19; ADR-0004: 2026-11-19; ADR-0005: 2026-11-20).

| Decisión | Resumen | Dónde |
|---|---|---|
| PostgreSQL y Flyway | PostgreSQL propio y exclusivo; migraciones SQL planas; Hibernate en `validate`; se saca H2 del esqueleto | `adr/0001-postgresql-y-flyway.md` |
| Cuatro módulos desde el inicio y reglas entre ellos | `user`, `professional`, `availability`, `reservation` + `config/` técnico; grafo `reservation → availability → professional`, `user` independiente; un módulo solo ve el modelo y los puertos de entrada de otro; reglas de ArchUnit que fijan los pares permitidos y prohíben ciclos; `professional` sin persistencia (cubre la base, no las cachés en memoria: esas se revisan a mano); sin `shared`; carpetas con el primer archivo | `adr/0002-organizacion-en-modulos.md` |
| Servidor de recursos y validación del JWT del usuario | `JwtDecoder` HS512 propio con el mismo `JWT_SECRET`, sin tolerancia de reloj; basta un token válido (no se exige rol todavía); `POST /api/register` y `/api/authenticate` públicos, con el header `Authorization` ignorado ahí; 401 `UNAUTHENTICATED` y 403 `FORBIDDEN` en `problem+json`, sin revelar el motivo del rechazo; sin revocación (limitación documentada) | `adr/0004-servidor-de-recursos-y-validacion-del-jwt.md` |
| Cliente del catálogo con JWT técnico | Token RS256 sin `userId`, único y reutilizado hasta 30 s antes de vencer, con un solo reintento ante `401`; sin otros reintentos (falla rápido) y timeouts de 2 s / 5 s; excepciones para las fallas (`CatalogUnavailableException`, `CatalogIntegrationException`) y `Optional` para «no existe»; clave privada por archivo (`CATALOGO_PRIVATE_KEY_FILE`), nunca versionada; WireMock en los tests | `adr/0005-cliente-del-catalogo-con-jwt-tecnico.md` |
| Registro, login y JWT del usuario | Contrato compatible con JHipster (`POST /api/register`, `POST /api/authenticate` con `username` y `password` → `id_token`); contraseña de 4 a 72 bytes (límite de BCrypt; desvío de JHipster, que permite 100); tabla `app_user` con `id` numérico y `ROLE_USER`; BCrypt; JWT HS512 con `JWT_SECRET` en `.env` (claims `sub`, `auth`, `uid`, `exp` 24 h); Spring Security mínima en T2; errores `problem+json` con `code` (409 para login o email duplicado, `MALFORMED_REQUEST` para JSON roto). Los límites de validación son un supuesto a confirmar | `adr/0003-registro-login-y-jwt-del-usuario.md` |

## Pendientes (decisiones a tomar en este servicio)

| Decisión | Bloquea | Notas |
|---|---|---|
| Cómo informa un puerto de entrada «no reservable» o «catálogo caído» entre módulos | T6, T7 | La regla estricta prohíbe usar las excepciones de otro módulo. Inclinación: resultado como tipo de `domain.model` (ADR-0002). «Catálogo caído» es un fallo de infraestructura: si va como valor del resultado, se traduce a `503` en el borde web. T4 fijó excepciones solo para el puerto de salida hacia el catálogo (ADR-0005); entre módulos sigue abierto |
| Confirmar con la cátedra las validaciones y rutas del usuario | T2 | E §3.2 dice «respetando las validaciones definidas por la cátedra», que no figuran en E ni en I. ADR-0003 usa las habituales de JHipster como supuesto (login 1-50, email 5-254...), salvo la contraseña: 4-72 bytes por el límite de BCrypt en vez de 100. **Gap a confirmar** |
| Qué es el `externalPatientId` | T7 | I §2 pide "un identificador estable del usuario dentro de la aplicación"; propuesta: su ID interno inmutable, nunca el `login` |
| Máquina de estados de la reserva | T7-T9 | E §10 la deja a cargo del alumno; E §7: no retroceder desde un estado final. Estados externos en I §10, §11 y §15 |
| Idempotencia | T8-T9 | Por `eventId` de Kafka y por `reservationProcessId`. Aprendizaje del catálogo (ADR-0028): registrar el id procesado **en la misma transacción** que el efecto |
| Algoritmo de disponibilidad | T6 | Slots a partir de los horarios habilitados (`slotDurationMinutes`) menos las ocupaciones `CONFIRMED` y `HELD` (I §8). Las horas son locales de atención (I §4) |
| Persistencia del progreso y recuperación al reiniciar | T12 | E §8, I §18.3 y §18.4: reconciliar con REST usando los identificadores persistidos |
| Reintentos y backoff en segundo plano | T12 | T4 ya fijó los timeouts (2 s / 5 s) y el único reintento del `401` (ADR-0005). Falta la recuperación de procesos: reintentos limitados y observables (I §18.4). Referencia: ADR-0030 del catálogo (backoff exponencial del listener) |
| Contrato REST hacia KMP | T5-T11 | Endpoints y DTO; se documenta como contrato (E §12) |
| Cómo se filtran las reservas por usuario | T10 | `GET /api/appointments` devuelve las de toda la cuenta técnica: hay que aplicar la propiedad por usuario (I §11) |
| Topics, grupo de consumo y claves de Kafka | T8 | Vienen de la configuración de integración (I §5.1); la key es `reservationProcessId` (I §15) |
| Dockerización del servicio | T1, T14 | E §3: "la infraestructura local de los dos servicios backend, sus bases de datos y las dependencias necesarias deberá poder levantarse mediante Docker Compose". Ambiguo: ¿incluye los servicios o solo sus bases? **Se pregunta a la cátedra el 17/11.** Que el servicio nazca con `Dockerfile` y Compose, sin bloquear el hito del 17/11 |
