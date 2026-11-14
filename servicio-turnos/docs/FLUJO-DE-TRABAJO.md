# Flujo de trabajo con el agente

Cómo trabajamos en el servicio de catálogo y cómo seguimos acá. Si es la primera vez que leés esto en una sesión, es el
contrato de colaboración: respetalo aunque el pedido del momento no lo repita.

## Fechas clave
- **17/11/2026**: reunión de seguimiento con el profesor. Se muestra el catálogo funcionando y avances de este servicio.
- **29/11/2026**: entrega de los dos servicios (E §13.1: últimos días de noviembre).

## Roles
- **Nehuen (alumno)** decide el diseño y **escribe el código de producción**: aprende arquitectura hexagonal, así que el
  agente no lo reemplaza. Hay excepciones solo cuando lo pide de forma explícita ("escribí vos estos archivos").
- **El agente** es copiloto: lee las fuentes, propone opciones con una recomendación, redacta los ADR y la documentación,
  **revisa** lo que el alumno escribe (errores de compilación, imports, desvíos de diseño), **escribe y corre los tests**
  y verifica. No commitea, no crea ramas, no pushea; recuerda usar Conventional Commits.

## Ciclo de cada funcionalidad
1. **Leer las fuentes** que correspondan (`docs/fuentes.md`): E, I y los ADR previos. Citar `E §x` / `I §x`.
2. **Opciones y recomendación.** Presentar 2 a 4 alternativas con a favor y en contra, y recomendar una con motivo. Si el
   contrato no define algo, **decirlo** y proponer un supuesto marcado como tal (confirmable con la cátedra).
3. **El alumno decide.** No avanzar sobre una decisión de diseño sin su confirmación.
4. **ADR.** Redactarlo con las opciones consideradas (`docs/adr/ADR-template.md`), agregar su fila en `docs/DECISIONES.md`
   y actualizar el backlog. Estado `propuesta` hasta que confirme; después, `aceptada`.
5. **Mostrar el código en el chat**, por archivo y con los imports, explicando por qué. Antes de mostrarlo, verificarlo en
   una **copia del proyecto** (no en el repo del alumno): compila y pasan sus tests.
6. **El alumno escribe.** El agente revisa lo escrito: compilar, listar los errores, explicar la causa. Si el alumno pide
   explícitamente que el agente escriba código, hacer **reemplazos puntuales** y comprobar antes con `git diff` que no se
   pisan cambios locales.
7. **Tests.** El agente los escribe y los copia al proyecto cuando el código compila. Probar contra PostgreSQL real
   (Testcontainers) lo que dependa de la base; usar el decodificador real al probar seguridad.
8. **Suite completa** (`./mvnw test`, con Docker). Informar el conteo y lo que no pudo correr.
9. **Commits atómicos**, con mensajes Conventional Commits y separando `feat`/`fix`, `test` y `docs`. Excepción: si el
   commit del código quedaría con tests en rojo, el código y su test van juntos.
10. **PR**: título breve y descripción con qué hace, cambios, cómo se probó, pendientes y `Closes #n`.

Un issue de GitHub por milestone; una rama por issue; un PR por rama.

## Espejo de la documentación
Todo lo que está en `docs/` de este repo se refleja en el repo de documentación, en `../documentacion/servicio-turnos/docs/`
(y el del catálogo en `../documentacion/servicio-catalogo/docs/`). **El `docs/` del repo del servicio manda**: el espejo debe
tener el mismo contenido y difiere solo en las rutas relativas (el espejo está un nivel más abajo). Se actualiza en cada
cambio de documentación, **antes** de commitear. Para comprobar la alineación:
`diff -rq docs ../documentacion/servicio-turnos/docs` (solo deben diferir las líneas con rutas relativas).

## Cómo verificar sin romper nada
- Trabajar en una **copia** (`cp -r pom.xml mvnw .mvn src <scratchpad>`, el directorio temporal de la sesión) para probar
  código nuevo o variantes. Ese directorio puede borrarse entre días: lo que importe conservar, llevarlo al proyecto.
- **No arrancar la aplicación real** desde el agente: consume offsets de Kafka y escribe en la base del alumno. Pedirle que
  la pruebe a mano y darle los comandos exactos y las respuestas esperadas.
- **No llamar a la cátedra** (REST, Kafka ni Redis) desde el agente: es un servicio central compartido y sus operaciones
  (crear un hold, confirmar, cancelar) generan reservas reales. Los comandos los ejecuta el alumno. Tampoco leer ni imprimir
  `.env` ni `cuenta_tecnica.json`.
- Consultas a la base del alumno: **solo lectura**, cargando el `.env` en un subshell
  (`( set -a; source .env; set +a; docker compose exec -T <servicio-de-la-base> psql -U "$DB_USERNAME" -d "$DB_NAME" -c "..." )`) y sin
  imprimir secretos.
- Para reproducir un problema de seguridad o de despacho de errores usar un **servidor real**
  (`@SpringBootTest(webEnvironment = RANDOM_PORT)`): `MockMvc` no repite el despacho de errores.
- Revisar la **mutación**: antes de dar por buena una regla de ArchUnit o un test de seguridad, comprobar que falla cuando
  se rompe lo que protege.

## Estilo
- Documentación, mensajes y nombres de test en español; identificadores del dominio en inglés (como en el catálogo).
- Tests con nombres descriptivos en `snake_case`. Mockito con `MockitoExtension` (stubs estrictos).
- Todo número que se afirme (tests, versiones) sale de una corrida real. Si no se corrió, decirlo.
- **Preferir lo simple.** Si el alumno descarta algo por sobreingeniería (por ejemplo, ignorar tildes o paginar), se anota como
  descartado en el ADR y se sigue. Hay fechas de entrega.
- Los mensajes de commit llevan como alcance el módulo: `feat(user): ...`, `test(reservation): ...`, `docs: ...`.
- Los ADR se numeran `NNNN-titulo-en-minusculas.md` (cuatro dígitos) y se enlazan desde `docs/DECISIONES.md`.
- **Si el agente se equivoca, lo dice**, corrige el ADR o el código afectado y explica qué suponía mal (en el catálogo ocurrió
  con `issuer-uri`: el ADR decía una cosa y el código de Spring Boot hacía otra).

## Trampas ya conocidas
- **El autocompletado del IDE elige la clase equivocada**: `Sort` de springdoc en vez de `org.springframework.data.domain`,
  `java.beans.Expression` en vez de `jakarta.persistence.criteria`, dos `Info` con el mismo nombre. Revisar siempre los imports.
- **Un solo error de compilación puede dar decenas de `cannot find symbol`**: Lombok no llega a generar getters. Buscar la
  causa raíz.
- **Hibernate en `validate`**: el nombre de la tabla de la entidad debe coincidir con la migración, y si falla una entidad
  fallan todos los tests `@DataJpaTest`.
- **Flyway**: nombre `V<n>__descripcion.sql` (dos guiones bajos); una migración aplicada no se edita.
- **Spring Security**: con clave pública local **no** configurar `issuer-uri` (Boot descarta la clave y busca por red);
  validar el emisor con un `JwtIssuerValidator`. `/error` también atraviesa los filtros: permitir el despacho `ERROR`.
- **springdoc** va en la raíz del YAML (`springdoc:`), no bajo `spring:`; el perfil `dev` lo habilita (`application-dev.yaml`).
- **Java**: la terminal puede usar Java 24; el proyecto necesita 25.
- **Docker apagado** hace fallar los tests de Testcontainers con `Could not find a valid Docker environment`.

## Prompt de arranque para una sesión nueva
> Estoy en el repo `servicio-turnos-reservas`. Leé `AGENTS.md`, `docs/FLUJO-DE-TRABAJO.md` y `docs/DECISIONES.md`. Vamos a
> trabajar el issue `<n>` de `docs/BacklogServicioTurnos.md`. Seguí el flujo de trabajo: primero opciones con
> recomendación, yo decido, después ADR y recién ahí el código en el chat.
