# Auditoría — Cinerex (Django)

* Diaz Maldonado Cesar Enrique 
* Medina Ayala José Antonio

---

## 1. Qué se encontró en general

El proyecto no usa ningún patrón de diseño completo. Todo lo que decide, guarda y muestra está mezclado dentro de las mismas funciones en `taquilla/views.py`. Incluso existe un archivo (`taquilla/cadena.py`) que parece el inicio de una cadena de filtros de seguridad, pero nunca se conecta al proyecto: en `settings.py` está comentado y no se usa.

| Archivo | Qué se observa | Por qué es un problema |
|---|---|---|
| `taquilla/views.py` (función `login`) | La consulta se arma pegando directamente lo que escribe el usuario (`f"...WHERE correo='{u}'..."`) | Alguien puede escribir un correo especial para entrar sin saber la contraseña real (inyección SQL). |
| `taquilla/views.py` (función `login`) | Si el tipo de usuario es `staff`, se le da una cookie `admin_bypass` | Cualquiera que tenga esa cookie entra como administrador después, sin volver a iniciar sesión. |
| `taquilla/views.py` (función `cartelera`) | Revisa la sesión de una forma, la cookie de otra, y si hay `admin_bypass` deja pasar igual | Hay tres formas distintas de "estar logueado" al mismo tiempo, y una de ellas no pide nada. |
| `cinerex/settings.py` | `AuthenticationMiddleware` está comentado a propósito, y la cadena de filtros (`CadenaGoF`) tampoco está activada | Django ya trae su propio sistema de seguridad de sesión, pero aquí se apaga a propósito. |
| `taquilla/views.py` (función `cartelera`) | Se hacen 9 llamadas seguidas a otros servicios (`mods`) antes de mostrar la página | Si uno de esos servicios tarda o no responde, toda la cartelera tarda o se cae con él. |
| `taquilla/templatetags/sql_cine.py` | Arma consultas SQL y las corre desde una etiqueta de plantilla (algo que debería solo mostrar datos) | Mezcla la forma en que se pide la información con la forma en que se muestra; además hace una consulta nueva por cada película dentro de un ciclo. |
| `taquilla/views.py` (función `comprar`) | Se hacen tres inserciones seguidas (boleto, asiento ocupado, snack) sin que sea una sola operación, y no hay control de doble clic | Un doble clic puede generar un boleto duplicado o un cobro doble. |
| `cinerex/settings.py` | La contraseña de la base de datos y la llave secreta del proyecto están escritas directamente en el código | Cualquiera que vea el código ve también la contraseña real. |
| `docs/ARQUITECTURA_OFICIAL.md` y `docs/modelo_datos.txt` | Describen un sistema de logística portuaria (contenedores, grúas), no un cine | La documentación no corresponde a lo que el código realmente hace; no sirve para entender el proyecto. |

### ¿Se corrige poco a poco o se rehace?

Conviene **rehacer la organización del proyecto desde cero**, reutilizando las reglas de negocio que sí se identifican (precios por tipo de función, flujo de compra). La razón es que las fallas de seguridad (login sin protección, cookie de administrador falsa) no son detalles menores que se puedan corregir uno por uno: afectan la base de cómo entra cualquier usuario al sistema, así que conviene reconstruir esa base completa en vez de ir parchando cada acceso.

---

## 2. Misión 1 — El problema no es un solo patrón

| # | Problema encontrado | Dónde ocurre | Patrón que lo resuelve | Por qué ese y no el que se confunde | Cuándo no usarlo |
|---|---|---|---|---|---|
| 1 | No hay un solo lugar que revise sesión y permisos antes de dejar pasar la petición; cada vista lo hace a su manera, y una cookie especial deja pasar a cualquiera | Políticas transversales | Front Controller + Chain of Responsibility | Decorator (con el que se suele confundir) envuelve la petición pero siempre la deja pasar; aquí se necesita poder **rechazarla** si la revisión falla | Si el sitio fuera de una sola página sin necesidad de login, no haría falta esta cadena |
| 2 | Al comprar, la forma de pago (efectivo, tarjeta, puntos) se decide con `if`/`elif`, y el pago con tarjeta llama a un banco externo con su propio formato | Aplicación/dominio + Integración | Strategy + Adapter | Factory decide qué objeto crear; aquí el usuario ya eligió el método de pago, lo que cambia es cómo se cobra, no qué objeto se construye | Si solo existiera pago en efectivo, no se necesitaría esta separación |
| 3 | La etiqueta de plantilla `sql_cine.py` arma y ejecuta SQL directamente, y hace una consulta nueva por cada película en un ciclo | Presentación + Datos | Repository + Service Layer | Facade agruparía llamadas de un mismo subsistema para simplificar su uso, pero el problema real es que la plantilla no debería saber de SQL en absoluto | Si fuera una sola consulta usada una sola vez, no se justifica crear una capa completa |
| 4 | Comprar boletos hace tres escrituras seguidas sin ser una sola operación, y no evita el doble clic | Aplicación/dominio + Datos | Idempotencia + Unit of Work | No basta con agregar validaciones sueltas: el cargo, el asiento y el snack deben guardarse juntos o no guardarse | Si la compra fuera un solo paso atómico garantizado por el proveedor de pago, no haría falta |
| 5 | La página principal llama a 9 servicios distintos uno por uno antes de mostrar algo, sin límite real de espera | Integración | Timeout + Circuit Breaker | Reintentar contra un servicio caído no ayuda si sigue caído; hace falta dejar de insistir y no bloquear el resto de la página | Si esos servicios fueran siempre confiables y rápidos, esta protección sería innecesaria |
| 6 | La contraseña de la base de datos y la llave secreta están escritas directamente en `settings.py` | Datos + Integración | Externalizar configuración (variables de entorno) | Juntar todo en un solo archivo de constantes no resuelve el problema: la contraseña real seguiría en el código fuente | En una prueba personal sin datos reales, es aceptable dejarlo así |

---

## 3. Misión 2 — El camino de "comprar un boleto"

### Lo que hace el código

`comprar()` no revisa sesión ni cupo antes de cobrar. Cualquiera puede enviar el formulario de compra directamente, sin haber iniciado sesión, y el pago se procesa igual.

### Camino que debería seguir

| Paso | Qué pasa | Tipo de componente | Patrón |
|---|---|---|---|
| 1 | El usuario elige función, asiento y forma de pago, y presiona "Comprar" | Pantalla | — |
| 2 | Se envía como `POST /comprar` | Petición HTTP | — |
| 3 | La petición entra por un único punto de recepción del sitio | Enrutador de entrada | Front Controller |
| 4 | Antes de procesar el pago se revisa: sesión válida, que el asiento siga libre, y se registra el intento | Filtros previos | Chain of Responsibility |
| 5 | Se llama al caso de uso de comprar boleto, sin decidir reglas de negocio en el controlador | Controlador | — |
| 6 | Se ejecuta `comprarBoleto(funcion, asiento, metodoPago)` | Caso de uso | Service Layer |
| 7 | Se elige cómo cobrar según el método de pago | Selector de cobro | Strategy |
| 8 | Se revisa una clave de esta operación en particular para que un doble clic no duplique la compra | Verificación previa | Idempotencia |
| 9 | Se traduce el cobro al formato que entiende el banco externo, y su respuesta a un formato interno | Conector externo | Adapter |
| 10 | El banco procesa el cobro y responde | Servicio externo | — |
| 11 | Si el cobro fue exitoso, se guarda el boleto y se marca el asiento como ocupado en una sola operación | Persistencia | Repository + Unit of Work |
| 12 | Se avisa a quien deba enterarse (correo al cliente, aviso a taquilla) | Notificación | Observer |
| 13 | Si el banco tarda o falla seguido, se deja de esperar sin detener el resto de la cartelera | Protección externa | Timeout + Circuit Breaker |
| 14 | Se muestra la confirmación de compra | Salida | — |

---

## 4. Misión 3 — Revisiones que se repiten en cada archivo

**a) Por qué conviene un solo punto de entrada:** en este proyecto, la revisión de sesión se escribe distinto en cada vista (`login`, `cartelera`, `membresia`), y una de ellas ni siquiera revisa nada. Cuando la misma tarea se repite en cada archivo, tarde o temprano una copia queda mal hecha o incompleta, como pasó aquí. Un solo punto de entrada evita ese riesgo porque la revisión se escribe una sola vez.

**b) Qué ya trae el framework:** Django ya incluye un sistema de middleware (`MIDDLEWARE` en `settings.py`) pensado exactamente para esto — de hecho el proyecto trae `AuthenticationMiddleware`, pero está deshabilitado a propósito. No hace falta escribir una cadena de filtros a mano: basta con activar lo que Django ya ofrece.

**c) Orden correcto:** primero autenticación, después revisar cupo (que el asiento siga libre), después registrar el intento en bitácora, y al final ejecutar la compra. Cobrar antes de confirmar quién es el usuario es grave porque, si después resulta que no se pudo verificar su identidad, ya se le cobró a alguien sin saber con certeza quién es ni cómo revertirlo con seguridad.

**d) Qué no debe ir en esos filtros:** las reglas propias de cada acción (calcular el precio del boleto, elegir el banco, armar la confirmación) no deben vivir en los filtros generales; esas decisiones son del caso de uso específico, no de una revisión que corre para todas las peticiones por igual.
