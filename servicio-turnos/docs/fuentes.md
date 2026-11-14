# Fuentes de verdad y mapa de lectura

Dos documentos mandan sobre todo lo demás, y viven fuera de este repo:

| Sigla | Archivo | Qué es |
|---|---|---|
| **E** | `../../PROJECT_STATEMENT-v1.md` | Enunciado: qué hay que construir, restricciones, evidencias, entrega |
| **I** | `../../INTEGRATION_REFERENCE-v2.md` | Contratos REST / Redis / Kafka con la cátedra y recuperación |

Si `docs/` o el código contradicen a E o I, gana E o I. Las citas usan `E §6.1`, `I §18.2`.

> Los números de línea son de E v1 e I v2. Si los archivos cambian, relocalizar con
> `grep -n '^## \|^### ' <archivo>` y actualizar esta tabla.

## Qué leer de E para este servicio

| Tema | Sección | Líneas |
|---|---|---|
| Dos servicios, contratos con JWT, prohibiciones (sin base compartida, sin réplica del catálogo) | E §3 | 38-60 |
| Registro y login del usuario final (compatible con JHipster) | E §3.2 | 68-84 |
| Responsabilidades de turnos | E §4.2 | 103-118 |
| Alcance funcional obligatorio | E §5 | 120-138 |
| Disponibilidad y flujo de reserva | E §7 | 158-178 |
| Robustez y recuperación | E §8 | 180-195 |
| Seguridad (JWT, propiedad por usuario, autorización) | E §9 | 197-220 |
| Decisiones a cargo del alumno y pruebas | E §10, §10.1 | 222-244 |
| Evidencias mínimas | E §11 | 246-265 |
| Documentación obligatoria | E §12 | 267-280 |

## Qué leer de I para este servicio

| Tema | Sección | Líneas |
|---|---|---|
| `groupId`, JWT técnico vs usuario final, `externalPatientId` | I §2 | 11-26 |
| Convenciones (fechas, JSON, `problem+json`, Kafka at-least-once) | I §4 | 40-51 |
| Config de integración (hosts, topics) | I §5.1-5.3 | 55-173 |
| Mapa REST | I §6 | 179-193 |
| **Ocupaciones** `GET /api/appointment-occupancies` | I §8 | 251-292 |
| **Crear hold** | I §9 | 294-326 |
| **Confirmar hold** | I §10 | 328-362 |
| **Listar reservas** (de toda la cuenta técnica: filtrar por usuario) | I §11 | 364-407 |
| **Cancelar reserva** | I §12 | 409-436 |
| Errores REST y matriz completa | I §13, §13.1 | 438-509 |
| **Kafka**: convenciones y topics | I §15.1-15.2 | 631-650 |
| Kafka `AdditionalInformationRequested` | I §15.4 | 669-687 |
| Kafka `AdditionalInformationSubmitted` | I §15.5 | 689-711 |
| Kafka `AppointmentConfirmed` | I §15.6 | 713-733 |
| Kafka `AdditionalInformationRejected` | I §15.7 | 735-755 |
| Kafka `AppointmentProcessExpired` | I §15.8 | 757-773 |
| Kafka `AppointmentProcessInvalid` | I §15.9 | 775-793 |
| Kafka `AppointmentCancelled` | I §15.10 | 795-818 |
| **Diagrama de reserva** | I §17 | 849-892 |
| Recuperación: duplicado, hold vencido, reintentos | I §18.1, 18.3, 18.4 | 896-900, 908-916 |
| Límite del contrato (qué decide el alumno) | I §19 | 918-930 |

(Snapshot, Redis y `CatalogUpdated` son del servicio de catálogo: no se leen acá.)

## Contratos propios entre los dos servicios
- Catálogo → turnos: `../../servicio-catalogo/docs/contratos/catalogo-para-turnos.md` (token, operaciones, modelos,
  errores y reglas de uso).
