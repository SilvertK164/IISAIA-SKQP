# Prompts — TP 1

El registro del proceso, en orden. Una sola conversación de Gemini Canvas, sin reiniciar el hilo. Cinco prompts: uno de arquitectura, tres de expansión y uno de corrección.

---

## 1 — Prompt inicial (Patrón 1: describir el artefacto)

```
Construí una single-page app de pago de servicios (luz y agua) en un solo archivo HTML.
Es un ejercicio de "bad UI" para un curso: la mecánica debe ser deliberadamente
frustrante, pero el código 100% funcional y la estética pulida. El chiste está en el
contraste entre lo bien diseñada que se ve y lo mal que trata al usuario.

CONCEPTO CENTRAL:
El usuario debe pagar un recibo, pero no elige cómo. Una ruleta decide qué mecánica
de pago le toca y qué monto parcial puede abonar en ese turno. Gana o pierde ese
parcial, y la ruleta vuelve a girar por el saldo restante hasta llegar a cero.

ARQUITECTURA (respetala estrictamente, es lo más importante del pedido):
- Un único objeto `state` con todo el estado de la app.
- Una única función `aplicar(accion)` que muta `state`. Ningún otro código muta estado.
- Una única función `render()` que redibuja la UI leyendo `state`.
- Un único bucle `requestAnimationFrame` global que llama a `update(dt)` del minijuego
  activo. Prohibido crear setInterval o rAF adicionales dentro de los minijuegos.
- Cada minijuego es un objeto con el MISMO contrato:
    { id, nombre, montar(contenedor, montoAsignado), update(dt), desmontar() }
  Al terminar llama a `finalizar(montoLogrado)`, que es la ÚNICA vía para acreditar.
  Ningún minijuego toca saldoPendiente ni el DOM fuera de su contenedor.
- Los minijuegos viven en un array `MINIJUEGOS`, para que agregar uno nuevo no
  requiera tocar nada más.

ESTADO:
saldoTotal (87.40), saldoPendiente, montoRevelado (bool), fase
('recibo'|'ruleta'|'jugando'|'fin'), metodoActual, montoAsignado,
historialGiros (array de {metodo, asignado, logrado}), girando (bool).

FLUJO:
1. Fase 'recibo': el monto está tapado por una capa gris en un <canvas>. Hay que
   raspar con el mouse (mousedown + mousemove, destination-out) para revelarlo.
   Se considera revelado al 60% de píxeles borrados. Sin revelar no se puede avanzar.
2. Fase 'ruleta': un <canvas> con sectores iguales por minijuego. Al girar, se
   desacelera con fricción y se detiene en uno. Sin re-giro. El montoAsignado es un
   valor aleatorio entre el 20% y el 50% del saldoPendiente, redondeado a 2 decimales.
3. Fase 'jugando': se monta el minijuego que salió. Al terminar, `finalizar(logrado)`
   descuenta del saldoPendiente y vuelve a 'ruleta'.
4. Fase 'fin': saldoPendiente <= 0. Comprobante con el historial de giros.

MINIJUEGO 1 — Cinta transportadora:
Monedas (0.10, 0.20, 0.50, 1.00, 2.00, 5.00) desfilan horizontalmente en un <canvas>.
El usuario clickea la moneda que quiere sumar; solo cuenta si el click cae dentro de
la moneda. Debe alcanzar exactamente el montoAsignado. Si se pasa, pierde el turno y
`finalizar(0)`. Tiene 30 segundos; al agotarse, `finalizar` con lo acumulado hasta ahí.
La velocidad de la cinta sube 10% cada 5 segundos.

ESTILO — neobrutalismo, aplicalo con rigor:
Tokens en :root: --bg #FBFBF9, --surface #FFFFFF, --text #1C293C, --muted #6B7280,
--primary #FDC800, --secondary #432DD7, --danger #DC2626, --border #1C293C,
--border-w 3px, --radius 8px, --shadow-offset 6px, y escala de espaciado
4/8/12/16/24/32.
Bordes de 3px sólidos sobre toda superficie. Sombras duras sin blur:
box-shadow: 6px 6px 0 var(--border). Escala tipográfica 13/15/17/21/27/35.
Fuentes del sistema (-apple-system para texto, ui-monospace para montos). Sin degradados.

COPY — pasivo-agresivo sutil, cortés en la superficie, sin insultos ni emojis:
- Al perder un turno: "Su aporte no pudo ser acreditado. Puede volver a intentarlo."
- Al girar: "Asignando su método de pago preferido."
- Al revelar el recibo: "Gracias por su paciencia."

MOVIMIENTO: transiciones de 180-220ms, cubic-bezier(0.2,0,0,1). Botones que se hunden
translate(3px,3px) al :active con la sombra en 0. Respetá prefers-reduced-motion.

RESPONSIVE: <main> centrado, max-width 560px. Los <canvas> escalan al ancho del
contenedor manteniendo proporción, con devicePixelRatio para que no se vean borrosos.
Todo legible abajo de 400px sin scroll horizontal.

CONSTRAINTS: un solo archivo index.html, HTML + CSS + JS vanilla en el mismo archivo.
Sin librerías, sin CDN, sin imports, sin frameworks, sin fuentes externas.
Debe abrirse con doble click.
```

**Qué intentaba lograr:** fijar el artefacto entero de una vez, nombrando las cinco capas — concepto, arquitectura de estado, estructura, estilo y constraints de empaque. Lo que más pesa acá no es la descripción visual sino el bloque de arquitectura. Un artefacto que va a crecer sumando mecánicas necesita que el contrato exista antes que la primera mecánica; si lo dejo para después, cada minijuego nuevo se lleva puesto al anterior, y con un LLM eso pasa en el prompt siguiente y no en tres semanas.

**Por qué un solo minijuego especificado:** describir los ocho de entrada habría producido un prompt imposible de auditar. Pedí uno completo como referencia del contrato, para que los siguientes salieran por analogía.

**Qué devolvió:** el flujo de cuatro fases funcionando, con raspado por `destination-out` y muestreo de píxeles, ruleta con fricción, canvas con `devicePixelRatio`, y la arquitectura respetada al pie de la letra: `state` único, `aplicar()` como única mutación, `render()` de solo lectura, un `requestAnimationFrame` global y `finalizar()` como única vía de acreditación.

**Dos decisiones que tomó el modelo por su cuenta:** agregó dos minijuegos más (Teclado errante y Pulso sostenido) con el argumento de que una ruleta de un solo sector no es una ruleta, y cambió el redondeo del monto asignado de 0.01 a 0.10, porque con monedas de 0.10 en adelante un objetivo como 21.37 es inalcanzable exacto.

**Qué hice con eso:** acepté las dos. La primera es correcta y además valida el contrato: los dos minijuegos nuevos entraron sin tocar nada del flujo. La segunda es un razonamiento sobre alcanzabilidad que yo no había hecho y que evitaba un juego imposible — aunque terminó teniendo una consecuencia que ninguno de los dos vio, y que está en el prompt 5.

---

## 2 — Pedido abierto de expansión

```
genial se ve super bien que mas le añadirias
```

**Qué intentaba lograr:** con el esqueleto andando, ver qué proponía el modelo antes de decidir yo. Es un prompt deliberadamente abierto y no lo escribí por comodidad: en un artefacto de humor la generación de ideas es barata y el filtro es lo caro, así que prefería tener diez opciones para descartar ocho.

**Qué devolvió:** una lista ordenada por rendimiento, separada en fricción barata (términos que se desmarcan solos, comisión por intento fallido, historial de fracasos visible), minijuegos extra que respetaban el contrato (captcha de recibos sin respuesta correcta, slider de precisión, cola de espera) y pulido (comprobante descargable, sonido sintetizado con AudioContext, contador de tiempo invertido). Cerró recomendando tres.

**Qué hice con eso:** no acepté la terna que sugirió. Me llevé la comisión por fallo, que es la que cambia la naturaleza del juego —convierte una tarea difícil en una que puede empeorar— y descarté el captcha de recibos y la cola de espera por alcance. El resto lo redirigí en el prompt siguiente.

**Lo que aprendí de este prompt:** un pedido abierto le sirve al inventario, no a la dirección. El modelo propone bien, pero jerarquiza por lo que es barato de implementar, no por lo que sostiene el concepto. La terna que recomendó era la más fácil de escribir, no la mejor.

---

## 3 — Dirigir la expansión hacia el monto y el cierre

```
algo mas que sea de monto a pagar quiza los minijuegos si se ven bien, y algo mas al finalizar
```

**Qué intentaba lograr:** corregir el rumbo de la lista anterior. Los minijuegos ya estaban bien cubiertos; lo flaco eran los dos extremos del recorrido. El principio, porque un monto que aparece solo no genera ninguna relación con el usuario. Y el final, porque la fase de cierre era un comprobante y nada más: el momento de mayor alivio del usuario, desperdiciado.

**Qué devolvió:** en el monto, el desglose por concepto (energía 52.10 + agua 35.30), un cargo de 1.50 por consulta de importe —raspar tiene precio— y un contador de mora que suma 0.50 cada 45 segundos. En el cierre, tres cajas de resumen (tiempo invertido, intentos fallidos, cargos adicionales), una calificación obligatoria de 1 a 5 donde solo la estrella 5 está habilitada, y descarga del comprobante en `.txt` por Blob, generado desde el `state`.

**Qué hice con eso:** lo acepté completo. El cargo por consulta de importe es la mejor idea de toda la conversación y no es mía: cobrar por mirar tu propio recibo condensa el concepto entero en una línea. La calificación con una sola opción habilitada es la misma broma aplicada al cierre.

**Nota de arquitectura:** las features nuevas entraron como acciones (`TICK_RECIBO`, `CALIFICAR`) y el `rAF` pasó a tickear también en fase recibo. Nada rompió el contrato, que era la prueba de fuego del prompt 1.

---

## 4 — Profundizar el recibo y la ruleta

*El texto literal de este prompt no quedó guardado. Por lo que devolvió, pedía: un pincel de raspado más fino; un código de identidad con vida útil que se oculta y cobra por volver a verse; cuatro cambios en la ruleta (pausa antes de montar el juego, una calibración que la hace retroceder, arranque automático por inactividad y un sector que sube el saldo), y una propina obligatoria al cierre.*

**Qué intentaba lograr:** endurecer el recibo y la ruleta, que después del prompt 3 todavía se atravesaban rápido: raspar con un pincel ancho llevaba segundos, y la ruleta daba un resultado y lo respetaba. Quería que el usuario perdiera el control incluso donde no había nada que jugar.

**Qué devolvió:** pincel de raspado de 44 a 26 px, y un código de identidad con vida útil de 8 segundos que después se oculta y cuesta 0.30 volver a ver. En la ruleta, cuatro cambios: pausa de 1.6 s anunciando el método antes de montar el juego; una "calibración" que el 30% de las veces hace retroceder la rueda y puede cambiar de sector; arranque automático a los 10 segundos de inactividad ("Iniciamos el giro por usted"); y un sector "Cambio de monto" que no se juega y sube el saldo un 10%. En el cierre, propina obligatoria de 10/15/20% sin opción de omitir, que se cobra recién en el recibo siguiente.

**Qué hice con eso:** lo acepté. La calibración de la ruleta es la pieza que mejor captura el concepto: el usuario ve el resultado, lo procesa, y se lo sacan. Con esto la ruleta llegó a ocho sectores.

---

## 5 — Corrección de cuatro bugs encontrados en Review y Test

```
Encontré cuatro bugs. Corregilos sin cambiar nada más del archivo:

1. La fase 'fin' es inalcanzable. El "Cargo por saldo menor" deja saldoPendiente
   clavado en 1.00, y calcularAsignado solo devuelve el total cuando pendiente <= 0.5,
   así que nunca llega a cero. Cambiá la condición de calcularAsignado a `pendiente <= 1`.

2. En cerrarPrecision, `Math.floor(valor * 10) / 10` falla por punto flotante:
   con valor 0.70, 0.70*10 da 6.999... y acredita 0.60. Calculalo en centavos:
   `Math.floor(cents(valor) / 10) / 10`.

3. En 'Deslizador fugitivo' la rama de acierto es inalcanzable. Al soltar la perilla,
   la deriva (mínimo 0.6/s) drena el valor antes de que el usuario llegue al botón
   Confirmar, y la tolerancia es de centavos. Aplicá el mismo patrón de gracia que ya
   usa 'Monto por desplazamiento': la deriva arranca recién 600 ms después de la última
   interacción con el rango.

4. En 'Monto por desplazamiento', si `max` (scrollHeight - clientHeight) es 0, el
   cálculo de `valor` da NaN y el contador muestra "$ NaN". Agregá una guarda que
   deje valor en 0 cuando max sea 0.
```

**Qué intentaba lograr:** cerrar los cuatro hallazgos del Review en un solo pedido en vez de cuatro idas y vueltas, minimizando la superficie de regresión sobre un archivo que ya tenía 1300 líneas.

**Por qué está escrito así:** cada punto nombra síntoma, causa y arreglo concreto. Pedir "arreglá que no se puede terminar el juego" habría dejado al modelo eligiendo cuál de las dos reglas sacrificar, y lo más probable es que hubiera eliminado el cargo por saldo menor, que es justamente la parte que quiero conservar. La frase "sin cambiar nada más del archivo" es la defensa contra la regresión: en un pedido de corrección sobre un archivo largo, el modelo tiende a reescribir de más.

**Qué devolvió:** los cuatro arreglos, seis líneas tocadas y nada más. Verificado punto por punto contra la versión anterior: `calcularAsignado` con `pendiente <= 1`, `cerrarPrecision` en centavos enteros, el campo `ultimaInteraccion` en el deslizador con la gracia de 600 ms, y la guarda `max > 0` en el desplazamiento.

---

## Conversación completa

Una sola conversación de Gemini Canvas, sin reiniciar el hilo. El artefacto final tiene 1318 líneas en un archivo, con ocho minijuegos y sin dependencias externas.
