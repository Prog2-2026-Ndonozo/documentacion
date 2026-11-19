# ADR-0001: PostgreSQL como base de datos y Flyway para las migraciones

Fecha: 2026-11-15
Estado: aceptada

## Contexto
E §3 exige que cada servicio sea dueño exclusivo de sus datos y administre sus propias migraciones, y que la persistencia
principal use un gestor de base de datos **servidor**: no se aceptan H2, SQLite ni otras bases embebidas o en memoria. Cada
alumno elige el motor. El esqueleto de septiembre de este repo trae H2 en el `pom.xml` (`runtime`), así que hay que sacarlo.
El servicio de catálogo ya resolvió lo mismo con PostgreSQL y Flyway (ADR-0017 del catálogo: SQL plano en
`db/migration`, Hibernate en `ddl-auto: validate`).

## Opciones consideradas
1. **PostgreSQL + Flyway (el mismo stack del catálogo).**
   + Una sola tecnología que mantener, probar y explicar en los dos servicios.
   + Ya está resuelta la receta: migraciones en SQL plano, `validate`, Testcontainers con PostgreSQL real.
   + Las trampas conocidas (nombre `V<n>__descripcion.sql`, migración aplicada no se edita) ya están en
     `docs/FLUJO-DE-TRABAJO.md`.
   − Ninguna relevante para este alcance.
2. **Otro servidor (MySQL o MariaDB) con Flyway.**
   + Cumple E §3.
   − Dos productos distintos en el mismo proyecto, sin ninguna necesidad funcional que lo justifique.
3. **PostgreSQL con Liquibase.**
   + Cumple E §3.
   − Otra herramienta de migraciones que aprender, sin ventaja para el alcance. Descartada por sobreingeniería.
4. **PostgreSQL con `spring.jpa.hibernate.ddl-auto=update`/`create`, sin migraciones.**
   − No es una migración administrada por el servicio (E §3) y deja el esquema sin historial ni revisión. Descartada.
5. **H2, SQLite u otra base embebida.** Prohibidas por E §3 como persistencia principal. Descartadas.

## Decisión
Opción 1: PostgreSQL propio y exclusivo de este servicio, con Flyway para las migraciones (SQL plano en
`src/main/resources/db/migration`) y Hibernate en `ddl-auto: validate`.

## Justificación
Es lo que pide E §3, reutiliza una receta ya probada en el catálogo y evita sumar un segundo producto o una segunda
herramienta de migraciones a un proyecto con fecha de entrega.

## Consecuencias
- Se saca H2 del `pom.xml` y se agregan el driver de PostgreSQL, Flyway y Testcontainers (T1).
- `docker-compose.yml` levanta una base **propia** de este servicio, con su propio usuario y sin permisos sobre la del
  catálogo (E §3). Credenciales en `.env` (gitignoreado) y `.env.example` versionado; puerto distinto al del catálogo, a
  fijar en T1.
- Los tests que dependen de la base usan PostgreSQL real con Testcontainers y necesitan Docker corriendo (mismo desvío
  respecto de la skill, que propone H2, que el registrado en el ADR-0033 del catálogo).
- Las migraciones de este servicio son independientes de las del catálogo: cada una empieza en `V1`.
