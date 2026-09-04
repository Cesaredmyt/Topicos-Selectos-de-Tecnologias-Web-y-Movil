# Auditoría de arquitectura — Portal universitario (caso de estudio)

**Alumnos:** 
> **Diaz Maldonado Cesar Enrique**
> **Martínez Enrique**
> **Medina Ayala José Antonio**
---

Para cada archivo se buscó qué mala práctica aparece, indicando el archivo y la línea donde se observa.

---

## 1. General

### 1.1 Patrones de diseño

Después de revisar archivo por archivo, se observa que no hay ningún patrón de diseño aplicado de forma completa y consciente. Existen lo que parecen ser intentos de organizar el código (por ejemplo, separar la conexión a base de datos en su propio archivo, o agrupar funciones en `global/funciones.php`), pero ninguno llega a comportarse como el patrón que imita o siquiera de forma parecida.

- La separación de `config/db_connect.php` parece el inicio de un patrón de configuración externa, pero la contraseña sigue escrita dentro del código, así que no cumple el propósito real de ese patrón (sacar el secreto del código fuente).
- La agrupación de funciones en `global/funciones.php` parece una capa de utilidades, pero en realidad son cientos de funciones casi idénticas copiadas y pegadas, nada reutilizable.

El proyecto no sigue MVC, no sigue una arquitectura por capas, no tiene un punto único de entrada y no separa la lógica de negocio de la presentación ni del acceso a datos. Cada archivo PHP es un script independiente que mezcla las tres cosas a la vez, llamando cada parte en donde es necesario en lugar de centralizar los llamados.

### 1.2 Malas prácticas detectadas

| Archivo / carpeta | Qué se observa | Por qué es un problema |
|---|---|---|
| `index.php:3` | `error_reporting(0)` | Se ocultan todos los errores y avisos de PHP. Si algo falla, nadie se entera; los errores quedan invisibles tanto para el equipo de desarrollo (que parece no existir) como para quien da soporte. |
| `index.php:6` | `if(isset($_COOKIE['admin_bypass'])) { $_SESSION['user_id'] = 1; }` | Cualquier persona puede entrar como usuario administrador simplemente creando una cookie llamada `admin_bypass` en su navegador, dando paso a una puerta trasera que puede ser utilizada por cualquier persona con conocimiento básico acerca de un sistema sencillo. |
| `index.php` (líneas 8 en adelante) | Cientos de clases CSS `clase_basura_0` a `clase_basura_623`, todas con un patrón repetido y sin usarse en ningún lado | Código muerto que solo agrega peso al archivo y dificulta encontrar la parte que funciona incorrectamente si es que lo hace. |
| `pagar.php` | Un `switch` con 30 casos casi idénticos (`banco_0` a `banco_29`), cada uno llamando a un servicio SOAP distinto y comparando un código de estado distinto (`OK_0`, `OK_1`, …) | Agregar un banco nuevo obliga a copiar y pegar un bloque completo. Un error de tecleo en un solo caso (por ejemplo, comparar `OK_5` cuando el banco realmente devuelve `OK_05`) pasa fácilmente desapercibido entre 30 bloques iguales. |
| `pagar.php en los case`) | `mysqli_query($conn, 'UPDATE pagos SET st = 1')` sin ninguna condición `WHERE` | Esa instrucción marca todos los pagos de la tabla como pagados, no solo el del alumno que está pagando en ese momento. Es un error grave, no solo de estilo. |
| `pagar.php` / `pagar1.php` | No existe ningún control para evitar que la misma operación se ejecute dos veces (doble clic) | Un doble clic accidental puede generar un doble cobro real a la tarjeta del alumno. |
| `kardex.php: Primeras lineas` | Diez `if` anidados uno dentro de otro para descartar matrículas de prueba | Es muy difícil de leer y de modificar; agregar una matrícula más a la lista de exclusión implica anidar un nivel más. |
| `kardex.php:14 en adelante`| Dentro del ciclo que recorre las materias del alumno se ejecuta una consulta SQL nueva por cada materia (y en `kardex1.php` una tercera consulta por cada profesor) | Si un alumno tiene 40 materias en su historial, se disparan más de 80 consultas a la base de datos para mostrar una sola página armada por parches. Esto vuelve la página lenta y sobrecarga el servidor de base de datos sin necesidad. |
| `kardex.php` | `for($j=0; $j<1000; $j++) { $basura = md5($m_nom . $j); }` dentro del ciclo | Es trabajo que no sirve para nada: calcula mil veces un valor que nunca se usa. Solo hace que la página tarde más en cargar. |
| `global/funciones.php` (2001 líneas) | Alrededor de 600 funciones casi idénticas (`formatear_string_v0` … `formatear_string_v605`), cada una reemplazando un número distinto por la letra `X` | Es la misma función copiada 600 veces cambiando solo un valor. Debería ser una sola función que reciba ese valor como parámetro. |
| `js1/utilerias1.js` | El mismo patrón de copiar y pegar, del lado del navegador (`accion_interfaz_0`, `accion_interfaz_1`, …) | El problema de duplicación no solo está en servidor; se repite también en el código que corre en el navegador del usuario. |
| `config/db_connect.php`, `config/db_connect_produccion.php`, `config1/db_connect1.php`, `config1/db_connect_produccion1.php`, y además dentro de `login1.php`, `kardex1.php`, `pagar1.php` | La contraseña de la base de datos está escrita directamente en el código, y en `config/db_connect_produccion.php` aparece una contraseña real de producción (`P@ssw0rd_Real_2025!`) | Cualquier persona con acceso al código (o a una copia del repositorio) puede leer la contraseña real de la base de datos de producción y modificarla. Además, la misma configuración está copiada en cuatro lugares distintos: si la contraseña cambia, hay que recordarse de actualizarla en los cuatro archivos. |
| `api.php:5` | `eval($_POST['codigo_php'])` | Este archivo ejecuta como código PHP cualquier texto que alguien envíe por internet. Es dejar la puerta principal del servidor abierta, dando posibilidad a que quien encuentre esta dirección puede ejecutar lo que quiera en el servidor. |
| `api.php:7` | `echo file_get_contents($_GET['archivo'])` | Permite leer cualquier archivo del servidor que se indique por la dirección web, incluyendo archivos de configuración con contraseñas. |
| `login1.php:8` | `"SELECT ... WHERE correo = '$usr' AND password = '$pwd'"` construida concatenando directamente lo que escribe el usuario, abriendo la posibilidad hacia una inyección SQL | La consulta se arma pegando texto sin limpiarlo. Alguien puede escribir un correo especial en el formulario para alterar el significado de la consulta y entrar sin conocer la contraseña real. |
| `login1.php:8` | La contraseña se compara tal cual está guardada, sin ningún cifrado | Si alguien copia la base de datos, obtiene las contraseñas de todos los usuarios en texto legible. |
| `login1.php:13-14` y `index1.php:4` | El sistema recuerda quién inició sesión usando dos mecanismos distintos al mismo tiempo: una cookie (`usuario_logueado`, `admin`) y la sesión de PHP (`$_SESSION['id']`) | La cookie `admin` la controla el navegador del usuario, no el servidor. Basta con editar esa cookie a mano para hacerse pasar por administrador, sin necesidad de conocer ninguna contraseña. |
| `index1.php:8` | `"SELECT * FROM perfiles WHERE id = " . $perfil` con `$perfil` tomado directamente de la dirección web | Mismo problema de inyección SQL que en el inicio de sesión, ahora para consultar el perfil de cualquier usuario. |
| `pagar1.php:14-16` | El número de tarjeta y el código de seguridad (CVV) se arman como texto plano dentro de una variable llamada `$xml_banco` y se envían por internet a `https://api-banco-falso.com` | Los datos de la tarjeta viajan sin ninguna protección adicional dentro del código; cualquier error en ese envío expone información financiera sensible. |
| `pagar1.php:11` | `$importe = 2500;` fijo en el código, sin relación con el monto real de la inscripción del alumno | El sistema siempre cobra la misma cantidad sin importar cuánto debería pagar el alumno en realidad. |
| `documentacion/diagrama_base_datos.txt` | Describe tablas como `Estudiantes_V2`, `Carreras_Activas`, `Registro_Pagos_Finanzas`, y reglas ("no usar la tabla perfiles", "las calificaciones están en un XML externo" que no existen en el código ni en en el archivo que pareciera ser la base de datos `bd/script_produccion_real_no_tocar.sql` (que sí define `perfiles`, `historial_academico`, `cat_materias` y `pagos`) | La documentación entregada no corresponde a lo que el código realmente hace. Si alguien nuevo en el equipo confía en ese documento para trabajar, va a modificar tablas que no se usan y va a dejar sin tocar las que sí importan. |
| `bd/script_produccion_real_no_tocar.sql` | Contiene 50 tablas llamadas `tabla_abandonada_0` a `tabla_abandonada_49`, vacías y sin relación con el resto | Es una base de datos con mucho contenido que nadie usa, mezclado con el contenido real. Dificulta saber qué tabla es la que realmente importa. |
| Archivos duplicados: `index.php`/`index1.php`, `kardex.php`/`kardex1.php`, `pagar.php`/`pagar1.php` | Existen dos versiones de la misma función del sistema, con reglas de negocio distintas entre sí (por ejemplo, `pagar.php` cobra por SOAP a 30 bancos distintos y `pagar1.php` cobra un monto fijo por una API externa distinta) | No se indica cuál de las dos versiones es la que realmente está en uso. Ambos archivos duplican el trabajo de mantenimiento y multiplica el riesgo de que uno de los dos quede desactualizado sin que nadie lo note. |

### 1.3 ¿Se puede corregir poco a poco, o conviene rehacerlo desde cero?

**Se recomienda rehacer la arquitectura desde cero**, apoyándose en el código actual solo como referencia de las reglas de negocio (qué debe pasar cuando alguien paga, qué debe mostrar el kardex), pero no como base para editar línea por línea.

Lo que sí se puede reutilizar del proyecto actual: el modelo de datos base una vez depurado (quitando las tablas `tabla_abandonada_N`), y las reglas de negocio identificables (por ejemplo, los rangos de calificación del kardex, o el flujo general de "cobrar → dar de alta materias → avisar").

---

## 3. Misión 1 — El portal no es un patrón


| # | Problema detectado en el proyecto | Capa afectada | Patrón(es) que lo resuelve | Por qué ese y no el patrón parecido que suele confundirse | Cuándo NO conviene aplicarlo |
|---|---|---|---|---|---|
| 1 | No existe un punto único de entrada: la sesión, las cookies y la autorización se revisan de forma distinta en `index.php`, `index1.php`, `kardex1.php` y `pagar1.php`, cada quien usando diferentes formas | Políticas transversales | **Front Controller + Chain of Responsibility** | La cadena puede cortar la petición si una revisión falla (por ejemplo, sin sesión válida → acceso denegado). El patrón Decorator, que a veces se confunde con este caso, siempre envuelve la petición y la deja pasar; no está pensado para detenerla | Si el sitio tuviera solo dos o tres páginas fijas, sin planes de crecer, centralizar la entrada sería más trabajo de configuración que beneficio real |
| 2 | `pagar.php` es un `switch` de 30 casos casi idénticos, uno por banco, cada uno con su propio protocolo SOAP y sus propios códigos de estado (`OK_0` a `OK_29`) | Aplicación/dominio + Integración | **Strategy** (para elegir el algoritmo de cobro) y ** Adapter** (para traducir cada protocolo bancario a un formato común) | El patrón Factory, que suele confundirse aquí, sirve para decidir qué objeto crear; en este caso el método de pago ya lo eligió el alumno en el formulario, lo que cambia es el algoritmo de cobro en sí, no el tipo de objeto | Si el sistema trabajara con un solo banco, sin planes de agregar otro, armar una interfaz con una sola implementación sería complejidad de más |
| 3 | `kardex.php` y `kardex1.php` ejecutan SQL directamente dentro del script que arma la salida en pantalla, y hacen una consulta nueva por cada materia dentro de un ciclo | Presentación (servidor) + Datos | **Repository + Service Layer** | El patrón Facade, con el que se suele confundir este caso, agrupa llamadas a un subsistema propio para simplificar su uso, pero no resuelve el problema de fondo, que es que la vista contiene SQL mezclado con HTML. Repository es el nombre exacto de la pieza que falta: una capa que aísla cómo se consultan los datos | Si fuera una sola consulta simple, usada una única vez y sin repetirse en ningún otro reporte, no se justifica construir una capa de repositorio completa |
| 4 | `pagar.php` actualiza la tabla `pagos` sin condición que la ligue a un alumno o pago específico, y no existe ningún control para evitar que un doble clic dispare dos cobros | Aplicación/dominio + Datos | **Idempotencia (clave de operación) + Unit of Work** | Arreglar solo la condición de la consulta no evita que la misma petición se ejecute dos veces | Si el cobro fuera un solo paso completamente sincronizado y el banco garantizara que un mismo folio nunca se cobra dos veces, añadir esta capa sería innecesario |
| 5 | `global/funciones.php` contiene cerca de 600 funciones casi idénticas, cada una reemplazando un número distinto por la letra `X` | Aplicación/dominio (utilidades compartidas) | **Función única parametrizada** (no se necesita Strategy) | Strategy, tiene sentido cuando el comportamiento cambia entre variantes. En este caso el comportamiento es exactamente el mismo, solo cambia un valor de entrada; basta un parámetro | Cuando el comportamiento cambia de forma real entre casos (no solo un valor), forzarlo a una sola función con muchos `if` internos sí sería un error; ahí conviene Strategy |
| 6 | Las credenciales de la base de datos están escritas directamente en el código, repetidas en cuatro archivos distintos (`config/db_connect.php`, `config/db_connect_produccion.php`, `config1/db_connect1.php`, `config1/db_connect_produccion1.php`), incluida una contraseña real de producción | Datos + Integración (configuración transversal) | **Externalización de configuración (variables de entorno) + un único punto de conexión** | Aún unificando los archivos, la contraseña real seguiría dentro del código fuente. Lo que corrige el problema es sacar el secreto del código y tener un solo lugar que construya la conexión | En un script personal, de un solo uso, sin datos sensibles reales, escribir la conexión directamente en el código es aceptable |

---

## 2. Misión 2 — Una petición, varios patrones

### 2.1 Lo que hace el código

No existe un solo camino claro para "pagar la inscripción": el proyecto tiene dos versiones distintas y contradictorias (`pagar.php` y `pagar1.php`), cada una accesible directamente por su propia dirección web, sin pasar por ningún punto de control común, y sin que quede documentado cuál es la versión vigente.

### 2.2 Camino corregido a seguir


| Paso | Qué ocurre | Tipo de componente | Patrón que representa en ese momento |
|---|---|---|---|
| 1 | El alumno llena el formulario de pago (monto, método) y presiona "Pagar" | Vista / pantalla del cliente | Representación de la interfaz (fuera del alcance de esta tabla de backend) |
| 2 | El formulario se envía como `POST /pagos` hacia el servidor | Petición HTTP | — |
| 3 | La petición entra por un único punto de recepción del portal, en vez de llegar directo a un archivo `.php` suelto | Enrutador de entrada | **Front Controller** |
| 4 | Antes de llegar a la lógica de pago, se revisa en orden: que la sesión sea válida, que el alumno tenga cupo para inscribirse, y se registra el intento en la bitácora; cualquiera de estas revisiones puede detener la petición | Middleware / filtros previos | **Chain of Responsibility** |
| 5 | Un componente recibe los datos ya validados del formulario y llama al caso de uso de negocio, sin decidir reglas por su cuenta | Controlador de la operación | Controlador de la operación (recibe la entrada, no decide el negocio) |
| 6 | Se invoca la operación `pagarInscripcion(alumnoId, metodo, monto)` | Capa de casos de uso | **Service Layer** |
| 7 | Dentro de esa operación, se elige cómo cobrar según el método (tarjeta, banco específico, ventanilla) | Selector de algoritmo de cobro | **Strategy** |
| 8 | Antes de ejecutar el cobro, se revisa una clave que identifica esta operación en particular, para que un doble clic no dispare un segundo cobro | Verificación de operación repetida | **Idempotencia** |
| 9 | Se traduce la petición de cobro al formato que entiende el banco o pasarela externa (SOAP, XML u otro), y se traduce también su respuesta a un formato interno común | Conector hacia el proveedor externo | **Adapter** |
| 10 | El banco procesa el cobro y responde con un código de resultado | Servicio externo | — |
| 11 | Si el cobro fue exitoso, se guardan el cargo en la cuenta del alumno y el alta de las materias como una sola operación conjunta: o se guardan las dos cosas, o no se guarda ninguna | Persistencia de datos | **Repository + Unit of Work** |
| 12 | Una vez confirmado el pago, se avisa a quienes deben reaccionar (correo al alumno, aviso a control escolar, actualización de caja), sin que la operación de pago tenga que conocer a cada uno de ellos por nombre | Notificación de eventos internos | **Observer** |
| 13 | Si el banco tarda demasiado o falla de forma repetida, se deja de esperar después de un tiempo límite y se evita seguir insistiendo mientras el problema continúe, sin bloquear el resto del sistema (por ejemplo, que el kardex, que no depende del banco, siga funcionando) | Protección ante fallas del proveedor externo | **Timeout + Circuit Breaker** |
| 14 | Se arma la respuesta para el alumno confirmando el pago y, si corresponde, el detalle de las materias inscritas | Formato de salida | Plantilla de salida (adaptada al tipo de cliente que la pidió) |

---


## 3. Misión 3 — Políticas transversales

### a) Cuándo y por qué conviene una sola puerta de entrada (Front Controller)

En el proyecto actual, la revisión de sesión aparece de forma distinta en index.php (revisa $_SESSION['user_id'] y además acepta una cookie de puerta trasera), en index1.php (revisa una cookie y la sesión al mismo tiempo) y en kardex1.php (no revisa nada). Cuando la misma tarea — revisar quién es el usuario, si tiene permiso, y dejar registro del intento — se repite en cada archivo por separado, tarde o temprano una de las copias queda mal hecha o se le olvida al programador, como pasó aquí. Conviene una sola puerta de entrada cuando el sistema tiene varias rutas o funciones que todas necesitan pasar por las mismas revisiones antes de ejecutarse: en ese caso, escribir la revisión una sola vez y hacer que todo pase por ahí evita que se repita (y se desactualice) en cada archivo.

### b) Qué ya traen los marcos de trabajo modernos para esto

Los marcos de trabajo modernos ya incluyen un mecanismo llamado middleware. Permite declarar, en un solo lugar, la lista de revisiones que debe pasar una petición antes de llegar a la lógica de negocio, sin tener que escribir manualmente el código que revisa cada cosa dentro de cada archivo. No hace falta construir una cadena de filtros a mano.

### c) Orden correcto de la revisión previa

El orden recomendado es:

1. *Autenticación* — confirmar quién es la persona que hace la petición.
2. *Cupo / validación de negocio* — confirmar que esa persona puede realizar la acción que pide (por ejemplo, que tiene lugar disponible para inscribirse).
3. *Bitácora* — dejar constancia de que la petición ocurrió.
4. *Caso de uso* — ejecutar la acción de negocio en sí (el cobro, el guardado, etc.).

Cobrar antes de autenticar es un error grave porque, si se cobra primero y la identidad de la persona se confirma después, el sistema puede terminar cobrando a alguien que ni siquiera pudo demostrar quién es. Si esa autenticación falla después del cobro, ya no hay una manera limpia de saber a quién pertenece ese dinero ni cómo revertirlo con seguridad. El orden correcto evita que el sistema tome una acción irreversible antes de tener la certeza de con quién está tratando.

### d) Qué NO debe ir dentro de esa cadena de filtros previos

No deben colocarse dentro de la cadena de filtros previos las decisiones que pertenecen al negocio específico de cada operación. Esas decisiones dependen del caso de uso concreto (pagar, consultar kardex, etc.) y deben vivir dentro de esa operación, no en un filtro genérico que se ejecuta para todas las peticiones por igual. Meter reglas de negocio dentro de los filtros previos vuelve esos filtros difíciles de reutilizar y mezcla otra vez, sin necesidad, cosas que deberían mantenerse separadas.
