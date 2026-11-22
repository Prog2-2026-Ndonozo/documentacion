# ADR-0006: Búsqueda y categorías para la app KMP (módulo `professional`)

Fecha: 2026-11-22
Estado: aceptada

## Contexto
E §4.1 pide filtros por categoría, nombre, estado habilitado y disponibilidad, y que se puedan usar desde la interfaz KMP. El
ADR-0036 del catálogo decidió que **KMP habla solo con turnos** y que turnos le reenvía la búsqueda y las categorías al
catálogo; el catálogo conserva sus rutas. E §3 y E §4.2 prohíben replicar el catálogo: reenviar la consulta en cada pedido no
es una réplica.

T4 dejó el lado de salida (el cliente del catálogo con el JWT técnico). T3 dejó la protección con el JWT del usuario. T5 las une
y expone el módulo `professional` hacia afuera: casos de uso, fachada, controller, errores y el contrato publicado en Swagger.
El caso de uso de la agenda de un profesional queda fuera: lo necesita `availability` y se crea en T6, junto a quien lo usa.

## Opciones consideradas

### Rutas y forma de la respuesta hacia KMP
1. **Espejo del catálogo.** `GET /api/professionals/search` y `GET /api/professional-categories`, con los mismos parámetros y los
   mismos campos, sin valores por defecto ocultos. + Un contrato que ya está documentado y probado en el catálogo; el
   ADR-0036 dice que esas rutas se mantienen. − KMP recibe `categoryId` y no el nombre de la categoría.
2. **Diseño propio** (`/api/professionals?...`, `/api/categories`). + Más libre. − Hay que inventar y documentar otro contrato.

### Cómo se traducen los errores del catálogo a HTTP
1. **`503` y `502`.** Catálogo no disponible (no responde, tiempo de espera, `5xx`) → `503` `CATALOG_UNAVAILABLE`; error de
   integración (token rechazado, `403`, `400`) → `502` `CATALOG_ERROR`. + Distingue «reintentá más tarde» de «hay un error que
   arreglar».
2. **`503` para los dos.** + Más simple. − Esconde la diferencia.
3. **`500` para el de integración.** − `502` es más preciso: la falla viene del servicio al que se le reenvió el pedido.

### Dónde viven los manejadores de errores genéricos
1. **En `config/`.** Un `GlobalExceptionHandler` con la validación y el JSON roto; cada módulo conserva solo las excepciones de
   su dominio. + Lo previsto en el ADR-0004 («cuando haya un segundo módulo»). − Obliga a refactorizar el handler de T2.
2. **Duplicarlos en `professional`.** + No toca T2. − Dos handlers atrapan las mismas excepciones y el resultado depende del
   orden de Spring.

### Dónde se valida lo que escribe el usuario
1. **En la capa web con Bean Validation.** + Igual que registro y login; `400 VALIDATION_ERROR` con los campos.
2. **En el caso de uso con excepciones propias.** − Más código para validaciones de formato.

## Decisión
- **Espejo del catálogo** (opción 1). Rutas, parámetros y campos idénticos; si KMP no manda `enabled`, no se filtra. La respuesta
  de la búsqueda no agrega el nombre de la categoría: obligaría a una segunda llamada al catálogo por cada búsqueda, y KMP ya
  tiene la lista de categorías. No hay paginación (el catálogo tampoco la ofrece) ni se reordena: el orden es el del catálogo.
- **`503` / `502`** (opción 1), con los `code` `CATALOG_UNAVAILABLE` y `CATALOG_ERROR`. Los mensajes hacia la app son genéricos:
  no llevan la URL del catálogo, el token ni detalles internos. El detalle real queda en el registro (`WARN` si no está
  disponible, `ERROR` si es de integración).
- **Handlers genéricos en `config/`** (opción 1): `GlobalExceptionHandler` (validación y JSON roto) y `FieldViolation`. El handler
  de `user` pasa a llamarse `UserExceptionHandler` y conserva sus tres excepciones; el de este módulo es
  `ProfessionalExceptionHandler`. Los nombres deben ser distintos: dos clases con el mismo nombre simple en módulos distintos
  hacen fallar el arranque de Spring por conflicto de nombres de bean.
- **Validación en la capa web** (opción 1) con Bean Validation sobre un objeto de parámetros: `categoryId` positivo, `name` de
  hasta 100 caracteres, `enabled` booleano y `availableOn` en formato `yyyy-MM-dd` (los mismos límites del catálogo). Un
  valor que ni siquiera se puede convertir (`categoryId=abc`, una fecha mal escrita) responde `400 VALIDATION_ERROR` con el
  campo y el mensaje genérico «tiene un formato inválido»: el mensaje técnico de Spring nombra clases internas y se
  reemplaza. Un nombre en blanco equivale a no filtrar, igual que en el catálogo. Un pedido inválido **no llega** al catálogo.
- **Protección:** las dos rutas exigen el JWT del usuario por la cadena de T3, sin configuración extra. La identidad no viaja al
  catálogo: este recibe solo el token técnico RS256 de turnos (ADR-0005), nunca el del usuario.
- **Contrato publicado en Swagger** (`OpenApiConfig`, solo con el perfil `dev`): título, esquema `bearer` y botón *Authorize*;
  `register` y `authenticate` se marcan sin candado por ser públicos. Cada operación documenta sus respuestas (`200`, `400`,
  `401`, `502`, `503`).

## Justificación
Reutilizar las rutas y los campos del catálogo es la forma más barata de tener un contrato ya probado, y el ADR-0036 lo
anticipa. Separar `503` de `502` le da a la app dos comportamientos distintos (reintentar o informar un error). Mover lo
genérico a `config/` evita que cada módulo nuevo repita la validación y deja una sola forma de responder a un pedido mal
formado. Validar en la web sigue el criterio de T2 y protege al catálogo de pedidos que sabemos inválidos: un `400` del
catálogo sería, para turnos, un error de integración.

## Consecuencias
- **Se refactoriza T2:** `ApiExceptionHandler` de `user` se renombra a `UserExceptionHandler` y pierde sus dos handlers
  genéricos; `FieldViolation` pasa de `user` a `config`. Los tests de la capa web importan `GlobalExceptionHandler`.
- **Sin caché:** cada consulta de KMP vuelve a preguntarle al catálogo (E §3, E §4.2). Si el catálogo cae, la búsqueda cae: no se
  responde con datos viejos.
- **Limitación:** sin paginación, la respuesta crece con el catálogo. Aceptable para el volumen del proyecto.
- **Limitación:** el orden y los datos son los del catálogo en ese instante; entre dos llamadas pueden cambiar (el catálogo
  converge con la cátedra por Kafka).
- La agenda de un profesional (`GET /api/professionals/{id}/agenda`) no se expone a KMP: la usa `availability` internamente
  (T6). Cuando se cree, un id mal formado en la ruta necesitará su propio manejo de error.
- El contrato hacia KMP se documenta en Swagger; el documento de contratos de E §12 se completa en T14.

## Pendiente
- Probar el contrato a mano en Swagger con turnos en el 8081 (perfil `dev`) y el catálogo real en el 8080 (último ítem de T5).
