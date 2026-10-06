# Servicio de Catálogo y Sincronización — documentación

Backend del proyecto integrador 2026 (Programación 2). Mantiene la copia local del catálogo de categorías,
profesionales y horarios semanales, la sincroniza con el servicio central de la cátedra (snapshot completo e
incremental por Kafka + Redis) y resuelve las búsquedas de profesionales únicamente con datos locales.

Es uno de los dos backends del proyecto; el otro es `servicio-turnos-reservas`. Solo se comunican por HTTP con
JWT, sin acceso cruzado a bases de datos.

## Fuente única de la verdad

Todo el contrato de este servicio se deriva de dos documentos de la cátedra, que prevalecen sobre cualquier
otro texto:

- [`PROJECT_STATEMENT-v1.md`](../PROJECT_STATEMENT-v1.md) — enunciado y restricciones (**E**).
- [`INTEGRATION_REFERENCE-v2.md`](../INTEGRATION_REFERENCE-v2.md) — contratos REST, Redis y Kafka (**I**).

[`docs/fuentes.md`](docs/fuentes.md) indica qué sección de cada uno aplica a este servicio.

## Stack

Java 25 · Spring Boot 4.1 · Maven · PostgreSQL · Spring Data JPA · Kafka · Redis · springdoc (OpenAPI) ·
Lombok · ArchUnit.

## Requisitos y ejecución

- JDK 25; Docker y Docker Compose para PostgreSQL local (*pendiente: issue #2*).
- Cuenta técnica registrada en la cátedra (I §5); su JWT va en `cuenta_tecnica.json`, que **no** se versiona.

```bash
./mvnw test                # tests, incluidas las reglas de arquitectura
./mvnw spring-boot:run     # ejecución local
```

La configuración (hosts de Redis/Kafka/API de la cátedra, credenciales) se externaliza en la issue #2
(`.env.example`). Ningún valor real se sube al repositorio.

## Arquitectura

Hexagonal, un único módulo `catalog` con tres capas (`domain`, `application`, `infrastructure`) más `config/`.
Reglas de dependencia automatizadas con ArchUnit. Detalle:
[`docs/arquitecturaHexagonalServicioCatalogo.md`](docs/arquitecturaHexagonalServicioCatalogo.md).

## Documentación

| Documento | Contenido |
|---|---|
| [`docs/fuentes.md`](docs/fuentes.md) | Índice de las fuentes de verdad por tema |
| [`docs/arquitecturaHexagonalServicioCatalogo.md`](docs/arquitecturaHexagonalServicioCatalogo.md) | Paquetes, puertos, fachadas, reglas ArchUnit |
| [`docs/BacklogServicioCatalogo.md`](docs/BacklogServicioCatalogo.md) | Issues, milestones y definición de terminado |
| [`docs/DECISIONES.md`](docs/DECISIONES.md) | Decisiones tomadas y pendientes |
| [`docs/glosario.md`](docs/glosario.md) | Vocabulario del proyecto |
| [`AGENTS.md`](AGENTS.md) | Contexto y reglas para agentes de IA |

## Convenciones

Un issue = una rama = un PR. Commits con [Conventional Commits](https://www.conventionalcommits.org/).
