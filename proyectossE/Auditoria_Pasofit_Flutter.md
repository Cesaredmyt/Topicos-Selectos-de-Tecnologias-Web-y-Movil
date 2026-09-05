# Auditoría — PasoFit (Flutter)

* Diaz Maldonado Cesar Enrique
* Medina Ayala José Antonio

---

## 1. Qué se encontró en general

Este proyecto es distinto a los demás porque es una aplicación de celular (Flutter), no un servidor web. Aun así, tiene los mismos problemas de fondo: revisiones de acceso que no revisan nada de verdad, dos formas distintas de avisar cambios dentro de la app al mismo tiempo, y SQL escrito directamente dentro de las pantallas.

| Archivo | Qué se observa | Por qué es un problema |
|---|---|---|
| `pantallas/inicio.dart` | Se compara si el usuario trae la marca de sesión correcta, pero el bloque que debería bloquear el acceso está vacío (`// a veces entra igual`) | La revisión existe en el código, pero no hace nada: cualquiera entra a la pantalla de inicio sin haber iniciado sesión. |
| `pantallas/login.dart` | Se revisa si la respuesta del servidor "contiene" la palabra `ok` o `id` como texto | Es una forma muy débil de confirmar que el login fue exitoso; cualquier respuesta que por casualidad tenga esas letras se toma como válida. |
| `estado/oyentes.dart` | Existen `RelojBus` y `PesoBus`, listas caseras de funciones para avisar cambios | Ya existe en el proyecto (`RelojCubit`) una forma correcta de hacer esto mismo con el paquete `flutter_bloc`, pero nunca se usa; en su lugar, cada pantalla usa la versión casera. |
| `pantallas/cronometro.dart` | Usa `RelojBus` en vez del `RelojCubit` que ya está en el proyecto | Hay dos formas de resolver lo mismo (avisar que pasó un segundo) y solo una de ellas realmente se usa, dejando a la otra como código muerto. |
| `pantallas/rutina.dart` | El costo en calorías se decide con una cadena de `if` según el tipo de rutina, y luego se hacen dos escrituras seguidas en la base local | No hay control de doble clic ni una sola operación que agrupe ambos guardados. |
| `pantallas/rutina.dart` y `pantallas/peso.dart` | El SQL se escribe directamente dentro de la pantalla (`State`) | La propia documentación del proyecto dice que el SQL debería vivir en un solo lugar dedicado a los datos, no en la pantalla. |
| `pantallas/inicio.dart` | Se hacen 8 llamadas seguidas a distintos servicios antes de mostrar la pantalla, sin límite de espera | Si uno de esos servicios tarda, toda la pantalla de inicio tarda con él. |
| `pantallas/rutina_old.dart` | Un archivo completo que solo dice "usar rutina.dart" | Es una pantalla vieja que ya no se usa pero se dejó en el proyecto. |
| `docs/ARQUITECTURA.md` | Describe una aplicación bancaria, no una app de ejercicio | La documentación no corresponde a lo que la app realmente hace. |

### ¿Se corrige poco a poco o se rehace?

Conviene **rehacer la forma en que la app guarda datos y avisa cambios**, ya que hoy conviven dos sistemas distintos para lo mismo (la lista casera y `flutter_bloc`), y ninguna pantalla revisa realmente si el usuario inició sesión. Las pantallas visuales (los widgets) sí se pueden conservar casi tal cual, solo cambiando de dónde obtienen y a dónde avisan los datos.

---

## 2. Misión 1 — El problema no es un solo patrón

| # | Problema encontrado | Dónde ocurre | Patrón que lo resuelve | Por qué ese y no el que se confunde | Cuándo no usarlo |
|---|---|---|---|---|---|
| 1 | La revisión de sesión en la pantalla de inicio no bloquea nada aunque el usuario no haya iniciado sesión | Políticas transversales | Chain of Responsibility (una revisión que si falla, sí corta el paso a la pantalla) | No es un problema de Decorator: aquí no se necesita "envolver" la pantalla, se necesita poder impedir que se muestre | Si la app no manejara información privada, no sería tan grave omitir esta revisión |
| 2 | El costo en calorías se decide con una cadena de `if` según el tipo de rutina | Aplicación/dominio | Strategy | Factory serviría para decidir qué objeto crear; aquí el usuario ya eligió el tipo de rutina, lo que cambia es el cálculo, no el tipo de objeto | Si solo existiera un tipo de rutina, no haría falta esta separación |
| 3 | Hay dos formas de avisar cambios dentro de la app: una lista casera (`RelojBus`, `PesoBus`) y un `RelojCubit` de `flutter_bloc` que no se usa | Aplicación (estado de la app) | Observer (usando un solo mecanismo, el de `flutter_bloc`, no dos en paralelo) | No se trata de elegir entre Observer y otro patrón, sino de dejar de tener dos implementaciones del mismo patrón compitiendo entre sí | Si la app fuera de una sola pantalla sin necesidad de avisar cambios a otras, no haría falta ningún mecanismo de este tipo |
| 4 | Guardar una rutina hace dos escrituras seguidas en la base local sin ser una sola operación | Datos | Unit of Work + Idempotencia | No basta con revisar los datos antes de guardar; hace falta que ambas escrituras ocurran juntas o ninguna, y evitar el doble guardado si se toca el botón dos veces | Si solo se guardara un dato a la vez, no haría falta agrupar nada |
| 5 | La pantalla de inicio hace 8 llamadas seguidas a distintos servicios sin límite de espera | Integración | Timeout + Circuit Breaker | Insistir con un servicio caído no ayuda si sigue caído; hace falta dejar de esperar y no bloquear el resto de la pantalla | Si esos servicios fueran siempre rápidos y confiables, esta protección no haría falta |
| 6 | El SQL está escrito directamente dentro de las pantallas (`rutina.dart`, `peso.dart`) | Datos | Repository | La pantalla no debería saber cómo se guarda un dato, solo qué dato mostrar; separar esto facilita cambiar de base de datos sin tocar cada pantalla | En una prueba muy pequeña, con una sola pantalla, no se justifica crear esta capa |

---

## 3. Misión 2 — El camino de "guardar una rutina"

Como esta app es un cliente móvil (no incluye el servidor con el que se conecta), el camino se traza desde que el usuario toca el botón hasta que el dato queda guardado y se avisa dentro de la misma app. La parte de "filtros de entrada" descrita en otros proyectos correspondería al servidor que la app consume, que no forma parte de este proyecto.

| Paso | Qué pasa | Tipo de componente | Patrón |
|---|---|---|---|
| 1 | El usuario elige el tipo de rutina y presiona "Iniciar" | Pantalla | — |
| 2 | Antes de continuar, se revisa que haya una sesión activa; si no la hay, se detiene aquí | Revisión previa | Chain of Responsibility |
| 3 | Se llama a la función que maneja la lógica de guardar la rutina | Controlador de la pantalla | — |
| 4 | Se calcula cuántas calorías corresponden según el tipo de rutina elegido | Selector de cálculo | Strategy |
| 5 | Se revisa que esta misma rutina no se haya guardado ya por un doble toque del botón | Verificación previa | Idempotencia |
| 6 | Se guardan la sesión de ejercicio y el logro pendiente como una sola operación | Persistencia local | Repository + Unit of Work |
| 7 | Se avisa a otras pantallas que hay un nuevo dato disponible | Notificación interna | Observer (usando `flutter_bloc`, no la lista casera) |
| 8 | Se envía un aviso al servidor (por ejemplo, para mandar un correo) | Conector externo | Adapter |
| 9 | Si el servidor tarda o no responde, se deja de esperar sin bloquear la pantalla | Protección externa | Timeout |
| 10 | Se muestra en pantalla que la rutina quedó guardada | Salida | — |

---

## 4. Misión 3 — Revisiones que se repiten en cada pantalla

**a) Por qué conviene un solo punto de entrada:** en esta app, cada pantalla decide por su cuenta si revisa la sesión o no, y una de ellas (`inicio.dart`) tiene la revisión escrita pero sin efecto. Si todas las pantallas pasaran primero por una sola revisión compartida, no dependería de que cada pantalla se acuerde de hacerlo bien.

**b) Qué ya trae el framework:** Flutter, junto con paquetes como `flutter_bloc` o `Provider`, ya permite centralizar el estado de la app y las revisiones de acceso en un solo lugar (por ejemplo, decidiendo qué pantalla mostrar según si hay sesión o no), en vez de repetir la revisión copiada en cada pantalla.

**c) Orden correcto:** primero revisar que haya sesión activa, después revisar que la acción tenga sentido (por ejemplo, que el tipo de rutina exista), después dejar registro de la acción, y al final guardar el resultado y avisar. Guardar datos antes de confirmar la sesión es un error porque esos datos podrían quedar asociados a nadie, o a la persona equivocada.

**d) Qué no debe ir en esa revisión general:** el cálculo de calorías, la elección de qué mostrar en cada pantalla o el armado de mensajes son responsabilidad de cada pantalla en particular, no de la revisión general de sesión.
