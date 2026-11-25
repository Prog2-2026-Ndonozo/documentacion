# ADR-0008: Crear el hold y máquina de estados de la reserva (módulo `reservation`)

Fecha: 2026-11-24
Estado: aceptada

## Contexto
E §7 define el comienzo de una reserva: el usuario elige un profesional, una fecha y un horario válidos; el servicio de turnos
consulta la información necesaria y solicita un hold temporal a la cátedra. I §9 define el hold (`POST /api/appointment-holds`):
devuelve `holdId`, `reservationProcessId`, `status: HELD` y `expiresAt`, y la vigencia real la fija `expiresAt` (una copia local no
la extiende). E §4.2 y E §8 piden validar con datos vigentes del catálogo antes de operar, y ADR-0002 deja que `reservation`
llegue al catálogo solo a través de `availability`. E §10 deja la máquina de estados a cargo del alumno y E §7 prohíbe retroceder
un proceso que ya alcanzó un estado final.

Este es el primer módulo con base de datos propia (el proceso de reserva se persiste, asociado al usuario) y el primero que
llama a la cátedra para una operación con efectos. I §18.4 advierte que un timeout no prueba que la operación falló, y la
cátedra no ofrece una clave de idempotencia para el hold.

## Opciones consideradas

### Qué es el `externalPatientId` (pendiente de `DECISIONES.md`)
1. **El id interno del usuario con prefijo (`user-42`).** + Estable, único, sin datos personales, sin columna nueva; sigue el
   ejemplo de I §9. − Depende de que el id interno no cambie (es una clave primaria inmutable).
2. **Un UUID guardado por usuario.** + Opaco. − Columna y migración nuevas sin ventaja concreta.
3. **El `login`.** − Mutable y cercano a un dato personal.

### Cómo valida `availability` que un turno es reservable
1. **Un puerto de entrada nuevo en `availability` con un resultado propio.** + Valida la agenda vigente sin repetir la
   consulta de ocupaciones; los conflictos los decide la cátedra, que es la fuente de verdad; aplica el criterio «entre módulos,
   resultados» (ADR-0007). − Un puerto y un resultado más.
2. **Reutilizar la disponibilidad y buscar el turno en la lista.** + Sin puertos nuevos. − Consulta las ocupaciones a la cátedra
   dos veces y hace que turnos decida un «ocupado» que no es su dato.
3. **No validar y confiar en `INVALID_SLOT` de la cátedra.** − Contradice E §4.2 y el backlog de T7.

### Máquina de estados local
1. **Definirla completa ahora, guardar el estado como texto con restricción.** + T8 a T12 no vuelven a tocar la tabla; las
   reglas de transición quedan en el dominio y se prueban desde ya. − Estados que T7 todavía no usa.
2. **Solo `HELD` ahora y ampliar en cada milestone.** + Menos diseño inicial. − Una migración y una revisión de reglas por
   milestone.

### Persistencia y propiedad por usuario
1. **`user_id` con clave foránea a `app_user`.** + Integridad real y simple. − La tabla de `reservation` conoce la de `user`
   (acoplamiento de esquema, no de código: ArchUnit no lo ve).
2. **`user_id` sin clave foránea.** + Cada módulo es dueño de sus tablas. − La integridad queda a cargo de la aplicación.

### Orden del hold y fallas
1. **Primero la cátedra, después guardar; sin reintentos automáticos del POST.** + Simple; no se crea un segundo hold por
   reintento. − Si guardar falla, queda un hold huérfano hasta que venza.
2. **Guardar primero un proceso «pendiente» y después llamar a la cátedra.** + Deja una pista para reconciliar. − La cátedra no
   permite reconocer el propio hold sin su id; agrega un estado y una transición que hoy no se pueden usar.

### Contrato hacia la app
1. **`POST /api/reservations`, sin exponer el `holdId`.** + La ruta es de este servicio; el identificador interno de la cátedra
   no sale, y los pasos siguientes buscan el hold propio del usuario (propiedad por usuario, E §9).
2. **Reflejar `/api/appointment-holds` y devolver el `holdId`.** + Más literal. − Filtra un identificador interno y confunde los
   dos sistemas.

## Decisión
- **`externalPatientId = "user-" + id del usuario`** (opción 1). Una constante de formato en el dominio; nunca el `login`.
- **Puerto de entrada `CheckSlotReservableUseCase` en `availability`** (opción 1), con el resultado `SlotCheckResult`
  (`Reservable`, `InvalidSlot`, `ProfessionalDisabled`, `ProfessionalNotFound`, `CatalogUnavailable`, `CatalogFailed`). El turno es
  reservable si el profesional está habilitado y los horarios habilitados generan un turno que **empieza exactamente a esa hora**,
  que no empezó todavía. Usa el mismo calculador que la disponibilidad (`SlotCalculator`) con una lista de ocupaciones vacía:
  los conflictos de ocupación los decide la cátedra en el hold.
- **Máquina de estados completa** (opción 1), tabla más abajo. El estado se guarda como texto con una restricción `CHECK`. Las
  reglas de transición viven en el dominio (`ReservationStatus`) y T7 solo usa `HELD`.
- **Tabla `reservation_process`** (migración `V2`), `user_id` con clave foránea a `app_user` (opción 1). El identificador del
  usuario llega a `reservation` como parámetro desde su capa web, tomado del claim `uid` del JWT; `reservation` no importa nada de
  `user`. La clave pública del proceso es el `reservationProcessId` de la cátedra (único).
- **Primero la cátedra, después guardar** (opción 1). El POST del hold nunca se reintenta solo. Un timeout o una falla de
  conexión responde `503`; que el usuario repita es decisión suya. La llamada HTTP no ocurre dentro de una transacción de base.
- **`POST /api/reservations`** (opción 1). Cuerpo `{ professionalId, date, startTime }`; respuesta `201`
  `{ reservationProcessId, status, expiresAt }`. El `holdId` queda guardado y no sale. Protegida con el JWT del usuario.

### Flujo de crear la reserva
1. La capa web toma el `uid` del JWT y llama a la fachada.
2. El caso de uso consulta a `availability` (`CheckSlotReservableUseCase`). Si no es reservable, termina sin llamar a la cátedra.
3. Pide el hold a la cátedra con el `externalPatientId` del usuario. Hora enviada como `HH:mm` (como en el ejemplo de I §9).
4. Guarda el proceso en estado `HELD` con `userId`, `holdId`, `reservationProcessId`, turno, `externalPatientId` y `expiresAt`.
5. Devuelve el proceso; la capa web responde `201`.

### Máquina de estados local
| Estado | Significado | Puede pasar a |
|---|---|---|
| `HELD` | Hold creado en la cátedra (T7) | `WAITING_FOR_PHONE`, `EXPIRED`, `FAILED` |
| `WAITING_FOR_PHONE` | Confirmado inicialmente; la cátedra pidió el teléfono (T8) | `PHONE_SUBMITTED`, `EXPIRED`, `FAILED` |
| `PHONE_SUBMITTED` | Teléfono enviado; se espera el resultado (T9) | `CONFIRMED`, `WAITING_FOR_PHONE` (rechazo reintentable hasta `expiresAt`), `EXPIRED`, `FAILED` |
| `CONFIRMED` | Reserva confirmada | `CANCELLED` |
| `CANCELLED` | Cancelada (T11) | — (final) |
| `EXPIRED` | Hold o proceso vencido | — (final) |
| `FAILED` | Proceso inválido o fallido | — (final) |

Son finales `CANCELLED`, `EXPIRED` y `FAILED`: nada sale de ellos. De `CONFIRMED` solo se puede cancelar, así que un evento tardío
de vencimiento o de invalidez no la altera (E §7, I §18.3). Un estado no avanza a sí mismo ni a uno anterior salvo la excepción
de `PHONE_SUBMITTED` a `WAITING_FOR_PHONE`. Estados de la cátedra → locales: `PHONE_PENDING` y `WAITING_FOR_PHONE` →
`WAITING_FOR_PHONE`; `FAILED` y `AppointmentProcessInvalid` → `FAILED`.

### Errores
| Caso | HTTP | `code` |
|---|---|---|
| Cuerpo ausente, campo faltante o con formato inválido | 400 | `VALIDATION_ERROR` / `MALFORMED_REQUEST` |
| Turno fuera de la agenda habilitada, no alineado o que ya empezó | 400 | `INVALID_SLOT` |
| Profesional deshabilitado | 400 | `PROFESSIONAL_DISABLED` |
| Profesional inexistente | 404 | `PROFESSIONAL_NOT_FOUND` |
| Hold activo para el mismo turno | 409 | `SLOT_ALREADY_HELD` |
| Turno confirmado por otra reserva | 409 | `SLOT_ALREADY_RESERVED` |
| Catálogo no disponible / error de integración | 503 / 502 | `CATALOG_UNAVAILABLE` / `CATALOG_ERROR` |
| Cátedra no disponible / error de integración (incluye `401`, `403` y `VALIDATION_ERROR` de la cátedra) | 503 / 502 | `CATEDRA_UNAVAILABLE` / `CATEDRA_ERROR` |

Los `code` de negocio son los de I §13.1. Los mensajes hacia la app son genéricos: sin URL, token ni detalles internos. Si la cátedra
rechaza un turno que el catálogo dio por reservable (`INVALID_SLOT`, `PROFESSIONAL_DISABLED`, `PROFESSIONAL_NOT_FOUND`), se informa
con el mismo código: la respuesta más reciente es la de la cátedra.

## Justificación
Guardar la máquina completa ahora evita migraciones por milestone y fija en un solo lugar la regla de E §7 (no retroceder desde un
final), probada desde T7. Validar contra el catálogo por un puerto de `availability` cumple E §4.2 sin duplicar la consulta de
ocupaciones. Ocultar el `holdId` y guardar el `user_id` junto al proceso es lo que permitirá aplicar la propiedad por usuario
(E §9): la cátedra solo conoce la cuenta técnica. No reintentar el POST evita duplicar holds ante una respuesta perdida, que es el
riesgo que I §18.4 señala.

## Consecuencias
- Nueva migración `V2__reservation_process.sql`; el módulo `reservation` nace con los tres paquetes y su propia fachada.
- Nuevo puerto de entrada de `availability` (`CheckSlotReservableUseCase`); `reservation` lo usa y no ve nada más de ese módulo.
- `reservation` tiene su propio cliente de la cátedra (duplicar antes que compartir, ADR-0002) que reutiliza el `RestClient` de
  `config/` con el JWT técnico (ADR-0007).
- **Limitación:** si la cátedra crea el hold y el guardado local falla (o se pierde la respuesta), queda un hold que expira solo y
  bloquea ese turno hasta `expiresAt`. Se registra con el `holdId` cuando se conoce. T12 revisa la reconciliación.
- **Limitación:** si el usuario repite el pedido del mismo turno mientras su hold sigue vivo, la cátedra responde
  `SLOT_ALREADY_HELD` aunque el hold sea suyo. Reconocer el hold propio queda fuera de T7.
- **Limitación:** entre la validación y el hold otro usuario puede tomar el turno; lo resuelve la cátedra con `SLOT_ALREADY_HELD`
  o `SLOT_ALREADY_RESERVED`.
- **Confirmado el 2026-11-24 con la cátedra real:** acepta `startTime` como `HH:mm` (el formato de I §9).
- Probar el hold a mano crea un hold real en la cátedra que bloquea el turno hasta su vencimiento.
- El contrato hacia la app se publica en Swagger; el documento de contratos de E §12 se completa en T14.

## Pendiente
Ninguno. El contrato se probó a mano en Swagger con el catálogo y la cátedra reales el 2026-11-24 (14 casos).
