# Auditoría — Sabores (Laravel)

* Diaz Maldonado Cesar Enrique
* Medina Ayala José Antonio

---

## 1. Qué se encontró en general

Laravel ya trae un sistema de middleware (filtros que se pueden aplicar a las rutas), y el proyecto incluso tiene dos middleware ya escritos (`Authenticate.php`, `SoloRepartidor.php`). El problema es que ninguna ruta en `routes/web.php` los usa. Además existe una clase `FrontController.php` casera que tampoco está conectada a nada.

| Archivo | Qué se observa | Por qué es un problema |
|---|---|---|
| `LoginController.php` | La consulta de login se arma pegando directamente el correo y la clave | Permite inyección SQL: alguien puede alterar la consulta para entrar sin la clave real. |
| `LoginController.php` | Si el rol es `admin`, se agrega una cookie `admin_bypass` | Cualquiera con esa cookie puede pasar por administrador sin volver a iniciar sesión. |
| `routes/web.php` | Ninguna ruta usa `->middleware('auth')` ni `->middleware('repartidor')`, aunque esos middleware ya existen en el proyecto | Los filtros de seguridad ya están escritos pero no protegen nada porque no se conectaron a las rutas. |
| `RepartoController.php` | Vuelve a escribir a mano la misma revisión de sesión que ya debería hacer el middleware, y también acepta la cookie `admin_bypass` | Se repite la misma revisión en dos lugares distintos (el middleware sin usar, y esta copia manual), y la copia manual tiene la misma puerta trasera. |
| `Http/Controllers/FrontController.php` | Es una clase que dice manejar las rutas, pero no está conectada a `routes/web.php` | Es una pieza duplicada del enrutador que ya trae Laravel, y no hace nada en la práctica. |
| `PedidoController.php` | El costo de envío se decide con `if`/`elif`, y el pago con tarjeta arma un texto XML a mano con el número de tarjeta y llama a una pasarela externa | Es difícil agregar una nueva forma de envío o de pago sin tocar este mismo bloque grande. |
| `PedidoController.php` | Al guardar un pedido se hacen varias escrituras seguidas (pedido, detalle, cola de reparto) sin ser una sola operación, y no hay control de doble clic | Un doble clic puede generar un pedido o un cobro duplicado. |
| `ResenaController.php` y `MenuController.php` (método `verViejo`) | Arman las consultas SQL pegando directamente lo que envía el usuario | Mismo riesgo de inyección SQL que en el login. |
| `.env` | La contraseña de la base de datos es `password`, escrita directamente en el archivo | Es una contraseña débil y visible para cualquiera que tenga el proyecto. |
| `documentacion/LEAME_ARQUITECTO.txt` | Describe un sistema de censo de ganado, no una app de comida | La documentación no corresponde al código real, y además asegura que "todas las rutas ya tienen middleware", lo cual es falso. |

### ¿Se corrige poco a poco o se rehace?

Conviene **conectar primero los middleware que ya existen a todas las rutas que lo necesitan**, y de ahí revisar cada controlador para quitar el SQL escrito a mano. A diferencia de otros proyectos, aquí las piezas correctas (`Authenticate`, `SoloRepartidor`) ya están construidas; el problema es que nunca se usaron. Por eso no hace falta rehacer todo desde cero, pero sí revisar con cuidado cada controlador porque el patrón de "SQL pegado directamente" se repite en varios archivos distintos.

---

## 2. Misión 1 — El problema no es un solo patrón

| # | Problema encontrado | Dónde ocurre | Patrón que lo resuelve | Por qué ese y no el que se confunde | Cuándo no usarlo |
|---|---|---|---|---|---|
| 1 | Ninguna ruta usa los middleware ya existentes, así que no hay una revisión real antes de llegar a cada acción | Políticas transversales | Front Controller + Chain of Responsibility (usando el middleware que Laravel ya trae) | No hace falta programar nada nuevo: hace falta **conectar** lo que ya existe; por eso no aplica construir otro Front Controller casero como el que ya sobra en el proyecto | Si solo hubiera páginas públicas sin datos sensibles, no sería tan grave |
| 2 | `RepartoController` repite a mano la misma revisión que debería hacer el middleware, incluyendo la puerta trasera de la cookie | Políticas transversales | Mismo patrón anterior, aplicado en un solo lugar central | Duplicar la revisión en cada controlador multiplica el riesgo de que una copia quede mal hecha, como pasó aquí | — |
| 3 | El costo de envío y la forma de pago se deciden con `if`/`elif`, con una llamada externa a una pasarela de pago | Aplicación/dominio + Integración | Strategy + Adapter | Factory decide qué objeto crear; aquí el cliente ya eligió la forma de envío y de pago, lo que cambia es el cálculo y la forma de cobrar | Si solo existiera pago en efectivo, no haría falta esta separación |
| 4 | Guardar un pedido hace varias escrituras seguidas sin ser una sola operación, y no evita el doble clic | Aplicación/dominio + Datos | Idempotencia + Unit of Work | El pedido, su detalle y la cola de reparto deben guardarse juntos o no guardarse; arreglar solo una consulta no evita el doble pedido | Si el pedido fuera un solo paso garantizado por el proveedor de pago, no haría falta |
| 5 | `ResenaController` y `MenuController::verViejo()` arman las consultas SQL pegando directamente los datos recibidos | Datos | Repository (acceso a datos separado y con entradas limpiadas) | No es un problema de presentación sino de que el acceso a datos no valida lo que recibe | En un script de una sola vez, sin usuarios reales, sería aceptable |
| 6 | La contraseña de la base de datos está escrita como valor débil directamente en `.env` | Datos + Integración | Externalizar configuración con un valor real y seguro | El archivo `.env` ya es el lugar correcto para esto; el problema es el valor débil y compartido, no la falta de un mecanismo | En una prueba personal sin datos sensibles, un valor simple es aceptable |

---

## 3. Misión 2 — El camino de "hacer un pedido"

### Lo que hace el código

`PedidoController::guardar()` no revisa sesión antes de crear el pedido y cobrar, porque la ruta `/pedir` no tiene middleware de autenticación asignado.

### Camino que debería seguir

| Paso | Qué pasa | Tipo de componente | Patrón |
|---|---|---|---|
| 1 | El cliente arma su pedido y presiona "Pedir" | Pantalla | — |
| 2 | Se envía como `POST /pedir` | Petición HTTP | — |
| 3 | La petición entra por el enrutador único de Laravel | Enrutador de entrada | Front Controller |
| 4 | Antes de llegar al controlador, se revisa: sesión válida, que el restaurante siga abierto, y se registra el intento | Middleware | Chain of Responsibility |
| 5 | El controlador recibe los datos ya validados y llama al caso de uso | Controlador | — |
| 6 | Se ejecuta `hacerPedido(cliente, platillos, envio, pago)` | Caso de uso | Service Layer |
| 7 | Se calcula el costo de envío y se elige cómo cobrar | Selector de cobro | Strategy |
| 8 | Se revisa una clave de esta operación para que un doble clic no duplique el pedido | Verificación previa | Idempotencia |
| 9 | Se traduce el cobro al formato de la pasarela externa y su respuesta a un formato interno | Conector externo | Adapter |
| 10 | La pasarela procesa el cobro y responde | Servicio externo | — |
| 11 | Si el cobro fue exitoso, se guardan el pedido, su detalle y la cola de reparto en una sola operación | Persistencia | Repository + Unit of Work |
| 12 | Se avisa a quien deba enterarse (correo al cliente, aviso a cocina, aviso a reparto) | Notificación | Observer |
| 13 | Si la pasarela tarda o falla seguido, se deja de insistir sin detener el resto del sistema | Protección externa | Timeout + Circuit Breaker |
| 14 | Se confirma el pedido al cliente | Salida | — |

---

## 4. Misión 3 — Revisiones que se repiten en cada archivo

**a) Por qué conviene un solo punto de entrada:** en este proyecto la revisión de sesión ya está escrita como middleware, pero como nadie la conectó a las rutas, `RepartoController` tuvo que volver a escribirla a mano. Esto muestra justo el riesgo de no centralizar: la misma revisión termina copiada, y la copia resultó tener además una puerta trasera que el middleware original quizás no tendría.

**b) Qué ya trae el framework:** Laravel ya incluye middleware (`app/Http/Middleware/`) y una forma sencilla de asignarlo a las rutas (`Route::get(...)->middleware('auth')`). En este proyecto esa pieza ya existe, solo falta usarla en `routes/web.php`.

**c) Orden correcto:** primero autenticación, después revisar que el pedido tenga sentido (cupo/disponibilidad del restaurante), después bitácora, y al final el caso de uso de cobrar y guardar. Cobrar antes de confirmar la sesión es grave porque, si la sesión resulta inválida después, ya se hizo un cargo real sin poder asociarlo con certeza a un cliente identificado.

**d) Qué no debe ir en esos filtros:** el cálculo del costo de envío, la elección de la pasarela de pago o el armado del mensaje de confirmación son parte del caso de uso de "hacer un pedido", no de un middleware general que corre para todas las rutas.
