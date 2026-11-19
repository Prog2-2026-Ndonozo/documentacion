# Backlog — servicio de turnos y reservas

> Notación: `E §x` = `PROJECT_STATEMENT`, `I §x` = `INTEGRATION_REFERENCE` (ver `fuentes.md`).
> Es un plan inicial: se ajusta al avanzar. Los números `T1`... son de planificación; los de GitHub pueden diferir.

## Convención de trabajo
- Un issue de GitHub por milestone = una rama = un PR que lo cierra (`Closes #n`).
- El checklist de cada issue es la guía interna de implementación (capas hexagonales), no son sub-issues.
- Definición de terminado por issue: checklist resuelto; código en la capa correcta (skill y ArchUnit); al menos un test que
  cubra el caso de uso; `./mvnw test` en verde; decisiones de diseño con su ADR; documentación al día; y, desde que exista
  el Compose, `docker compose up --build` sigue levantando el servicio.

## Punto de partida
**Se parte de cero.** El esqueleto de septiembre (`Initial commit`) no es una base de trabajo: trae una clase y un test de
ejemplo y una dependencia de H2, que E §3 no admite como persistencia principal. T1 lo reemplaza. Del esqueleto solo se
aprovecha lo de Maven (parent de Spring Boot 4.1.1, wrapper, plugin de compilación con Lombok). Stack: ver `AGENTS.md`.

## Requisitos transversales
Valen para todos los issues; un issue no se da por terminado si los rompe.
- **PostgreSQL propio y exclusivo.** H2, SQLite y otras bases embebidas o en memoria están prohibidas como persistencia
  principal (E §3). Las pruebas contra base usan Testcontainers.
- **Dockerizado (E §3):** el servicio y su base se levantan con `docker compose up --build`, y el servicio sigue pudiendo
  correr desde el IDE. Se asume la lectura estricta del enunciado y se confirma con la cátedra el 17/11 si alcanza con
  levantar solo las bases (`DECISIONES.md`).
- **Secretos fuera del repositorio y de la imagen:** `.env`, claves privadas y la cuenta técnica de la cátedra.
- **Ningún acceso a la base del catálogo ni réplica de su contenido** (E §3, E §4.2).

## Objetivo para la reunión del 17/11
En la reunión se muestra el **catálogo funcionando**. De este servicio alcanza con haberlo **empezado** (T1: base en
PostgreSQL, módulos y reglas de ArchUnit). T2 a T5 (login con JWT, endpoints protegidos y una llamada real al catálogo con el
JWT técnico) ya no tienen fecha propia: se hacen antes de la entrega, priorizando hacerlo bien.

---

## Milestone 1 — Base del servicio y usuarios

### T1 chore: base del servicio `[chore]`
- [x] Sacar el esqueleto (`Persona`, `HelloTest`) y **H2**; agregar PostgreSQL, Flyway, Lombok, springdoc, ArchUnit, Testcontainers
- [x] `docker-compose.yml` con la base propia de este servicio y `.env` / `.env.example` (el `.env` está gitignoreado)
- [x] `Dockerfile` de dos etapas (JDK 25 para compilar, JRE 25 para ejecutar) y el servicio en el Compose, sin secretos en la imagen (requisito transversal; E §3). No bloquea el hito del 17/11
- [x] Estructura hexagonal y reglas de ArchUnit (partir de `ArchitectureRules` del catálogo): adaptar `MODULO_SOLO_TIENE_TRES_CAPAS`, descartar las dos reglas de la cátedra, sumar la regla entre módulos, la de los pares permitidos del grafo, la de ausencia de ciclos (`slices()` sin `config/`) y la de `professional` sin persistencia, cada una con su fixture que viola y otro que respeta (ADR-0002)
- [x] Sin carpetas de módulos vacías: nacen con su primer archivo (`user` en T2, `professional` en T4)
- [x] ADR-0001 (PostgreSQL y Flyway) y ADR-0002 (cuatro módulos y reglas entre ellos): aceptados

### T2 feat: registro y login del usuario final `[feature]` (E §3.2, E §5.1, E §9)
- [x] Modelo de usuario compatible con JHipster: `login`, `password`, `firstName`, `lastName`, `email`, `imageUrl`, `langKey`
- [x] Registro: queda activo sin verificar el correo; identificadores, estado y autoridades los administra el backend
- [x] Contraseña con hash adecuado (nunca en texto plano, ni en respuestas ni en logs)
- [x] Login que devuelve un JWT; ADR del contrato (rutas y validaciones: ver gap en `DECISIONES.md`)

### T3 feat: seguridad con JWT de usuario `[security]` (E §9)
- [ ] Servidor de recursos: validar firma y vigencia del JWT del usuario en los endpoints protegidos
- [ ] 401 y 403 como `application/problem+json` con `code` estable
- [ ] Tests con el decodificador real (token válido, sin token, firma inválida, vencido)

## Milestone 2 — Integración con el catálogo y disponibilidad

> Módulos de este milestone (ADR-0002): T4 y T5 son el módulo `professional` (sin persistencia); T6 es `availability` y
> depende del puerto de entrada de la agenda de `professional`.

### T4 feat: cliente del catálogo con JWT técnico `[feature]` (ADR-0037 del catálogo)
- [ ] Par de claves RS256 y emisor del token (`CatalogTokenIssuer`, ver el contrato del catálogo): reutilizar 5 min; ante `401`, emitir y reintentar una vez
- [ ] Puerto de salida y adaptador `RestClient` con timeouts; traducir `401`, `403`, `404` y `5xx` a errores propios
- [ ] Tests con un catálogo simulado (WireMock o similar) y contra el catálogo real a mano

### T5 feat: búsqueda y categorías para KMP `[feature]` (E §4.1, ADR-0036 del catálogo)
- [ ] `search` y `categories`: reenvían al catálogo y se protegen con el JWT del usuario
- [ ] Contrato REST hacia KMP documentado y probado en Swagger

### T6 feat: disponibilidad `[feature]` (E §7, I §8, I §17)
- [ ] Agenda de un profesional (catálogo) + ocupaciones de la cátedra → slots libres para un rango de fechas
- [ ] Usar solo horarios y profesionales con `enabled=true`; no cachear el catálogo como vigente (E §3, E §4.2)
- [ ] Tests del cálculo de slots (bordes: sin horarios, slot parcial, ocupado y retenido)

## Milestone 3 — Reserva (REST + Kafka)

### T7 feat: crear el hold y persistir el proceso `[feature]` (I §9)
- [ ] Antes del hold, validar con un puerto de entrada de `availability` que el horario sea reservable con datos vigentes del catálogo (E §4.2, E §8; ADR-0002). Decidir cómo informa «no reservable» y «catálogo caído» (pendiente en `DECISIONES.md`)
- [ ] Crear el hold con `externalPatientId` estable; persistir `holdId`, `reservationProcessId` y `expiresAt`, asociados al usuario
- [ ] Traducir los errores de I §13.1 (`SLOT_ALREADY_HELD`, `INVALID_SLOT`, `PROFESSIONAL_DISABLED`...)
- [ ] ADR: máquina de estados de la reserva

### T8 feat: confirmar el hold y recibir el pedido de teléfono `[feature]` (I §10, I §15.4)
- [ ] Confirmar por REST (`WAITING_FOR_PHONE`); consumir `AdditionalInformationRequested` y conservar su `eventId`
- [ ] Idempotencia por `eventId`, registrado en la misma transacción que el efecto

### T9 feat: enviar el teléfono y procesar el resultado `[feature]` (I §15.5 a §15.9, E §7)
- [ ] Publicar `AdditionalInformationSubmitted`; procesar confirmación, rechazo (reintentable), expiración e inválido
- [ ] No retroceder un proceso en estado final; duplicados sin efecto

## Milestone 4 — Consulta, cancelación y robustez

### T10 feat: consultar las reservas propias `[feature]` (E §5.13, I §11)
- [ ] Listar solo las del usuario (la API de la cátedra devuelve las de toda la cuenta técnica)

### T11 feat: cancelar una reserva propia confirmada `[feature]` (E §5.14, I §12, I §15.10)
- [ ] Cancelación idempotente; rechazo si la reserva es de otro usuario

### T12 feat: robustez y recuperación `[feature]` (E §8, I §18)
- [ ] Vencimiento de holds y procesos; recuperar procesos pendientes tras un reinicio reconciliando por REST
- [ ] Reintentos limitados y observables; no iniciar operaciones si el catálogo no responde

## Milestone 5 — Seguridad y cierre

### T13 test/security: propiedad por usuario y accesos `[security]` (E §9, E §11)
- [ ] Un usuario no consulta ni cancela reservas de otro; rechazo de accesos no autenticados o no autorizados

### T14 docs: documentación obligatoria `[docs]` (E §12)
- [ ] README reproducible; diagrama de arquitectura (app Android, los dos servicios, sus bases e integraciones con la cátedra)
- [ ] Modelo de datos y propiedad; estrategia de sincronización con el catálogo; idempotencia; decisiones de seguridad
- [ ] Contrato hacia KMP; checklist de evidencias mínimas (E §11)
- [ ] `docker compose` de los dos servicios: cada repositorio con su Compose y una red externa compartida (`proyecto-net`) para llamarse por nombre
