# Plantilla de ADR (Architecture Decision Record)

Copiá este bloque una vez por decisión importante, numerá el archivo (`docs/adr/0001-motor-de-base-de-datos.md`, `0002-...`) y guardalo en tu repo. El enunciado pide justificar decisiones (sección 10 y 12) — este archivo es literalmente esa justificación, ya escrita.

```markdown
# ADR-0001: <título corto de la decisión>

Fecha: 2026-09-08
Estado: aceptada | reemplazada por ADR-00XX

## Contexto
¿Qué problema o requisito del enunciado/contrato motiva esta decisión?

## Opciones consideradas
1. Opción A — ventajas / desventajas
2. Opción B — ventajas / desventajas

## Decisión
Qué elegiste.

## Justificación
Por qué, en relación al contexto del punto anterior.

## Consecuencias
Qué implica esto para el resto del sistema (otros repos, tests, seguridad, etc.)
```

---

## Ejemplo completado, para que veas el nivel de detalle esperado

```markdown
# ADR-0002: Propagación de identidad entre servicio-turnos y servicio-catalogo

Fecha: 2026-09-08
Estado: aceptada

## Contexto
La sección 9 del enunciado exige JWT validado en la comunicación entre los
dos servicios propios, y pide definir si se propaga el JWT del usuario final
o se usa un JWT técnico interno, justificando trazabilidad y autorización.

## Opciones consideradas
1. Propagar el JWT del usuario final tal cual a servicio-catalogo.
   + Trazabilidad directa (mismo token en toda la cadena).
   − servicio-catalogo tendría que validar tokens emitidos por servicio-turnos,
     acoplando sus esquemas de seguridad.
2. Emitir un JWT técnico "interno" (servicio-a-servicio) firmado con un
   secreto propio, distinto del JWT de usuario y del JWT técnico de cátedra,
   incluyendo el userId como claim.
   + Desacopla los dos servicios; cada uno valida solo lo que le corresponde.
   − Hay que mantener trazabilidad manualmente pasando el userId como claim.

## Decisión
Opción 2: JWT interno servicio-a-servicio, con el userId del usuario final
como claim, firmado con secreto compartido solo entre los dos backends
propios (externalizado por variable de entorno).

## Justificación
Mantiene la separación de responsabilidades exigida por el enunciado (cada
servicio dueño de su seguridad) sin perder la trazabilidad del usuario que
originó la operación.

## Consecuencias
- servicio-catalogo necesita un filtro de seguridad adicional para este JWT
  interno, distinto del que usa para requests de la app KMP.
- Los tests de integración entre servicios deben mockear/generar este token.
```
