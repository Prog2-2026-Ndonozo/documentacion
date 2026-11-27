# ADR-0011: Enviar el teléfono y procesar el resultado del proceso de reserva (T9)

Fecha: 2026-11-27
Estado: aceptada

## Contexto
Con T8 el proceso queda en `WAITING_FOR_PHONE` y guarda el `eventId` del pedido (`phone_request_event_id`). T9 cierra el flujo
de E §7: turnos envía `AdditionalInformationSubmitted` (I §15.5) y procesa el resultado final que la cátedra publica en el
topic de acciones: `AppointmentConfirmed` (I §15.6), `AdditionalInformationRejected` (I §15.7), `AppointmentProcessExpired`
(I §15.8) y `AppointmentProcessInvalid` (I §15.9).

Las fuentes definen los eventos pero no cómo le llega el teléfono al backend de turnos: la app KMP habla únicamente con este
servicio, así que ese camino es decisión nuestra. La cátedra normaliza el teléfono (quita espacios, guiones, paréntesis y
puntos) y exige `^\+?[0-9]{7,15}$`. Responde en alrededor de un segundo, por lo que el resultado puede llegar antes de que
turnos termine de guardar su propio estado.

Hoy el listener descarta los eventos de otros tipos y no se vuelven a recibir (ADR-0010). Si se probara el envío contra la
cátedra real sin procesar los resultados, la cátedra confirmaría una reserva real y turnos perdería el `AppointmentConfirmed`.

## Opciones consideradas

### Orden de guardar y publicar
1. **Guardar `PHONE_SUBMITTED`, publicar con confirmación del broker y, si falla, volver a `WAITING_FOR_PHONE`.** + El resultado
   de la cátedra nunca llega antes del estado local; la vuelta atrás ya existe en la máquina de estados (el rechazo también la
   usa). − Un corte entre el commit y la publicación deja el proceso en `PHONE_SUBMITTED` sin que se haya enviado nada.
2. **Publicar y después guardar.** + No hay estado sin publicar. − El resultado puede llegar antes del guardado y chocar por la
   versión; un fallo al guardar deja un teléfono enviado que turnos no registró.
3. **Outbox transaccional** (tabla de eventos pendientes y un proceso que los reenvía). + Sin ventana de pérdida. − Suma una
   tabla, un reenviador y sus pruebas: fuera de alcance para el plazo.

### Entrada del teléfono
Un endpoint propio `POST /api/reservations/{reservationProcessId}/phone` es lo único coherente con que la app solo hable con
turnos; no hay otra alternativa que cumpla E §3.

### Etapas
Una sola etapa obliga a un cambio muy grande. Dos etapas (9a: enviar; 9b: resultados) mantienen commits atómicos, con el
cuidado de probar a mano **una sola vez**, al terminar 9b.

## Decisión
- **Endpoint:** `POST /api/reservations/{reservationProcessId}/phone`, cuerpo `{"phoneNumber": "..."}`, respuesta `202` con
  `{reservationProcessId, status, expiresAt}` (sin `holdId`). Un proceso de otro usuario responde `404 RESERVATION_NOT_FOUND`.
- **Teléfono:** el caso de uso quita espacios, guiones, paréntesis y puntos, valida `^\+?[0-9]{7,15}$` y envía el valor ya
  normalizado. Un valor inválido responde `400 INVALID_PHONE_NUMBER` (código propio, como `INVALID_SLOT`); un cuerpo sin
  `phoneNumber` sigue siendo `400 VALIDATION_ERROR`.
- **Estados:** solo `WAITING_FOR_PHONE` puede enviar. Cualquier otro estado responde `409 INVALID_RESERVATION_STATE`. Si el
  `expiresAt` ya pasó, responde `409 HOLD_EXPIRED` y el proceso pasa a `EXPIRED`, como en 8a.
- **Pedido de teléfono todavía no recibido:** la confirmación REST puede dejar el proceso en `WAITING_FOR_PHONE` antes de que llegue
  el evento de Kafka, y entonces no hay `requestEventId` para enviar. Se responde `409 INVALID_RESERVATION_STATE` con el mensaje
  «la cátedra todavía no pidió el teléfono» y el usuario repite en unos segundos.
- **Orden (opción 1):** guardar `PHONE_SUBMITTED`; publicar con espera de la confirmación del broker (`send(...).get(timeout)`);
  si la publicación falla, volver a `WAITING_FOR_PHONE` y responder el error de integración. La publicación ocurre **después** del
  commit, nunca dentro de la transacción. El corte entre el commit y la publicación se reconcilia en T12.
- **Mensaje enviado:** `eventId` nuevo, `eventType` y `schemaVersion` fijos, `groupId`, `reservationProcessId`, `holdId`,
  `requestEventId` (el `phone_request_event_id` guardado), `information.phoneNumber` y `occurredAt`. La key del mensaje es el
  `reservationProcessId`.
- **Configuración nueva**, ambas de `integration` (I §5.1): `CATEDRA_GROUP_ID` (`groupId`, va dentro del evento) y
  `CATEDRA_KAFKA_APPOINTMENT_PHONE_TOPIC` (`kafkaAppointmentPhoneTopic`). Solo los nombres van al repo (`.env.example`,
  `application.yaml`, compose). El productor reutiliza `CATEDRA_KAFKA`.
- **Resultados** (9b), aplicados en una `@Transactional` y con el `processed_event` de ADR-0010 para los duplicados:

| Evento | Proceso en | Efecto |
|---|---|---|
| `AppointmentConfirmed` | `PHONE_SUBMITTED` | Pasa a `CONFIRMED` y guarda `reservation_id` |
| `AdditionalInformationRejected` | `PHONE_SUBMITTED` | Vuelve a `WAITING_FOR_PHONE`, guarda el motivo y actualiza `expiresAt` con el del evento |
| `AppointmentProcessExpired` | `HELD`, `WAITING_FOR_PHONE`, `PHONE_SUBMITTED` | Pasa a `EXPIRED` |
| `AppointmentProcessInvalid` | `HELD`, `WAITING_FOR_PHONE`, `PHONE_SUBMITTED` | Pasa a `FAILED` |
| Cualquiera | Estado final, `CONFIRMED` o de otro `holdId` | Sin cambios (registrado como procesado; `holdId` distinto o proceso desconocido, `WARN`) |

  El proceso confirmado solo admite cancelación (ADR-0008). Un resultado repetido o tardío no retrocede ningún estado final.
- **Etapas:** 9a (endpoint, productor, `PHONE_SUBMITTED`; sin migración) y 9b (resultados; migración `V4` con `reservation_id`
  y el motivo del último rechazo). La prueba manual contra la cátedra real se hace una sola vez, con 9a y 9b terminados.

## Justificación
Guardar antes de publicar elimina la carrera con el resultado de la cátedra sin pagar un outbox, y la vuelta a
`WAITING_FOR_PHONE` reutiliza una transición ya permitida. Normalizar el teléfono igual que la cátedra evita un rechazo evitable
y un viaje por Kafka. Guardar `reservation_id` deja listo T11 (cancelar necesita ese identificador).

## Consecuencias
- Nueva configuración Kafka de productor (serializadores de `String`, key `reservationProcessId`) y dos propiedades más.
- **Limitación:** si el servicio se corta entre el commit y la publicación, el proceso queda en `PHONE_SUBMITTED` sin haber
  enviado el teléfono; T12 lo reconcilia por REST (I §18.4).
- **Limitación:** hasta terminar 9b, los resultados que publique la cátedra se descartan; por eso no se prueba 9a solo.
- **Limitación:** no se compara `requestEventId` de los resultados con el guardado; se valida el `holdId`, como en ADR-0010.
- Enviar el teléfono por segunda vez tras un rechazo genera un `eventId` nuevo (I §15.7).

## Verificación manual (2026-11-27)
Probado a mano con la cátedra real, con 9a y 9b terminados: confirmar el hold, enviar el teléfono y recibir la confirmación,
que lleva el proceso a `CONFIRMED` con su `reservation_id`. El rechazo, la expiración y el proceso inválido no se provocaron
contra la cátedra: los cubren los tests con Kafka embebido y PostgreSQL.
