# ADR-0007: Disponibilidad de turnos (módulo `availability`)

Fecha: 2026-11-23
Estado: aceptada

## Contexto
E §7 pide que turnos construya la disponibilidad combinando las reglas semanales vigentes del catálogo, las ocupaciones de la
cátedra y el profesional y las fechas elegidos por el usuario. I §8 define las ocupaciones (`GET /api/appointment-occupancies`:
posiciones `CONFIRMED` y `HELD`, rango de fechas inclusivo, lista vacía válida) e I §17 muestra el recorrido: KMP le pide la
disponibilidad a turnos, turnos consulta la agenda al catálogo y las ocupaciones a la cátedra, y le devuelve los turnos libres.
Ni E ni I fijan la ruta ni el formato de lo que turnos le ofrece a KMP: lo decide este servicio (E §5, contratos entre los
dos servicios).

`availability` es el primer módulo que depende de otro. Según el ADR-0002 solo puede ver el `domain.model` y los
`domain.ports.in` de `professional`, de modo que no puede atrapar las excepciones de `application` de ese módulo. T5 dejó
fuera el caso de uso de la agenda de un profesional porque lo necesita este módulo.

## Opciones consideradas

### Cómo avisa un módulo a otro que algo falló (pendiente de `DECISIONES.md`)
1. **Un tipo resultado en `domain.model` del módulo que responde.** + Es lo previsto en el ADR-0002 y el patrón de T2
   (`RegistrationResult`); no toca las reglas de ArchUnit. − Cada falla nueva es una variante más del resultado.
2. **Excepciones en `domain.exception`.** + Estilo clásico de Java. − Obliga a ampliar la regla de ArchUnit para dejar ver otro
   paquete y a enmendar el ADR-0002.

### Dónde vive la configuración del cliente de la cátedra
1. **Una sola configuración compartida en `config/`.** + La cátedra la usan `availability` ahora y `reservation` después
   (holds, confirmar, listar, cancelar); la URL, el JWT y los tiempos de espera se definen una vez. − Un `RestClient` como bean
   compartido.
2. **Un cliente configurado en cada módulo.** + Más aislado. − Duplica la URL, el JWT y los tiempos de espera.

### Contrato hacia la app
1. **Recurso del profesional:** `GET /api/professionals/{professionalId}/availability`. − La URL cuelga de `professionals`, que
   es de otro módulo, y un id mal formado en la ruta necesita su propio manejo de error.
2. **Parámetros planos:** `GET /api/availability?professionalId&from&to`. + La ruta pertenece al módulo que la implementa;
   reutiliza el patrón de validación de T5 (objeto de parámetros con Bean Validation); se parece a la ruta de la cátedra.

### Turnos que ya pasaron
1. **Excluirlos, con una zona horaria configurable.** + La app no muestra turnos imposibles. − La zona de las «horas locales de
   atención» (I §4) no está fijada por las fuentes: es un supuesto.
2. **Ofrecer todo lo que dice el horario.** + Sin supuestos sobre la zona. − Ofrece turnos que ya empezaron.

## Decisión
- **Resultado entre módulos** (opción 1). `professional` expone `GetProfessionalAgendaUseCase`, que devuelve un
  `ProfessionalAgendaResult` (`Found`, `NotFound`, `CatalogUnavailable`, `CatalogFailed`); su implementación traduce las
  excepciones del cliente del catálogo a esas variantes y deja el detalle en el registro. `availability` devuelve a su vez un
  `AvailabilityResult`. Criterio general: **entre módulos, resultados; hacia sistemas externos, excepciones** (ADR-0005). La
  fachada de cada módulo traduce el resultado a excepciones propias y la capa web las traduce a HTTP (como en T2).
- **Configuración compartida de la cátedra en `config/`** (opción 1): `CatedraProperties` (`app.catedra.base-url`,
  `app.catedra.jwt`, tiempos de espera de 2 s / 5 s) y un `RestClient` de la cátedra con el JWT técnico en el encabezado
  `Authorization`. Los módulos lo reciben por inyección y no importan clases de otro módulo. El JWT técnico es el de la cuenta de
  la cátedra (I §5): se lee de `CATEDRA_JWT`, nunca se versiona ni se loguea (`toString` de las propiedades lo oculta) y no se
  renueva. Un `401` o `403` de la cátedra es un error de integración, no un motivo para reintentar. A la cátedra nunca viaja la
  identidad del usuario.
- **Ruta con parámetros planos** (opción 2): `GET /api/availability?professionalId=15&from=2026-11-23&to=2026-11-29`, los tres
  obligatorios (`professionalId` positivo; fechas `yyyy-MM-dd`). Respuesta `200`:
  `{ professionalId, from, to, slots: [{ date, startTime, endTime }] }`, ordenada por fecha y hora de inicio. Las horas salen
  como `HH:mm:ss`, igual que en el catálogo y la cátedra. Protegida con el JWT del usuario por la cadena de T3.
- **Excluir los turnos que ya empezaron** (opción 1), con la hora actual tomada de un `Clock` en la zona
  `app.business-time-zone` (variable `BUSINESS_TIME_ZONE`, por defecto `America/Argentina/Buenos_Aires`). Un turno cuyo inicio es
  igual o anterior al momento actual no se ofrece.

### Reglas del cálculo
- Los turnos salen de los horarios semanales **habilitados** (`enabled=true`) del profesional, según el día de la semana de cada
  fecha del rango. Cada horario genera turnos desde `startTime`, de a `slotDurationMinutes`, mientras el turno entre completo
  antes de `endTime`: **un turno parcial no se ofrece** (de 9:00 a 10:00 con turnos de 45 minutos solo existe el de 9:00).
- Una ocupación (`CONFIRMED` o `HELD`) bloquea el turno que **empieza** a esa fecha y hora. Los dos estados bloquean por igual.
- Si dos horarios del mismo día generan un turno con el mismo inicio, se ofrece uno solo.
- Un profesional **deshabilitado** devuelve `200` con `slots: []` y no se consulta a la cátedra.
- Se consulta la agenda al catálogo y las ocupaciones a la cátedra **en cada pedido**; no se guarda nada (E §3, E §4.2).

### Errores
| Caso | HTTP | `code` |
|---|---|---|
| Parámetro ausente o con formato inválido | 400 | `VALIDATION_ERROR` |
| `from` posterior a `to` | 400 | `INVALID_DATE_RANGE` (el mismo `code` de la cátedra, I §13.1) |
| Rango de más de 31 días | 400 | `INVALID_DATE_RANGE` |
| Profesional inexistente | 404 | `PROFESSIONAL_NOT_FOUND` |
| Catálogo no disponible / error de integración | 503 / 502 | `CATALOG_UNAVAILABLE` / `CATALOG_ERROR` |
| Cátedra no disponible / error de integración | 503 / 502 | `CATEDRA_UNAVAILABLE` / `CATEDRA_ERROR` |

Los mensajes hacia la app son genéricos (sin URL, token ni detalles internos); el detalle queda en el registro.

## Justificación
El resultado como tipo de dominio resuelve el pendiente sin cambiar las reglas del ADR-0002 y deja un criterio simple y
reutilizable (T7 lo aplicará al «no reservable»). Compartir la configuración de la cátedra evita que `availability` y
`reservation` repitan la URL, el JWT y los tiempos de espera. La ruta plana pertenece al módulo y reutiliza la validación ya
probada. Excluir lo que ya pasó evita ofrecer turnos imposibles; el rango máximo protege del cálculo y de la consulta a la
cátedra para rangos enormes.

## Consecuencias
- `ProfessionalService` no cambia: la agenda no se expone a la app, solo a `availability` por su puerto de entrada.
- `config/` pasa a tener `CatedraProperties` y `CatedraRestConfig`; hace falta `CATEDRA_BASE_URL` y `CATEDRA_JWT` para arrancar.
  Los tests de contexto los reciben por propiedades.
- **Supuestos a confirmar con la cátedra:** la zona horaria de las «horas locales de atención», qué hace la cátedra con un turno
  pasado y el límite de 31 días (no figura en ninguna fuente).
- **Limitación:** la disponibilidad es una foto del instante de la consulta. Entre la consulta y el hold otro usuario puede tomar
  el turno; el hold (T7) lo resuelve y debe revalidar con un puerto de entrada de `availability`.
- **Limitación:** si dos horarios solapados del mismo día generan turnos que empiezan igual pero terminan distinto, se ofrece el
  primero que aparece.
- El contrato hacia la app se publica en Swagger; el documento de contratos de E §12 se completa en T14.

## Pendiente
- Probar el contrato a mano en Swagger con el catálogo real en el 8080 y la cátedra real (último ítem de T6).
