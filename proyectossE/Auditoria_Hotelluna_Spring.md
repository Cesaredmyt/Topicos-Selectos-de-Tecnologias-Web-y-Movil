# Auditoría — Hotel Luna (Spring)

* Diaz Maldonado Cesar Enrique
* Medina Ayala José Antonio

---

## 1. Qué se encontró en general

El proyecto usa Spring, un framework que ya trae su propia forma de recibir peticiones y de asegurar transacciones. Aun así, el código construye por su cuenta un "controlador único" casero (`FrontControllerServlet`) que compite con el mecanismo normal de Spring, y ese controlador casero en realidad no revisa nada porque su lista de filtros siempre está vacía.

| Archivo | Qué se observa | Por qué es un problema |
|---|---|---|
| `LoginCtrl.java` | La consulta de login se arma pegando directamente el correo y la clave (`"...correo='"+correo+"'..."`) | Permite inyección SQL: alguien puede alterar la consulta para entrar sin la clave real. |
| `LoginCtrl.java` | Si el rol es `gerente`, se agrega una cookie `admin_bypass` | Cualquiera con esa cookie puede pasar por administrador sin volver a autenticarse. |
| `FrontControllerServlet.java` | Existe un servlet que dice ser el punto único de entrada (`@WebServlet("/*")`), pero su lista `cadena` nunca se llena | No filtra nada en realidad; es una pieza que aparenta seguridad pero no hace nada. Además compite con las rutas normales de Spring (`ReservaCtrl` también atiende `/reservar`), así que no está claro cuál de los dos responde. |
| `ReservaCtrl.java` | El precio se decide con una cadena de `if` (tipo de cuarto, tarifa) y el pago con tarjeta llama a un banco externo armando un texto XML a mano | Cada vez que se agregue un tipo de cuarto o tarifa hay que tocar este mismo bloque de código, y es fácil equivocarse. |
| `ReservaCtrl.java` | Al reservar se hacen tres escrituras seguidas (reserva, habitación, spa de cortesía) sin que sea una sola operación, y no hay control de doble clic | Un doble clic puede generar una reserva o un cobro duplicado. |
| `FacturaSql.java` | Arma el HTML de la factura a mano y hace una consulta nueva por cada reserva dentro de un ciclo | Mezcla la presentación con el acceso a datos, y es lento si hay muchas reservas. |
| `CheckinCtrl.java` | La consulta de check-in pega directamente el número de reserva recibido | Mismo riesgo de inyección SQL que en el login. |
| `application.properties` | La contraseña de la base de datos está escrita directamente en el archivo | Cualquiera que vea el proyecto ve también la contraseña real. |
| `docs/ER.txt` y `docs/README_ARQ.txt` | Describen un sistema de inventario de piezas (SKU, racks), no un hotel | La documentación no corresponde al código real y además dice que "ya existe" `@Transactional` en todos los servicios, cosa que no es cierta en `ReservaCtrl`. |

### ¿Se corrige poco a poco o se rehace?

Conviene **rehacer la capa de entrada y de acceso a datos desde cero**. El motivo principal es que el proyecto tiene dos mecanismos de entrada compitiendo entre sí (el servlet casero y el propio Spring): antes de corregir cualquier detalle hay que decidir cuál de los dos se queda, y esa decisión ya implica rehacer esa parte. Las reglas de precio y el flujo general de reserva sí se pueden reutilizar como referencia.

---

## 2. Misión 1 — El problema no es un solo patrón

| # | Problema encontrado | Dónde ocurre | Patrón que lo resuelve | Por qué ese y no el que se confunde | Cuándo no usarlo |
|---|---|---|---|---|---|
| 1 | Existen dos formas de recibir peticiones al mismo tiempo: un servlet casero que no filtra nada, y las rutas normales de Spring | Políticas transversales | Front Controller + Chain of Responsibility (usando el mecanismo real de Spring, no uno duplicado a mano) | No es un problema de Decorator: aquí no hace falta "envolver" nada, hace falta un único camino de entrada que si algo falla, corte la petición | Si el sistema tuviera una sola ruta fija, no se justificaría construir esta cadena |
| 2 | El precio se decide con `if` según tipo de cuarto y tarifa, y el cobro con tarjeta llama a un banco externo con su propio formato | Aplicación/dominio + Integración | Strategy + Adapter | Factory decide qué objeto crear; aquí el huésped ya eligió el tipo de cuarto, lo que cambia es el cálculo del precio y la forma de cobrar | Si solo existiera pago en mostrador, no haría falta esta separación |
| 3 | `FacturaSql` arma HTML a mano y hace una consulta nueva por cada reserva en un ciclo | Presentación + Datos | Repository + Service Layer | Facade simplificaría llamadas a un subsistema propio, pero el problema real es que la factura no debería tener SQL mezclado con HTML | Si fuera una consulta simple usada una sola vez, no se justifica una capa completa |
| 4 | Reservar hace tres escrituras seguidas sin ser una sola operación, y no evita el doble clic | Aplicación/dominio + Datos | Idempotencia + Unit of Work | La reserva, la habitación y el spa deben guardarse juntos o no guardarse; arreglar solo una de las consultas no resuelve el doble cobro | Si la reserva fuera un solo paso garantizado por el proveedor de pago, no haría falta |
| 5 | El check-in arma la consulta SQL pegando directamente el número de reserva recibido | Datos | Repository (acceso a datos separado y con entradas limpiadas) | No es un problema de patrón de presentación; es que el acceso a datos no valida lo que recibe | En un script de una sola vez sin usuarios reales, sería aceptable |
| 6 | La contraseña de la base de datos está escrita directamente en `application.properties` | Datos + Integración | Externalizar configuración (variables de entorno) | Juntar todo en un archivo de constantes no saca la contraseña real del código fuente | En una prueba personal sin datos sensibles, es aceptable dejarlo así |

---

## 3. Misión 2 — El camino de "reservar una habitación"

### Lo que hace el código

`ReservaCtrl` no revisa sesión antes de crear la reserva, y como existen dos entradas posibles (`FrontControllerServlet` y la ruta normal de Spring), no queda claro cuál realmente atiende la petición.

### Camino que debería seguir

| Paso | Qué pasa | Tipo de componente | Patrón |
|---|---|---|---|
| 1 | El huésped llena el formulario de reserva y presiona "Reservar" | Pantalla | — |
| 2 | Se envía como `POST /reservar` | Petición HTTP | — |
| 3 | La petición entra por un único punto de recepción del sistema | Enrutador de entrada | Front Controller |
| 4 | Se revisa, en orden: sesión válida, disponibilidad de habitación, y se registra el intento | Filtros previos | Chain of Responsibility |
| 5 | El controlador recibe los datos ya validados y llama al caso de uso | Controlador | — |
| 6 | Se ejecuta `reservarHabitacion(huesped, tipo, tarifa, pago)` | Caso de uso | Service Layer |
| 7 | Se calcula el precio según tipo y tarifa, y se elige cómo cobrar | Selector de cobro | Strategy |
| 8 | Se revisa una clave de esta operación para que un doble clic no duplique la reserva | Verificación previa | Idempotencia |
| 9 | Se traduce el cobro al formato del banco externo y su respuesta a un formato interno | Conector externo | Adapter |
| 10 | El banco procesa el cobro y responde | Servicio externo | — |
| 11 | Si el cobro fue exitoso, se guardan la reserva y la habitación ocupada en una sola operación | Persistencia | Repository + Unit of Work |
| 12 | Se avisa a quien deba enterarse (correo al huésped, aviso a recepción) | Notificación | Observer |
| 13 | Si el banco tarda o falla seguido, se deja de insistir sin detener el resto del sistema | Protección externa | Timeout + Circuit Breaker |
| 14 | Se confirma la reserva al huésped | Salida | — |

---

## 4. Misión 3 — Revisiones que se repiten en cada archivo

**a) Por qué conviene un solo punto de entrada:** aquí el problema es más grave que "repetir código": hay dos entradas distintas (el servlet casero y Spring) y ninguna revisa realmente la sesión antes de reservar. Un único punto de entrada evita esta confusión, porque toda petición pasaría siempre por el mismo camino.

**b) Qué ya trae el framework:** Spring ya incluye interceptores y filtros (y también Spring Security, mencionado en la propia documentación del proyecto) pensados exactamente para revisar sesión, permisos y registrar peticiones sin tener que programar un servlet casero. No hace falta reinventar esa parte.

**c) Orden correcto:** primero autenticación, después revisar disponibilidad (cupo de habitaciones), después bitácora, y al final la reserva y el cobro. Cobrar antes de confirmar la identidad del huésped es un error grave porque, si la autenticación falla después, ya se hizo un cargo real sin poder relacionarlo con certeza a la persona correcta.

**d) Qué no debe ir en esos filtros:** el cálculo del precio, la elección del banco o el armado de la factura son parte del caso de uso de reservar, no de una revisión general que corre para todas las peticiones. Meter esas reglas en los filtros los vuelve difíciles de reutilizar para otras acciones del hotel (como el check-in o el spa).
