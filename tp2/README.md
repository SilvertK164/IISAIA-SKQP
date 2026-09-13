# TP 2 — API de inspecciones de obra

Un `openapi.yaml` que describe una API donde cada inspección de obra agrupa las observaciones que se levantaron durante ella. Seis endpoints, cuatro paths, sin nada implementado: el entregable es el contrato.

## Cómo se lee

Pegar el contenido de [openapi.yaml](openapi.yaml) en [editor.swagger.io](https://editor.swagger.io). Aparece la documentación navegable del lado derecho, con cada endpoint desplegable.

## Qué me propuse construir

Un dominio que conozco de trabajar en obra, no un ejemplo de manual. Una inspección es una visita fechada con un responsable, y las observaciones son los hallazgos que quedan registrados en esa visita: un desprendimiento en el eje 4, un encofrado mal apuntalado, una armadura sin recubrimiento. La relación es de pertenencia real —una observación fuera de una inspección no tiene fecha, ni inspector, ni contexto— así que la jerarquía en el path se justifica sola.

Salió en cinco prompts, en una sola conversación.

## Decisiones que tomé yo

**Anidar `observaciones` dentro de `inspecciones` en vez de `/observaciones?inspeccion=4`.** La decisión de fondo del contrato. Elegí anidar porque la pertenencia es estructural: una observación se levanta *durante* una inspección y hereda de ella la fecha y el responsable. Si el modelo admitiera observaciones sueltas que después se asocian a una visita, el path plano con filtro sería la forma correcta. Las dos son válidas; lo que no es válido es elegir sin darse cuenta de que se está eligiendo.

**Schemas de entrada y de salida separados.** `InspeccionInput` y `Inspeccion` no son el mismo objeto, y `ObservacionInput` y `Observacion` tampoco. Los de salida tienen `id` y `inspeccion_id`, que los genera o los deduce el servidor; los de entrada no los tienen porque el cliente no los manda. Colapsarlos en un solo schema con campos opcionales esconde justamente eso.

**`inspector` requerido al crear, `ubicacion` opcional.** Son las dos caras de la misma pregunta y las resolví distinto a propósito. Una inspección sin responsable no es una inspección: si nadie firma la visita, el registro no sirve para nada. Una observación sin ubicación, en cambio, existe todo el tiempo — los hallazgos generales de obra, los que no aplican a un sector puntual, son casos normales y no defectuosos.

**`204` sin cuerpo en el borrado.** Devolver `200` con la observación borrada adentro es lo que sale por default y no tiene sentido: si el recurso ya no existe, mandarlo de vuelta es describir algo que no está.

**El `404` del `DELETE` no reusa el component `InspeccionNoEncontrada`.** El resto de los `404` del contrato significan una sola cosa: la inspección no existe. El del borrado cubre dos —puede faltar la inspección o puede faltar la observación— así que lleva descripción propia. Reusar el component ahí habría sido más prolijo y menos cierto.

**Un schema `Error` con `detalles` por campo.** La consigna pedía documentar un error, no darle forma. Le puse `codigo`, `mensaje` y una lista opcional de `{campo, mensaje}` porque un `400` de validación que no dice qué campo falló obliga al cliente a adivinar. Es la diferencia entre un error que se puede mostrar en un formulario y uno que solo se puede loguear.

**Las inspecciones no se eliminan, se archivan.** Empecé con un `DELETE /inspecciones/{inspeccionId}` con borrado en cascada, y lo saqué. En construcción la trazabilidad es parte del trabajo: hay que poder reconstruir qué se observó, cuándo y quién firmó, porque un hallazgo levantado en obra puede terminar en un expediente meses después. Un endpoint que destruye una inspección y arrastra sus observaciones va en contra de eso, y ninguna advertencia en la `description` lo arregla — a lo sumo lo avisa.

Lo reemplacé por `PATCH /inspecciones/{inspeccionId}` con un campo `archivada`. La decisión de method es la parte importante: archivar es un cambio de estado del recurso, no una eliminación, así que `DELETE` habría sido un method que no hace lo que su nombre dice. El registro sobrevive, las observaciones siguen accesibles por su path, y la operación es reversible en los dos sentidos.

**`archivadas` como query param en el listado.** Archivar solo sirve si lo archivado desaparece de la vista por default, así que `GET /inspecciones` devuelve únicamente las activas y hay que pedir las archivadas explícitamente. Va en query y no en el path porque no identifica un recurso distinto: es la misma colección, filtrada. El path identifica, el query modifica.

**Las observaciones sí se borran.** Es una asimetría deliberada con el punto anterior. Una observación cargada por error —el sector equivocado, un duplicado— es ruido que conviene sacar, y no tiene valor probatorio por sí sola. La inspección es otra cosa: es el registro de que alguien recorrió la obra un día determinado, y eso no se destruye.

**`estado` y `resuelta` son escribibles al crear.** Lo natural sería que los moviera solo el servidor: una inspección nace `programada`, una observación nace sin resolver. Los dejé en los schemas de entrada a propósito, porque la API no sirve únicamente para operar en vivo — tiene que poder recibir el registro de una inspección de hace tres meses, ya cerrada, con sus observaciones ya levantadas. Si los sacara del input, cargar el historial de una obra sería imposible sin endpoints aparte. El costo es que un cliente puede crear una inspección directamente en `cerrada`, y lo asumo: es una API de registro, no un workflow con máquina de estados.

## Qué salió mal y cómo lo corregí

En el primer prompt describí los recursos por su forma completa: `Inspeccion { id, obra, fecha, inspector, estado }` y `Observacion { id, descripcion, severidad, ubicacion, resuelta, inspeccion_id }`. El modelo hizo lo razonable con eso: puso todos los campos como `required` en los schemas de salida y armó los de entrada sacando solamente lo que el servidor genera.

El resultado fue un contrato con dos campos huérfanos. `Inspeccion` declaraba `inspector` como requerido, pero `InspeccionInput` no lo pedía ni le daba un default. Lo mismo con `ubicacion` en `Observacion`. O sea que un cliente podía crear una inspección mandando solo `obra` y `fecha` —perfectamente válido según el contrato— y el servidor quedaba obligado a devolver un objeto con un campo que nadie le dio y que no puede deducir. El contrato se contradecía consigo mismo, y ninguna de las dos mitades era incorrecta leída sola.

No lo vi leyendo el yaml de corrido. Apareció recién al cruzar los `required` de cada par entrada/salida y preguntarme, campo por campo, de dónde sale el valor: del cliente, de un default, o del servidor. Los que no tenían ninguna de las tres respuestas eran exactamente esos dos. `estado`, `severidad` y `resuelta` no tenían el problema porque llevan `default` en el input, y `id` e `inspeccion_id` tampoco porque los pone el servidor.

El problema no era el yaml, era el prompt. Yo escribí la forma del recurso y la usé como si fuera también la forma del pedido, sin separarlas. El modelo copió esa confusión. La regla que me llevo: describir un recurso no es describir cómo se crea, y un campo requerido en la respuesta tiene que tener un origen declarado en el contrato — cliente, default o servidor. Si no lo tiene, el contrato le está pidiendo al servidor que invente.

Hubo un segundo hallazgo, menor pero de la misma familia. Los defaults de `estado` y `severidad` estaban escritos como hermanos de un `$ref`. En OpenAPI 3.1 es legal, pero muchas herramientas ignoran esos hermanos en silencio — y acá el default no era cosmético, era lo único que sostenía la consistencia entre entrada y salida. Los envolví en `allOf` para que sobrevivan al ecosistema.

## Prompts

El registro completo está en [prompts.md](prompts.md). El que más pesó fue el primero, que fija los cuatro endpoints iniciales y la separación entre schemas de entrada y de salida; el segundo agrega el borrado de observaciones y corrige los campos huérfanos.

## Herramientas

El contrato se construyó en una sola conversación de IA, como pide la consigna. Usé Claude por fuera de esa conversación para auditar el yaml; el cruce de `required` entre schemas de entrada y salida salió de esa revisión.
