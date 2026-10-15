# Trabajo Práctico Final — Madori

**Diseñador de interiores 2D y 3D para viviendas y comercios**, hecho entre **Silvert Quispe** ([@SilvertK164](https://github.com/SilvertK164)) y **Victoria Villalba** ([@vivillalbatevez](https://github.com/vivillalbatevez)). El trabajo es de los dos.

El código no se copió acá: vive en su propio repositorio, que es el entregable.

| | |
| --- | --- |
| **Repositorio** | https://github.com/SilvertK164/interiores-3d (rama `main`) |
| **Aplicación en línea** | https://madori-web-opal.vercel.app |
| **API** | https://madori-api.vercel.app |
| **Videos** | Teaser (30 s): _[enlace]_ · Explicativo (3 min): _[enlace]_ |

## Cómo se lee

El README del repositorio explica qué hace, el stack, la arquitectura y cómo correrlo. La evidencia del proceso está en `docs/`: `ESTADO.md` (decisiones, hecho y pendiente), `docs/superpowers/specs/` y `docs/superpowers/plans/` (un spec y un plan por cada tema, aprobados por los dos antes de implementar).

## Qué construimos

Plantillas de vivienda y comercio, editor 2D y 3D con Pascal, niveles y escaleras, estantes y gabinetes paramétricos, exportación del plano en PDF (A4/A3, escalas 1:25, 1:50 y 1:100, modo papel o color, cajetín con logotipo y hoja isométrica por planta), imagen 3D en color o blanco y negro, y **Escena desde una foto**: se sube la foto de un ambiente, Gemini propone el plano con sus muebles y se revisa en una vista previa antes de crear la escena en 3D con la cámara donde se tomó la foto.

## Para probarla

1. Entrar a la web y crear una cuenta (no hay que confirmar el correo).
2. Crear un proyecto desde una plantilla, mover algo y recargar: el guardado es automático.
3. **Más → Exportar plano** y, en Mis proyectos, **Escena desde una foto → Probar con un ejemplo**.

## Decisiones que tomamos

- **Apoyarnos en Pascal Editor** (MIT) en vez de escribir un motor 3D: el esfuerzo fue integrarlo y ocultar su interfaz detrás de un puente propio, para que solo una carpeta del código lo importe.
- **Contrato primero:** esquemas en zod y un `openapi.yaml` que una prueba compara con el código.
- **Supabase con RLS** para datos y cuentas, API propia en Fastify, todo desplegado en Vercel.
- **Gemini solo desde el servidor:** la clave nunca llega al navegador, con tope de 10 análisis por hora por persona y un acomodador por código que corrige lo que el modelo devuelve.
- **Flujo del curso:** spec → plan → TDD con la prueba vista en rojo → revisión → verificación con salida real. Más de 1900 pruebas automáticas.
- **Uso de IA para ahorrar:** agentes distintos por tema independiente, con un modelo liviano para codificar y uno más capaz para revisar.

## Qué salió mal y cómo lo corregimos

- **El análisis de fotos no tenía sentido al principio:** Gemini daba posiciones sobre la imagen con perspectiva y todos los giros en 0°. Pasamos a pedirle un plano visto desde arriba en metros, y a corregirlo con código.
- **Límites de la clave gratuita de Gemini:** 20 peticiones por día por modelo. Las pruebas lo agotaron; agregamos modelos de respaldo, un ejemplo sin servicio y una vista previa para no depender de la demo en vivo.
- **Integración con el TP3 de Victoria:** su servicio en Python era un buen punto de partida, pero dependía de un servidor local; lo reescribimos dentro de nuestra API.
- **Gasto de tokens:** varias tareas se alargaron más de lo razonable; después acotamos cada encargo y cortamos las iteraciones.

## Pendientes

- Confirmar la licencia de la foto de ejemplo del salón.
- Probar el análisis con más fotos reales.

## Herramientas

Claude Code (agentes por tema), Pascal Editor, Next, Fastify, Supabase, Vercel, Gemini, Vitest, Playwright, HyperFrames (videos).
