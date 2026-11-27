# ADR-0010: Consumir eventos de Kafka y concurrencia sobre el proceso de reserva (T8b)

Fecha: 2026-11-25
Estado: aceptada

## Contexto
El ADR-0009 decidió que el pedido de teléfono (`AdditionalInformationRequested`, I §15.4) se consume por Kafka con `spring-kafka`,
que el `eventId` se registra en `processed_event` en la misma transacción que el efecto y que la transición `HELD` →
`WAITING_FOR_PHONE` es idempotente desde el REST y desde el evento. Quedaban sin cerrar tres puntos del tramo 8b.

Primero, la cátedra publica el evento justo después de responder `202` a la confirmación (I §10), así que el evento puede
llegar **antes** de que turnos guarde el resultado del REST. Si el REST guarda después una copia vieja del proceso, borra el
`eventId` que acaba de guardar el evento y T9 lo necesita como `requestEventId`.

Segundo, el servicio no tiene configuración de Kafka y el catálogo ya fijó nombres de propiedades para el mismo
`integration` de la cátedra. Tercero, en la máquina del alumno están `spring-kafka` y `spring-kafka-test` pero no el módulo
Kafka de Testcontainers ni una imagen de Kafka.

## Opciones consideradas

### Carrera entre el REST y el evento
1. **Bloqueo optimista** (columna `version`, `@Version`, campo `version` en `ReservationProcess`). + Solución general: T9 a T12
   también escriben el proceso; el choque se detecta y se resuelve releyendo. − Una columna y un campo más; el caso de uso de 8a
   gana un `catch`.
2. **`UPDATE` condicional por transición** («pasar a X solo si sigue en Y»). + Sin columna nueva. − Hay que reescribir lo de 8a y
   cada transición futura.
3. **No hacer nada.** + Cero cambios. − La ventana es de milisegundos pero el daño (un `eventId` perdido) es silencioso.

### Qué hace el evento según el estado del proceso
Una sola tabla de reglas (ver Decisión); la alternativa de ignorar el evento si el REST no terminó obliga a reintentar y no
converge por sí sola.

### Configuración y pruebas de Kafka
1. **Mismos nombres de propiedad que el catálogo; `@EmbeddedKafka` en las pruebas.** + Sin descargas ni imagen; coherente con el
   otro servicio. − Un broker embebido no es el de la cátedra.
2. **Kafka de Testcontainers.** + Más cercano a un broker real. − Módulo nuevo e imagen que hoy no están disponibles.

## Decisión
- **Bloqueo optimista** (opción 1). `ReservationProcess` suma `Long version`; la entidad usa `@Version`. Un choque de versión sale
  del puerto como `ReservationConflictException` (dominio, `reservation/domain/exception`). En el listener la excepción se deja
  propagar: Spring Kafka reintenta y la segunda vez ve el estado nuevo. En la confirmación REST el caso de uso relee el proceso y,
  si ya está en `WAITING_FOR_PHONE`, responde igual que si hubiera guardado él.
- **Reglas del evento** `AdditionalInformationRequested`, aplicadas en una sola transacción:

| Proceso | Efecto |
|---|---|
| `HELD` | Pasa a `WAITING_FOR_PHONE` y guarda `phone_request_event_id` |
| `WAITING_FOR_PHONE` | Guarda `phone_request_event_id` (el último pedido es el que vale) |
| `PHONE_SUBMITTED`, `CONFIRMED` o final | Sin cambios; el evento se registra como procesado |
| No existe, o el `holdId` del evento no coincide con el guardado | `WARN` y se descarta sin reintentar; se registra como procesado |

  Antes de aplicar, el caso de uso inserta el `eventId` con `INSERT ... ON CONFLICT DO NOTHING`; si no inserta, el evento ya se
  procesó y termina. Todo (insertar, leer, actualizar) ocurre en una `@Transactional` del caso de uso, como el ADR-0023 del catálogo.
- **Configuración**, con los nombres del catálogo: `CATEDRA_KAFKA` (servidor), `CATEDRA_KAFKA_CONSUMER_GROUP` y
  `CATEDRA_KAFKA_APPOINTMENT_ACTIONS_TOPIC` (el `kafkaAppointmentActionsTopic` de I §5.1). Los valores viven en el `.env`; en el
  repo solo van los nombres (`.env.example`, `application.yaml`, compose). `auto-offset-reset: earliest`. Se usa el grupo de consumo
  que asignó la cátedra, el mismo que usa el catálogo: cada servicio escucha un topic distinto.
- **Listener:** recibe el mensaje como `String`, lo parsea y descarta con `WARN` lo ilegible o de otro `eventType`. Reintentos
  como el ADR-0030 del catálogo: 5 reintentos con espera exponencial (1 s inicial, multiplicador 2, tope 10 s) y un `ERROR` al
  agotarlos con `partition`, `offset` y `key`, sin registrar el payload.
- **Pruebas:** `@EmbeddedKafka` para el listener y `spring.kafka.listener.auto-startup=false` en el resto de los tests, para que
  no intenten conectarse a un broker.

## Justificación
El bloqueo optimista es la forma más simple de que dos escritores del mismo proceso no se pisen, y se va a necesitar en T9 a
T12. Una tabla de reglas por estado hace al evento independiente del orden de llegada. Reutilizar los nombres, los reintentos y
la forma del listener del catálogo evita inventar convenciones distintas para el mismo `integration`.

## Consecuencias
- Migración `V3`: `phone_request_event_id`, `version` (`BIGINT NOT NULL DEFAULT 0`) y la tabla `processed_event(event_id PK,
  processed_at)`. `spring-kafka` entra al `pom.xml`.
- 8a cambia un poco: `ReservationProcess` suma `version` y el caso de uso de confirmación maneja el choque.
- **Limitación:** el backlog del topic llega con `earliest`; los eventos de procesos que no están en la base se descartan con `WARN`.
- **Limitación:** los eventos de otros tipos se consumen y se descartan; no se vuelven a recibir cuando T9 los trate (T12 reconcilia).
- **Limitación:** no se compara `occurredAt`; dos pedidos fuera de orden dejan el último procesado.
- **Verificado con la cátedra real (2026-11-26):** el broker acepta el grupo de consumo compartido con el catálogo.
- **Limitación:** el evento se aplica aunque el `expiresAt` local ya haya pasado (un proceso `HELD` vencido pasó a
  `WAITING_FOR_PHONE`). Se confía en que la cátedra emite el pedido solo si el hold seguía vigente; T12 reconcilia las expiraciones.
- La tabla `processed_event` crece con cada evento; la limpieza queda fuera de alcance, como en el catálogo.

## Verificación manual (2026-11-26)
Con la cátedra real: el consumidor se unió al grupo, recibió la partición del topic de acciones y leyó desde `earliest`. Un hold
confirmado por REST (profesional 203) quedó en `WAITING_FOR_PHONE` con el `phone_request_event_id` del evento y `version = 2`
(confirmación REST y evento, cada uno una vez). Un proceso del backlog quedó con `version = 1` (solo el evento). Cada `eventId`
quedó registrado una vez en `processed_event`.
