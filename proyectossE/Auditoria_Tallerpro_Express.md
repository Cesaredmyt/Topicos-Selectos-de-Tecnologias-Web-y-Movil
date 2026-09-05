# Auditoría — TallerPro (Express)

* Diaz Maldonado Cesar Enrique
* Medina Ayala José Antonio

---

## 1. Qué se encontró en general

Este proyecto es el único de los revisados que sí tiene, en un lugar, una cadena de responsabilidad bien armada: `cadena/Handler.js` define `AuthHandler` y `BitacoraHandler`, y se usan en orden correcto para la ruta `/nueva` (revisar sesión, luego dejar registro, y hasta el final cobrar). El problema es que esa cadena no se usa en el resto de las rutas, y la propia revisión de sesión dentro de ella tiene una puerta trasera.

| Archivo | Qué se observa | Por qué es un problema |
|---|---|---|
| `app.js` (ruta `/login`) | La consulta se arma pegando directamente el correo y la clave | Permite inyección SQL: alguien puede alterar la consulta para entrar sin la clave real. |
| `app.js` (ruta `/login`) | Si el rol es `jefe`, se agrega una cookie `admin_bypass` | Cualquiera con esa cookie entra como jefe del taller sin volver a iniciar sesión. |
| `cadena/Handler.js` (`AuthHandler`) | Aunque revisa la sesión correctamente, también acepta la cookie `admin_bypass` para saltarse la revisión | La misma puerta trasera del login se repite aquí, dentro de la única parte del proyecto que sí está bien organizada. |
| `app.js` (ruta `/`) | Vuelve a escribir a mano la misma revisión de sesión que ya hace `AuthHandler`, en vez de reusarlo | La misma revisión existe copiada en dos lugares distintos del mismo archivo. |
| `app.js` (ruta `/`) | Se hacen 8 llamadas seguidas a distintos servicios antes de mostrar la página | Si uno de esos servicios tarda, toda la página principal tarda con él. |
| `cadena/Handler.js` (clase `CobroOrden`) | El costo se decide con `if`/`elif` según el tipo de servicio, y el pago con tarjeta llama a un banco externo armando un texto XML a mano | Agregar un nuevo tipo de servicio obliga a tocar este mismo bloque de código. |
| `cadena/Handler.js` (clase `CobroOrden`) | Se hacen tres escrituras seguidas (orden, refacciones, mecánico ocupado) sin ser una sola operación, y nunca se revisa si el mecánico ya estaba ocupado antes de asignarlo | Puede asignarse el mismo mecánico a dos órdenes al mismo tiempo, y un doble clic puede generar un cobro duplicado. |
| `app.js` (ruta `/entrega`) | La consulta SQL pega directamente el id recibido por la URL | Mismo riesgo de inyección SQL que en el login. |
| `app.js` (ruta `/inventario`) | Se hace una consulta nueva por cada fila dentro de un ciclo | Vuelve la página lenta si hay muchas refacciones registradas. |
| `app.js` | La contraseña de la base de datos y el secreto de las sesiones (`secret: 'password'`) están escritos directamente en el código | Cualquiera que vea el código ve también estos valores reales. |
| `app.old.js` | Es una copia completa y vieja del sistema, dejada "por si acaso" | Aumenta la confusión sobre cuál versión es la que realmente está en uso. |
| `documentacion/SISTEMA.txt` | Describe una clínica dental, no un taller mecánico, y además dice que no se debe usar una cadena de responsabilidad con clases | La documentación no corresponde al código real, y contradice justo la única parte bien hecha del proyecto. |

### ¿Se corrige poco a poco o se rehace?

A diferencia de los demás proyectos, aquí **sí conviene corregir poco a poco**, partiendo de la cadena de responsabilidad que ya existe (`cadena/Handler.js`). El trabajo principal es: quitar la puerta trasera de `AuthHandler`, usar esa misma cadena en todas las rutas (no solo en `/nueva`), y limpiar el resto de las consultas SQL escritas a mano. No hace falta rehacer la arquitectura completa porque ya hay una base organizada de la cual partir.

---

## 2. Misión 1 — El problema no es un solo patrón

| # | Problema encontrado | Dónde ocurre | Patrón que lo resuelve | Por qué ese y no el que se confunde | Cuándo no usarlo |
|---|---|---|---|---|---|
| 1 | La cadena de responsabilidad que revisa sesión solo se usa en `/nueva`; el resto de las rutas repite la revisión a mano o no revisa nada | Políticas transversales | Front Controller + Chain of Responsibility, aplicada a **todas** las rutas | Ya existe la pieza correcta (`AuthHandler`, `BitacoraHandler`); el problema no es construir el patrón, es usarlo de forma consistente en todo el proyecto | Si el sitio tuviera una sola ruta pública sin datos sensibles, no sería tan grave omitir esta cadena |
| 2 | La cookie `admin_bypass` permite saltarse la revisión de `AuthHandler` | Políticas transversales | Quitar el atajo dentro del propio `AuthHandler`, sin excepciones por cookie | No es un problema de qué patrón usar, sino de que el patrón correcto está mal implementado por dentro | — |
| 3 | El costo de la orden se decide con `if`/`elif` según el tipo de servicio, con una llamada externa a un banco | Aplicación/dominio + Integración | Strategy + Adapter | Factory decide qué objeto crear; aquí el cliente ya eligió el tipo de servicio, lo que cambia es el cálculo del costo y la forma de cobrar | Si solo existiera un tipo de servicio, no haría falta esta separación |
| 4 | Al cobrar una orden se hacen tres escrituras seguidas sin ser una sola operación, y nunca se revisa si el mecánico ya estaba ocupado | Aplicación/dominio + Datos | Unit of Work + una revisión de disponibilidad antes de cobrar | No basta con agrupar las escrituras: además falta revisar el cupo (mecánico libre) antes de cobrar, que es un problema distinto | Si solo hubiera un mecánico y nunca se cruzaran citas, esta revisión sería menos urgente |
| 5 | La ruta `/entrega` arma la consulta SQL pegando directamente el id recibido | Datos | Repository (acceso a datos separado y con entradas limpiadas) | El problema no es de presentación, es que el acceso a datos no valida lo que recibe | En un script personal de una sola vez, sería aceptable |
| 6 | La contraseña de la base de datos y el secreto de sesión están escritos directamente en `app.js`, y además existe `app.old.js` como copia completa sin usar | Datos + Integración | Externalizar configuración y eliminar el archivo duplicado que ya no se usa | Guardar una "copia por si acaso" no reemplaza el control de versiones; solo genera confusión sobre qué versión es la real | En una prueba personal, sin necesidad de historial, podría no importar |

---

## 3. Misión 2 — El camino de "abrir y cobrar una orden"

### Lo que hace el código

La ruta `/nueva` sí usa la cadena `AuthHandler → BitacoraHandler → CobroOrden`, en un orden razonable (primero sesión, luego bitácora, al final el cobro). Lo que falta es quitar la puerta trasera de la cookie y agregar el resto de las protecciones (cupo, idempotencia, manejo de fallas del banco).

### Camino que debería seguir

| Paso | Qué pasa | Tipo de componente | Patrón |
|---|---|---|---|
| 1 | El cliente llena el formulario de nueva orden y presiona el botón | Pantalla | — |
| 2 | Se envía como `POST /nueva` | Petición HTTP | — |
| 3 | La petición entra por el único punto de recepción del servidor | Enrutador de entrada | Front Controller |
| 4 | Se revisa, en orden: sesión válida (sin puerta trasera), que el mecánico esté disponible, y se registra el intento | Cadena de filtros | Chain of Responsibility |
| 5 | Se llama al caso de uso de cobrar la orden, sin decidir reglas de negocio en la ruta | Controlador | — |
| 6 | Se ejecuta `cobrarOrden(cliente, auto, tipo, mecanico, pago)` | Caso de uso | Service Layer |
| 7 | Se calcula el costo según el tipo de servicio y se elige cómo cobrar | Selector de cobro | Strategy |
| 8 | Se revisa una clave de esta operación para que un doble clic no duplique el cobro | Verificación previa | Idempotencia |
| 9 | Se traduce el cobro al formato del banco externo y su respuesta a un formato interno | Conector externo | Adapter |
| 10 | El banco procesa el cobro y responde | Servicio externo | — |
| 11 | Si el cobro fue exitoso, se guardan la orden, el uso de refacciones y la ocupación del mecánico en una sola operación | Persistencia | Repository + Unit of Work |
| 12 | Se avisa a quien deba enterarse (correo al cliente, aviso a recepción) | Notificación | Observer |
| 13 | Si el banco tarda o falla seguido, se deja de insistir sin detener el resto del sistema | Protección externa | Timeout + Circuit Breaker |
| 14 | Se confirma la orden al cliente | Salida | — |

---

## 4. Misión 3 — Revisiones que se repiten en cada archivo

**a) Por qué conviene un solo punto de entrada:** en este proyecto ya se ve el beneficio de tenerlo: la ruta `/nueva` usa la cadena y queda ordenada, mientras que la ruta `/` repite la misma revisión a mano por separado. Si todas las rutas usaran la misma cadena, no habría dos versiones de la revisión de sesión que mantener por separado.

**b) Qué ya trae el framework:** Express ya permite declarar middleware compartido con `app.use(...)`, que se aplicaría a todas las rutas automáticamente. En este proyecto ya existe una versión casera de esa idea (`cadena/Handler.js`), pero solo se conecta a una ruta; se podría usar `app.use` para aplicarla a todas sin repetir código.

**c) Orden correcto:** primero autenticación, después revisar cupo (que el mecánico esté libre), después bitácora, y al final el cobro. En este proyecto el orden dentro de `cadena/Handler.js` ya respeta autenticación antes que cobro, lo cual es correcto; lo que falta es agregar el paso de cupo antes de cobrar, porque hoy se asigna el mecánico sin revisar si ya estaba ocupado.

**d) Qué no debe ir en esa cadena de filtros:** el cálculo del costo del servicio, la elección del banco o el armado del mensaje de confirmación son parte del caso de uso de cobrar una orden, no de la revisión general de sesión y bitácora.
