# ADR-0009: Confirmar el hold y recibir el pedido de teléfono (módulo `reservation`)

Fecha: 2026-11-24
Estado: aceptada

## Contexto
E §7 define el paso siguiente al hold: la aplicación confirma inicialmente el hold por REST y después recibe por Kafka
`AdditionalInformationRequested`. I §10 define la confirmación (`POST /api/appointment-holds/{holdId}/confirm`, con
`externalPatientId`, `patientFirstName` y `patientLastName`; responde `202` con `WAITING_FOR_PHONE`) e I §15.4 el evento, cuyo
`eventId` hay que conservar para devolverlo como `requestEventId` (T9). Kafka entrega al menos una vez (I §15.1, I §18.1) y el
estado observado por REST y por Kafka puede llegar en momentos distintos (E §7).

Es la primera vez que este servicio consume Kafka. La cátedra exige nombre y apellido de 2 a 100 caracteres, mientras que el
registro propio los deja opcionales, con máximo 50 y sin mínimo (ADR-0003). `reservation` no puede depender de `user` (ADR-0002).

## Opciones consideradas

### Origen del nombre y apellido del paciente
1. **Cuerpo del pedido de confirmación, validado de 2 a 100.** + Respeta el grafo de módulos; no depende de un perfil que puede
   estar vacío. − La app los pide en cada reserva.
2. **Perfil del usuario, con un puerto nuevo en `user`.** + Una sola fuente. − `reservation` dependería de `user` (cambia el
   ADR-0002) y el perfil puede no cumplir las reglas de la cátedra.
3. **Claims del JWT.** − Datos personales en el token y posiblemente desactualizados.

### Quién pasa el proceso de `HELD` a `WAITING_FOR_PHONE`
1. **Cualquiera de los dos caminos, de forma idempotente.** + No importa quién llegue primero; no hay reintentos por carrera;
   converge sin retroceder (E §7). − Dos puntos de entrada a la misma transición.
2. **Solo el REST; el evento espera.** + Un solo punto. − Si el evento gana la carrera hay que reintentarlo.

### Idempotencia de los eventos
1. **Tabla genérica `processed_event` con el `eventId` como clave, escrita en la misma transacción que el efecto.** + Sirve para
   todos los eventos (T9); una clave duplicada significa «ya procesado». − Una tabla más.
2. **Solo una columna con el `eventId` en el proceso.** + Simple. − No cubre los eventos de T9.

## Decisión
- **Nombre y apellido en el cuerpo** de la confirmación (opción 1), de 2 a 100 caracteres, sin espacios solo. El `externalPatientId`
  es el de T7 (`user-<id>`).
- **Contrato:** `POST /api/reservations/{reservationProcessId}/confirm`, cuerpo `{ firstName, lastName }`, respuesta `202`
  `{ reservationProcessId, status, expiresAt }`. Protegido con el JWT del usuario. La propiedad por usuario se aplica con el
  `user_id` guardado: si el proceso no es del usuario se responde como si no existiera (`404 RESERVATION_NOT_FOUND`).
- **Transición idempotente desde ambos caminos** (opción 1): el REST y el evento llevan el proceso a `WAITING_FOR_PHONE` si está en
  `HELD`; si ya está ahí no hacen nada. El evento además guarda su `eventId` en `phone_request_event_id`. Un proceso que ya llegó a
  un estado final no se altera (E §7, I §18.3).
- **Idempotencia por `eventId`** (opción 1): tabla `processed_event(event_id PK, processed_at)`; el `eventId` se inserta en la
  misma transacción que el efecto y una clave duplicada se trata como evento ya procesado. Aprendizaje del ADR-0028 del catálogo.
- **Kafka:** dependencia `spring-kafka` y un listener en `reservation/infrastructure/kafka` que convierte el evento en un comando
  de un puerto de entrada de `reservation`; el dominio no conoce Kafka. El listener despacha por `eventType`; en T8 solo trata
  `AdditionalInformationRequested` y registra y descarta el resto. Reintentos con espera exponencial y recoverer, siguiendo la idea
  de los ADR-0026 y ADR-0030 del catálogo. Servidor, grupo de consumo y topic vienen de la configuración, nunca del código.
- **Dos tramos:** 8a, confirmar por REST (endpoint; no necesita migración porque el estado ya existe); 8b, el consumidor de Kafka, la migración `V3` y la tabla `processed_event`.

### Errores de la confirmación
| Caso | HTTP | `code` |
|---|---|---|
| Cuerpo inválido (nombre o apellido fuera de 2 a 100) | 400 | `VALIDATION_ERROR` |
| El proceso no existe o no es del usuario | 404 | `RESERVATION_NOT_FOUND` |
| El proceso ya no se puede confirmar (estado distinto de `HELD`) | 409 | `INVALID_RESERVATION_STATE` |
| Hold vencido o inexistente en la cátedra (`HOLD_EXPIRED`, `HOLD_NOT_FOUND`) | 409 | `HOLD_EXPIRED` (el proceso pasa a `EXPIRED`) |
| `RESERVATION_PROCESS_ALREADY_CONFIRMED` de la cátedra | 202 | el proceso pasa a `WAITING_FOR_PHONE` (reconcilia un timeout) |
| `EXTERNAL_PATIENT_MISMATCH`, `HOLD_OWNER_MISMATCH`, `INVALID_SLOT`, `PROFESSIONAL_NOT_FOUND` de la cátedra | 502 | `CATEDRA_ERROR` |
| `SLOT_ALREADY_RESERVED` de la cátedra | 409 | `SLOT_ALREADY_RESERVED` |
| Cátedra no disponible o timeout (sin reintento automático, I §18.4) | 503 | `CATEDRA_UNAVAILABLE` |

La llamada HTTP no ocurre dentro de una transacción de base. Si la cátedra confirma y guardar falla, el reintento del usuario recibe
`RESERVATION_PROCESS_ALREADY_CONFIRMED` y reconcilia.

## Justificación
Pedir el nombre en el cuerpo mantiene el grafo de módulos y evita depender de un perfil opcional. Hacer idempotente la transición
desde los dos caminos elimina la carrera entre el REST y Kafka sin reintentos. Una tabla genérica de eventos procesados, escrita
con el efecto, es la forma más simple de cumplir «al menos una vez» sin duplicar efectos.

## Consecuencias
- En 8b: nueva migración `V3` (columna `phone_request_event_id` y tabla `processed_event`); `spring-kafka` entra al `pom.xml`.
- **Limitación:** los eventos que el listener no trata en T8 se consumen y se descartan; no se vuelven a recibir cuando T9 los trate.
  En desarrollo no hay datos de producción que perder; T12 reconcilia por REST los procesos que queden sin resultado.
- **Limitación:** el nombre se pide en cada reserva.
- Las propiedades de Kafka (servidor, grupo de consumo, topics) se completan al implementar 8b, con los nombres que dé el alumno.
- El contrato hacia la app se publica en Swagger; el documento de contratos de E §12 se completa en T14.

## Pendiente
- Probar 8a y 8b a mano con la cátedra real (crea un hold real).
