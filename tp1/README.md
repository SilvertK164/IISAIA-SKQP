# TP 1 — PagoYa: el recibo que se paga por sorteo

Un portal de pago de servicios donde el usuario no elige cómo paga. Una ruleta decide qué mecánica le toca y cuánto puede abonar en ese turno, y hay que atravesarla tantas veces como haga falta hasta cubrir el recibo. Funciona de punta a punta y usarlo es una tortura, que era la idea.

## Cómo se ejecuta

Doble click en `index.html`. Un solo archivo, sin dependencias. Una partida completa lleva entre diez y quince minutos.

## Qué me propuse construir

Una bad UI que no fuera una sola broma repetida. El género tiende al gag suelto —un slider imposible, un botón que escapa— y eso se agota en diez segundos. Quería que la frustración tuviera estructura: que el usuario entendiera el sistema, lo aceptara, y aun así no pudiera salir rápido.

La ruleta resolvió eso. Deja de ser una colección de minijuegos pegados y pasa a ser un solo sistema con una regla: vos no elegís cómo pagás. Cada mecánica acredita apenas una parte del monto, así que la variedad es necesaria y no decorativa, y hay que pasar por ocho comportamientos distintos para cubrir los 87.40.

La otra mitad del trabajo fue el envoltorio. La interfaz se ve confiada y bien diseñada —neobrutalismo, tipografía cuidada, animación coherente— y el copy es cortés todo el tiempo. La crueldad está solo en la mecánica y en los cargos. Un portal feo no engaña a nadie; uno prolijo que te cobra 1.50 por consultar tu propio importe, sí.

Salió en cinco prompts, en una sola conversación de Gemini Canvas.

## Decisiones que tomé yo

**La ruleta como organizador, no como minijuego.** Es la decisión de la que cuelga todo el resto. Pedida a secas, una bad UI de pago sale como un formulario con un control roto. La ruleta convierte el artefacto en una máquina de estados con turnos, y hace que agregar una mecánica nueva no requiera rediseñar el flujo.

**El contrato de minijuego, antes del primer minijuego.** Los ocho exponen `montar / update / desmontar` y cierran llamando a `finalizar(montoLogrado)`, única vía por la que se acredita algo. Ninguno toca `saldoPendiente` ni escribe en el DOM fuera de su contenedor. Nombré esto en el prompt inicial, no después, y ahí está la diferencia: siete de los ocho minijuegos entraron en prompts posteriores sin romper nada del flujo. Si el contrato hubiera aparecido en el prompt tres, los primeros dos ya estarían escritos en contra.

**Un solo `requestAnimationFrame` para toda la app.** El default del modelo es darle su propio loop o su propio `setInterval` a cada mecánica. Anda, hasta que cambiás de turno y quedan timers corriendo sobre elementos que ya no existen. Un bucle global que llama al `update(dt)` del minijuego activo elimina la categoría entera de bug.

**Un solo `state` y una sola función que lo muta.** Todas las mutaciones pasan por `aplicar(accion)`; `render()` redibuja leyendo el estado y no lo toca. Es más ceremonia de la que pide un artefacto de este tamaño, y se pagó sola: el bug que rompía el juego lo encontré leyendo un `switch`, no rastreando ocho módulos.

**Especificar un minijuego y no ocho.** Describir los ocho en el prompt inicial habría dado un prompt imposible de auditar. Pedí la Cinta transportadora completa como referencia del contrato y dejé que los demás salieran por analogía.

**El monto y el cierre por encima de las mecánicas.** Cuando el modelo me ofreció una lista de agregados, casi todo eran minijuegos nuevos. Los descarté: lo flaco eran los extremos del recorrido. Un monto que aparece solo no genera ninguna relación con el usuario, y una fase final que es solo un comprobante desperdicia el único momento de alivio que tiene. De ahí salieron el desglose con cargo por consulta, la mora que corre mientras raspás, la propina obligatoria y la calificación donde solo la estrella 5 está habilitada.

**Los cargos, no el reloj.** La presión del artefacto es económica, no temporal. Consultar el importe cuesta 1.50, fallar un turno 0.50, ver de nuevo el código de identidad 0.30, y la mora suma 0.50 cada 45 segundos. Un reloj molesta; un saldo que sube mientras intentás bajarlo es otra cosa. Y produce un comprobante final donde se lee exactamente cuánto te cobraron por pagar.

**Dos sectores de la ruleta no se juegan.** "Cambio de monto" recalcula el saldo un 10% para arriba mientras mirás un contador, y "Sin método disponible" cierra el turno sin dejarte intervenir. Son la cuarta parte de los giros, y fue deliberado: la impotencia es peor que la dificultad, y ninguna mecánica la transmite tan bien como una pantalla donde no hay nada que tocar.

**Lo que descarté.** Arranqué pensando en una paginación binaria y un captcha de color de dificultad creciente: el primero es una broma de un segundo y no ejercita nada, y el segundo era el más caro de implementar y el que menos aportaba una vez que existía la ruleta. Del inventario que propuso el modelo dejé afuera el captcha de recibos y la cola de espera. Y por alcance quedaron fuera un laberinto de navegación y una honda con física de proyectil: los dos entraban en el contrato sin tocar nada más —que era el punto de haberlo definido así— pero cada uno agrega un motor de colisiones y el archivo ya tiene dos.

## Sobre delegar la generación de ideas

Dos de los cinco prompts fueron abiertos: le pedí al modelo qué agregaría y después redirigí. Vale decirlo de frente, porque varias de las mejores piezas del artefacto no son mías: el cargo por consulta de importe, la comisión por fallo y la calificación de una sola estrella salieron de ahí.

Lo que aprendí es dónde está el límite de ese modo de trabajo. El modelo genera bien y jerarquiza mal: cuando le pedí que eligiera tres de su propia lista, eligió las tres más baratas de implementar, no las que sostenían el concepto. La comisión por fallo —la única que cambia la naturaleza del juego, porque convierte una tarea difícil en una que puede empeorar— estaba en la lista pero no en su terna. Delegar la generación sale barato; delegar el criterio, no.

## Qué salió mal y cómo lo corregí

**El comprobante era inalcanzable.** El bug más caro y el más instructivo. Había dos reglas: un "cargo por saldo menor" que redondea al mínimo operativo cualquier saldo inferior a 1.00, y un monto por turno asignado entre el 20% y el 50% del pendiente, salvo cuando el pendiente ya es muy chico. Cada una es razonable leída sola. Juntas cierran la única salida: el saldo queda clavado en 1.00, se le asigna como mucho 0.50 por turno, y el cargo lo devuelve a 1.00 para siempre. `saldoPendiente` nunca llega a cero.

La consecuencia era que toda la fase final —comprobante, propina, calificación, encuesta, descarga— era código muerto. Nunca la había visto, ni jugando ni testeando, porque nunca llegué. Se arregló con una condición: el turno asigna el total cuando el pendiente es menor o igual a 1.00. El cargo sigue existiendo y sigue siendo abusivo, pero deja una puerta.

El modelo no se equivocó: ejecutó exactamente lo que le pedí. Una de las dos reglas la propuso él (el redondeo a 0.10 del monto asignado) y la otra la acepté yo sin cruzarlas. La contradicción no vivía en ninguna de las dos, sino entre ellas, y por eso era invisible desde adentro de cada una.

**Un segundo camino cerrado, por la misma causa.** En el "Deslizador fugitivo" la perilla vuelve sola a cero al soltarla, y confirmar requiere click en un botón aparte. Entre soltar y confirmar pasan unos cientos de milisegundos y la tolerancia es de centavos: la rama del acierto exacto era estadísticamente inalcanzable. Le puse una gracia de 600 ms antes de que arranque la deriva, el mismo patrón que ya usaba otro minijuego, así que además quedó más consistente.

**Punto flotante en el crédito parcial.** Los juegos de precisión acreditan lo alcanzado redondeado hacia abajo con `Math.floor(valor * 10) / 10`. Con 0.70, la multiplicación da 6.999... y el usuario cobraba 0.60. Pasado a centavos enteros, resuelto. Es el bug que no se ve leyendo, porque el código *parece* correcto.

**`NaN` en pantalla.** El minijuego de scroll calcula el monto como proporción del recorrido disponible. Si ese recorrido es cero, la división da `NaN` y el contador muestra "$ NaN". Nunca se disparó en mi navegador, pero es una división sin guarda esperando una ventana chica.

**Lo que verifiqué después de destrabar el final.** Me quedaba la duda de si la economía converge: entre la mora, la comisión por fallo, el ajuste del 10% y el cargo por saldo menor hay cuatro fuentes que empujan el saldo hacia arriba. Lo simulé en vez de estimarlo. Un usuario razonable termina en 17 a 28 turnos; el punto de quiebre está cerca del 75% de fallos, y recién por encima del 80% el saldo diverge y la partida no cierra nunca. Es el equilibrio que buscaba: hostil, pero no imposible. Si fuera imposible dejaría de ser funcional, y la consigna pide las dos cosas.

## Prompts

El registro completo está en [prompts.md](prompts.md). Los que más pesaron son el primero, que fija la arquitectura entera y explica por qué el resto pudo crecer sin romperse, y el último, que corrige los cuatro bugs del Review.

## Herramientas

El artefacto se construyó íntegramente en una sola conversación de Gemini Canvas, como pide la consigna. Usé Claude por fuera de esa conversación para revisar el código y para simular la convergencia económica; los cuatro bugs del último prompt salieron de esa revisión.
