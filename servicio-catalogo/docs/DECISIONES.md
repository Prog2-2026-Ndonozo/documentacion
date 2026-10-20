# Decisiones de diseño

Registro de decisiones (E §10 y §12). Numeración de ADR: `0002` está reservado para la propagación de identidad entre
servicios (issue #10); los ADR del modelo de dominio usan el rango `0010+`. Cada decisión importante tiene su ADR en `adr/NNNN-titulo.md`
(plantilla: `../../../ADR-template.md`). Esta tabla es el índice: se lee primero, el ADR solo si hace falta el detalle.

## Tomadas

| Decisión | Resumen | Dónde |
|---|---|---|
| Arquitectura hexagonal, un solo módulo `catalog` | Tres capas + `config/`; dominio sin framework; reglas verificadas con ArchUnit | `arquitecturaHexagonalServicioCatalogo.md` |
| Motor de base de datos | PostgreSQL propio y exclusivo (E §3) | — |
| Estructura de capas | Fijada en issue #1; los fixtures de `arquitecturafixtures/` prueban que las reglas detectan violaciones | `arquitecturaHexagonalServicioCatalogo.md` |
| Modelo de dominio base: 4 conceptos | `ProfessionalCategory`, `Professional`, `WeeklySchedule`, `CatalogVersion` (referencia por ID: propuesta) | `adr/0010-modelo-de-dominio-base.md` |
| Incluir `createdAt`/`updatedAt` | Se conservan los datos que publica la cátedra | `adr/0011-incluir-created-at-y-updated-at.md` |
| Tipos de fechas y horas | `Instant`, `LocalTime`, `DayOfWeek` | `adr/0012-tipos-de-fechas-y-horas.md` |
| Tipos numéricos, textos y estado | IDs y versión `Long`/`long`; `slotDurationMinutes` `int`; textos `String`; `enabled` `boolean` | `adr/0013-tipos-numericos.md` |
| Validaciones del modelo | Nombres no vacíos; `startTime < endTime`; rango múltiplo de la duración; versión ≥ 0 (0 = inicial) | `adr/0014-validaciones-del-modelo.md` |
| Modelo inmutable que se valida al construirse | Clases Lombok sin setters ni constructor vacío, con builder; **desvío de la regla 2 del skill** | `adr/0015-modelo-inmutable-con-validacion.md` |
| Excepción de validación del modelo en `domain/` | Tipo propio, lanzado por el modelo; **desvío de la regla 1 de excepciones del skill** | `adr/0016-excepcion-de-validacion-del-dominio.md` |
| Flyway para las migraciones | SQL plano en `db/migration`; Hibernate en `ddl-auto: validate` | `adr/0017-flyway-para-migraciones-de-bd.md` |
| Claves foráneas diferidas y reglas de esquema | FK `DEFERRABLE INITIALLY DEFERRED`; versión local en la misma base; ID de la cátedra como PK; índices en las FK | `adr/0018-claves-foraneas-diferidas.md` |

## Pendientes

| Decisión | Bloquea | Notas |
|---|---|---|
| Idempotencia: deduplicar por `eventId` de Kafka (I §15.1, §18.1) | #6 | Falta decidir dónde se persiste el `eventId` procesado |
| Qué significa "disponibilidad" en el filtro de búsqueda | #9 (issue #8, ADR) | E §4.1 no lo define |
| Quién consume la búsqueda: ¿KMP directo + turnos, o solo turnos? | #9 | Si solo turnos, se elimina la fila de KMP de la tabla de puertos de entrada |
| Dónde viven registro/login del usuario final (E §3.2) | — | E no asigna el servicio; `reunion.md` lo ubica en catálogo. No está en el backlog |
| Propagación de identidad entre servicios (JWT usuario vs técnico) | #10 | `ADR-0002` aún no escrito (E §9) |
| Estrategia transaccional de la sincronización (granularidad, reintentos, dato inválido) | #5, #6 | Principio fijado en ADR-0018: la unidad de trabajo es una versión completa en una transacción |
| `SyncDiagnosticsPort`: tabla propia o sobre `CatalogRepositoryPort` | #7, #12 | Depende de la máquina de estados de sync |
| Qué hacer si la cátedra publica un dato que viola una validación del dominio | #5, #6 | Cancelar toda la sync o saltear el registro (ADR-0014) |
| Significado de "nombre" en el filtro de búsqueda; efecto de una categoría deshabilitada sobre sus profesionales | #9 | Huecos de E §4.1 (ADR-0010) |
| ¿La cátedra publica alguna vez la versión 0? | #7 | Confirmar con el profesor (ADR-0014) |
