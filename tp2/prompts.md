# Prompts — TP 2

El registro del proceso, en orden. Cinco prompts en una sola conversación.

---

## 1 — Prompt inicial

```
Necesito un openapi.yaml (3.1) para una API de inspecciones de obra y sus observaciones.

recursos:
  Inspeccion   { id, obra, fecha, inspector, estado }
  Observacion  { id, descripcion, severidad, ubicacion, resuelta, inspeccion_id }

endpoints:
  GET    /inspecciones                              → 200 lista
  POST   /inspecciones                              → 201 / 400 si falta obra o fecha
  GET    /inspecciones/{inspeccionId}/observaciones → 200 lista / 404 si la inspección no existe
  POST   /inspecciones/{inspeccionId}/observaciones → 201 / 400 si falta descripcion / 404 si la inspección no existe

detalles:
- fecha con format date, severidad como enum (leve, moderada, critica),
  estado como enum (programada, en_curso, cerrada).
- Los schemas de entrada y de salida son distintos: el de salida incluye el id
  que genera el servidor, el de entrada no.
- El id de la inspección viaja solo en el path, nunca en el body.
```

**Qué buscaba:** fijar los cuatro endpoints y, sobre todo, dos cosas que el modelo no hace solo. La separación entre schema de entrada y de salida, porque si no la nombro suele devolver un único schema con el `id` marcado como opcional y ahí se pierde que el `id` lo genera el servidor. Y la última línea, que es la vacuna contra el error clásico de pedir el mismo dato dos veces: el `inspeccion_id` ya viaja en el path, no tiene que estar también en el body.

Volvió el yaml con los cuatro paths, `components/schemas` con los cuatro schemas separados, un `Error` con lista de detalles por campo, y el parámetro de path y la respuesta `404` factorizados como componentes reutilizables. `inspeccion_id` quedó fuera de `ObservacionInput`, como pedí.

---

## 2 — Agregar el borrado y cerrar los campos huérfanos

```
Dos cambios:

1. Agregá DELETE /inspecciones/{inspeccionId}/observaciones/{observacionId}, que
   devuelva 204 sin cuerpo si borró y 404 si la observación o la inspección no existen.
   Reusá el mismo schema Error para el 404.

2. Hay una contradicción entre los schemas de entrada y de salida. Inspeccion declara
   inspector como required y Observacion declara ubicacion como required, pero ninguno
   de los dos es required en su Input ni tiene default. Resolvelo así:
   - inspector pasa a ser required también en InspeccionInput.
   - ubicacion sale de los required de Observacion y queda opcional.
```

**Qué buscaba:** completar el tercer method y, en el mismo turno, cerrar los dos campos huérfanos que había encontrado al auditar el yaml (el detalle está en el README). Pedí `204` explícito porque si no lo digo el modelo tiende a devolver `200` con el objeto borrado, que es raro: si lo borraste, no tiene sentido devolverlo. Y le dicté la resolución de cada campo en vez de solo señalar la contradicción, porque la decisión de cuál se vuelve obligatorio y cuál opcional es de dominio, no de contrato — un modelo que la resuelve solo la va a resolver por simetría.

Agregó el path nuevo con su parámetro `observacionId` como componente, corrigió los dos `required`, actualizó la descripción del `400` de `POST /inspecciones` para incluir `inspector`, y no tocó nada más. Le puso al `404` del borrado una descripción propia en vez de reusar el component `InspeccionNoEncontrada`, que era lo correcto: ahí el error cubre dos causas y no una.

---

## 3 — Defaults que sobrevivan a las herramientas

```
Un último cambio, de portabilidad. En InspeccionInput.estado y ObservacionInput.severidad
tengo un `default` como hermano de un `$ref`. En 3.1 es válido, pero muchas herramientas
lo ignoran, y esos dos defaults son estructurales: son lo único que garantiza que `estado`
y `severidad`, que están en los required de los schemas de salida, tengan valor cuando el
cliente no los manda. Envolvé el $ref en allOf en los dos casos para que el default
sobreviva.
```

**Qué buscaba:** que la consistencia entre entrada y salida que acabábamos de arreglar no dependiera de qué herramienta lea el archivo. Es el mismo problema del prompt anterior visto una capa más abajo: un campo requerido en la respuesta necesita un origen declarado, y si ese origen es un `default` que el renderer descarta, vuelve el agujero.

---

## 4 — Borrado de inspecciones, con la cascada explícita

```
Agregá DELETE /inspecciones/{inspeccionId}:

- 204 sin cuerpo si borró.
- 404 si la inspección no existe: reusá el component InspeccionNoEncontrada.
- El borrado es en cascada: elimina también todas las observaciones de esa
  inspección. Documentalo en la description del endpoint, porque es una
  consecuencia que el cliente no puede deducir del path.
```

**Qué buscaba:** completar la simetría del contrato y no dejar implícito qué pasa con las observaciones al borrar su inspección. Un `DELETE` pedido a secas deja esa pregunta sin contestar, y el modelo la resuelve solo, en silencio y del modo que se le ocurra.

Lo devolvió bien, con la cascada en la `description` y el `404` reusando el component. El problema no fue la respuesta sino el pedido: al leer el endpoint terminado me di cuenta de que estaba contradiciendo mi propio dominio. El prompt 5 lo revierte.

---

## 5 — Archivar en vez de eliminar

```
Cambio de diseño: las inspecciones no se eliminan, se archivan. En obra un registro
tiene que sobrevivir por trazabilidad.

1. Sacá DELETE /inspecciones/{inspeccionId} por completo.

2. Agregá a Inspeccion un campo `archivada` (boolean, readOnly, required), que el
   servidor devuelve siempre. No va en InspeccionInput: toda inspección nace activa.

3. Agregá PATCH /inspecciones/{inspeccionId}, con un schema de entrada
   InspeccionPatch que tenga solo `archivada` (boolean, required). Devuelve 200
   con la Inspeccion actualizada, 400 si el cuerpo es inválido y 404 reusando
   el component InspeccionNoEncontrada. Documentá en la description que archivar
   no elimina las observaciones: siguen siendo accesibles por su path.

4. En GET /inspecciones agregá un query param `archivadas` (boolean, opcional,
   default false) que indica si el listado incluye las archivadas. Por default
   devuelve solo las activas.
```

**Qué buscaba:** corregir una decisión mía, no un error del modelo. El detalle está en el README. Dicté las cuatro piezas en un solo prompt porque son inseparables: sacar el `DELETE` sin dar una alternativa dejaba el contrato sin forma de retirar una inspección de circulación, y agregar `archivada` sin el query param en el listado lo habría vuelto un campo decorativo.

Lo aplicó completo y sin tocar nada de las observaciones, que era lo que quería comprobar.

---

## Conversación completa

Una sola conversación, sin reiniciar el hilo. El yaml final tiene 6 endpoints repartidos en 4 paths. Validado en `editor.swagger.io` antes de entregar.
