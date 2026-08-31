# Lenguajes de Programación, Patrones de Diseño y Paradigmas

# Cesar Enrique Diaz Maldonado                                                        31/Agosto/26

Investigar **20 lenguajes de programación o frameworks**, identificar el **paradigma** que utilizan, los **patrones de diseño** más asociados a ellos, y analizar **qué problema real le genera ese paradigma** cuando se usa mal o se lleva al extremo.

---

| # | Lenguaje / Framework | Paradigma principal | Patrones más asociados | Problema típico del paradigma |
|---|---|---|---|---|
| 1 | Java | OOP clásico, imperativo | Factory / Abstract Factory, Decorator, Observer | Herencia profunda + deadlocks por sincronización manual |
| 2 | C++ | Multiparadigma (OOP, genérico, procedural) | RAII, PIMPL | Problema del diamante (herencia múltiple) y destructores no ejecutados |
| 3 | C# | Multiparadigma, OOP empresarial | Dependency Injection, Observer (`event`/`delegate`) | Sobre-ingeniería (exceso de interfaces) y fugas de memoria por eventos "huérfanos" |
| 4 | Python | Multiparadigma, tipado dinámico | Decorator, Iterator/Generator | Errores que solo aparecen en ejecución + el GIL limita el paralelismo real |
| 5 | JavaScript | Multiparadigma, basado en prototipos, asíncrono | Module Pattern, Pub-Sub/EventTarget | Modificar prototipos globales rompe toda la app + "callback hell" pierde errores |
| 6 | TypeScript | Tipado estático sobre JS | Builder, Strategy, Adapter | Los tipos desaparecen al compilar (type erasure); `any` rompe la seguridad |
| 7 | Ruby | OOP dinámico + metaprogramación | Active Record, Template Method | "Fat Models" (modelos gigantes) y trazas de error confusas por magia dinámica |
| 8 | PHP / Laravel | Multiparadigma orientado a web | Facade, Service Container (IoC) | Dependencias ocultas + cada petición reinicia todo desde cero (más lento) |
| 9 | Go | Concurrente (CSP), procedural | Worker Pool, Fan-In/Fan-Out | Fugas de goroutines (quedan bloqueadas para siempre) + `if err != nil` repetitivo |
| 10 | Rust | Ownership/Borrowing, funcional | Typestate, RAII | Estructuras con referencias cruzadas (listas, árboles) son muy difíciles de escribir |
| 11 | Haskell | Funcional puro, evaluación perezosa | Monad, Functor/Applicative | "Space leaks": se acumulan operaciones pendientes en memoria hasta explotar |
| 12 | Next.js | Declarativo, componentes híbridos | Container/Presentational, Middleware | Los datos entre servidor y cliente deben ser serializables; librerías pesadas pueden colarse al navegador |
| 13 | Node.js + Express | Asíncrono, un solo hilo (event loop) | Middleware (Chain of Responsibility), Repository/Service | Una tarea pesada de CPU congela **todo** el servidor |
| 14 | Kotlin | Multiparadigma, null-safety | Extension Functions, Coroutines | Las funciones de extensión dispersan el código y ocultan qué pertenece a cada clase |
| 15 | Angular | OOP + reactivo (RxJS/Signals) | Dependency Injection jerárquica, Observer | Fugas de memoria si no te desuscribes de un Observable al destruir el componente |
| 16 | React | Declarativo, funcional reactivo | Custom Hooks, Context Provider | Re-renders en cascada + "stale closures" (datos viejos atrapados en un hook) |
| 17 | Django | MTV (Model-Template-View) | Active Record (ORM), Middleware | Problema de las consultas N+1: cientos de consultas SQL innecesarias |
| 18 | Spring (Java) | AOP + OOP empresarial | Inversion of Control, Dynamic Proxy | Si un método se llama a sí mismo internamente, el proxy se salta y falla en silencio |
| 19 | Prolog | Declarativo lógico | Pattern Matching/Unificación, Generate and Test | Explosión combinatoria: la búsqueda puede volverse infinita y tumbar la memoria |
| 20 | C | Imperativo, procedural, bajo nivel | Opaque Pointer, Function Tables (vtable manual) | Sin garbage collector ni límites de arreglos: buffer overflows, punteros colgantes, segfaults |

---

### 1. Java
- **Paradigma:** Orientado a Objetos clásico, basado en clases, imperativo y fuertemente tipado.
- **Patrones:** *Factory/Abstract Factory* (crea objetos sin exponer cómo), *Decorator* (así funciona `java.io`, cuando envuelves un `FileInputStream` en un `BufferedInputStream`), *Observer* (eventos y colecciones que "avisan" cambios).
- **El problema, en simple:** Como en Java "todo tiene que ser una clase", es fácil abusar de la herencia para reutilizar código. Terminas necesitando algo simple (una función) pero arrastrando toda una jerarquía de clases pesadas (esto se conoce como el problema del "gorila y la banana": pediste la banana y te trajeron al gorila con todo y la selva). Además, como los objetos compartidos entre hilos son mutables por defecto, hay que sincronizar todo a mano, lo que puede causar que dos hilos se queden esperando el uno al otro para siempre (**deadlock**).

### 2. C++
- **Paradigma:** Multiparadigma: OOP con herencia múltiple, genérico (plantillas), procedural y funcional.
- **Patrones:** *RAII* (el recurso se libera automáticamente cuando el objeto sale de "ámbito", como cuando una variable local termina su vida), *PIMPL* (esconder los detalles privados de una clase para que compile más rápido).
- **El problema, en simple:** C++ permite que una clase herede de dos clases a la vez. Si ambas heredan de una misma clase "abuela", terminas con dos copias del mismo dato y no se sabe cuál usar (**problema del diamante**). Además, si olvidas marcar un destructor como `virtual`, al borrar un objeto por su clase base el destructor de la clase hija nunca se ejecuta, dejando basura de memoria sin liberar.

### 3. C#
- **Paradigma:** Multiparadigma, con fuerte enfoque en OOP empresarial y algo de funcional (LINQ).
- **Patrones:** *Dependency Injection* (el propio .NET te entrega las dependencias listas), *Observer* (con las palabras clave `event` y `delegate`, integradas al lenguaje).
- **El problema, en simple:** La cultura de C# empuja a crear una interfaz para todo (`IUser`, `IUserService`, `IUserRepository`...), lo que vuelve el código difícil de seguir. Además, si un objeto se suscribe a un evento de otro objeto que vive más tiempo, ese segundo objeto lo va a "retener" en memoria para siempre, aunque ya no lo necesites, y el recolector de basura no puede liberarlo (fuga de memoria).

### 4. Python
- **Paradigma:** Multiparadigma con tipado dinámico (OOP, imperativo, funcional).
- **Patrones:** *Decorator* (el famoso `@` que modifica funciones o clases), *Iterator/Generator* (`yield`, para producir datos poco a poco sin llenar la memoria).
- **El problema, en simple:** Como Python no revisa los tipos antes de ejecutar el programa, un error de "esto no tiene ese método" solo aparece cuando esa línea exacta se ejecuta, no antes. Además, el **GIL** (un candado interno del intérprete) impide que el código Python use varios núcleos del procesador al mismo tiempo, así que si tu tarea depende de cálculo puro (no de esperar internet o disco), los hilos no te dan más velocidad real.

### 5. JavaScript
- **Paradigma:** Multiparadigma, asíncrono, orientado a eventos y basado en prototipos (no tiene clases "de verdad").
- **Patrones:** *Module Pattern* (encerrar variables privadas dentro de un archivo o función), *Pub-Sub/EventTarget* (`addEventListener`, `EventEmitter`).
- **El problema, en simple:** En JS, si una librería modifica un método compartido por todos los arreglos u objetos (`Array.prototype`), **toda la aplicación** se ve afectada, incluso partes que no tienen nada que ver. Y si no manejas bien las promesas o encadenas demasiados callbacks ("callback hell"), los errores se pierden silenciosamente y la app se queda como "congelada" sin explicar por qué.

### 6. TypeScript
- **Paradigma:** Multiparadigma con tipado estático (sobre JavaScript).
- **Patrones:** *Builder* y *Strategy* (usando genéricos `<T>` para hacerlos seguros), *Adapter* (para encajar una API externa con tus tipos).
- **El problema, en simple:** Los tipos de TypeScript solo existen mientras escribes el código; al compilar a JavaScript, **desaparecen por completo**. Si tu backend te manda un dato distinto al que declaraste, TypeScript no te va a avisar, el error va a aparecer solo cuando el programa ya está corriendo. Y si abusas de `any` o forzar tipos con `as`, básicamente le estás diciendo al compilador "confía en mí" y pierdes toda la protección.

### 7. Ruby
- **Paradigma:** OOP puro con metaprogramación dinámica (Rails usa MVC con "convención sobre configuración").
- **Patrones:** *Active Record* (una fila de la base de datos se convierte automáticamente en un objeto), *Template Method* (`before_action`, pasos que se ejecutan antes de cada acción del controlador).
- **El problema, en simple:** Como Active Record mezcla las reglas de negocio con el acceso a la base de datos en el mismo archivo, los modelos crecen y crecen hasta volverse gigantes e imposibles de testear por separado ("Fat Model"). Además, como Ruby genera métodos "mágicos" en tiempo de ejecución, cuando algo falla el mensaje de error es confuso porque ese método ni siquiera existía escrito en el código.

### 8. PHP / Laravel
- **Paradigma:** Multiparadigma orientado a la web (Laravel implementa MVC).
- **Patrones:** *Facade* (acceso simple tipo `Route::get()` a clases más complejas por dentro), *Service Container* (el framework resuelve automáticamente qué dependencias inyectar).
- **El problema, en simple:** Usar demasiadas Facades hace que el código parezca "no depender de nada", pero en realidad sí depende del framework completo por dentro, lo que obliga a levantar toda la aplicación solo para hacer una prueba simple. Además, PHP clásico destruye y vuelve a crear todo desde cero en cada petición HTTP, lo cual es más lento que mantener el estado en memoria.

### 9. Go
- **Paradigma:** Concurrente (basado en CSP: procesos que se comunican), procedural, prioriza composición sobre herencia.
- **Patrones:** *Worker Pool* (varias goroutines toman tareas de una fila compartida), *Fan-In/Fan-Out* (dividir el trabajo en paralelo y luego juntar los resultados).
- **El problema, en simple:** Go no tiene `try/catch`; los errores se devuelven como un valor más, lo que llena el código de `if err != nil` repetido una y otra vez. Y si lanzas una goroutine que espera un canal que nadie más va a usar, esa rutina se queda **bloqueada para siempre**, consumiendo memoria sin que nadie se dé cuenta (fuga de goroutines).

### 10. Rust
- **Paradigma:** Concurrente, funcional e imperativo, controlado por un sistema de "propiedad" (Ownership y Borrowing).
- **Patrones:** *Typestate* (el compilador impide usar un objeto en un estado inválido), *RAII* (el recurso se libera automáticamente al final del ámbito).
- **El problema, en simple:** Rust es muy estricto: no te deja tener al mismo tiempo una referencia que modifica algo y otra que solo la lee. Esto es genial para evitar bugs, pero hace que estructuras normales como listas doblemente enlazadas o árboles con referencias al "padre" sean muy difíciles de programar. Para lograrlo, terminas usando envoltorios especiales (`Rc<RefCell<T>>`, `Arc<Mutex<T>>`) que agregan complejidad y algo de costo en rendimiento.

### 11. Haskell
- **Paradigma:** Funcional puro, con evaluación perezosa (no calcula nada hasta que de verdad se necesita).
- **Patrones:** *Monad* (encapsula efectos como leer un archivo o cambiar un estado, sin romper la "pureza" del lenguaje), *Functor/Applicative* (aplicar funciones dentro de un contexto, como un valor que puede o no existir).
- **El problema, en simple:** Como Haskell pospone los cálculos hasta el último momento, se van acumulando "promesas de cálculo" pendientes (llamadas *thunks*) en memoria. Si una función recursiva acumula millones de estas promesas sin resolverlas antes, la memoria se dispara hasta tumbar el programa (esto se conoce como *space leak*).

### 12. Next.js
- **Paradigma:** Declarativo, basado en componentes híbridos (mezcla renderizado en el servidor y en el cliente).
- **Patrones:** *Container/Presentational* (separar componentes de servidor que traen datos, de componentes de cliente que manejan la interacción), *Middleware* (intercepta peticiones antes de llegar a una ruta).
- **El problema, en simple:** Al dividir la app en "mundo servidor" y "mundo cliente", cualquier dato que pases de uno a otro tiene que poder convertirse a JSON; si intentas pasar una función o una clase compleja, falla. Y si no tienes cuidado con qué importas, puedes terminar empaquetando librerías pesadas del backend dentro del código que se descarga en el navegador del usuario, haciendo la app más lenta de cargar.

### 13. Node.js + Express
- **Paradigma:** Asíncrono, no bloqueante, con un solo hilo principal (event loop).
- **Patrones:** *Middleware* (Chain of Responsibility: la petición pasa por una fila de funciones antes de llegar al controlador final), *Repository/Service* (separar el acceso a datos de la lógica de negocio).
- **El problema, en simple:** Node asume que casi todo el tiempo se va esperando cosas externas (bases de datos, red), no calculando. Si metes una operación pesada de CPU (procesar una imagen, un bucle enorme) en el hilo principal, **todo el servidor se congela** mientras dura ese cálculo, y no puede atender a ningún otro usuario mientras tanto.

### 14. Kotlin
- **Paradigma:** Multiparadigma, con interoperabilidad total con Java y seguridad contra nulos (Null Safety) integrada.
- **Patrones:** *Extension Functions* (agregar funciones nuevas a una clase que ya existe, sin heredar de ella), *Coroutines* (concurrencia estructurada: si una tarea falla, cancela automáticamente a sus tareas "hijas").
- **El problema, en simple:** Como puedes escribir una función de extensión para una clase desde cualquier archivo del proyecto, se vuelve difícil saber qué métodos "realmente pertenecen" a esa clase y cuáles son solo utilidades agregadas por fuera. En proyectos grandes esto complica mucho encontrar y mantener el código.

### 15. Angular
- **Paradigma:** Orientado a Objetos empresarial, declarativo y totalmente reactivo (RxJS/Signals).
- **Patrones:** *Dependency Injection* jerárquica (los servicios pueden ser globales o limitados a un componente), *Observer* (casi toda la comunicación se maneja con flujos de datos/Observables).
- **El problema, en simple:** Si un componente se suscribe a un Observable (por ejemplo, un temporizador) y luego se destruye sin que te acuerdes de desuscribirte, Angular no corta esa conexión automáticamente. El componente queda "atrapado" en memoria para siempre, acumulando listeners y haciendo la app cada vez más lenta.

### 16. React
- **Paradigma:** Declarativo y funcional reactivo (la interfaz es una función del estado).
- **Patrones:** *Custom Hooks* (encapsular lógica reutilizable con estado), *Context Provider* (compartir datos globales sin pasar props nivel por nivel).
- **El problema, en simple:** Cada vez que el estado cambia, React vuelve a calcular el árbol de componentes. Si le pasas a un hijo un objeto o función "nueva" en cada render, React piensa que cambió y lo vuelve a dibujar innecesariamente (renders en cascada). Y si un `useEffect` no declara bien sus dependencias, puede quedarse usando datos "viejos" atrapados en memoria (*stale closures*), generando bugs difíciles de rastrear.

### 17. Django (Python)
- **Paradigma:** MTV (Model-Template-View), basado en no repetir código (DRY) y diseño monolítico.
- **Patrones:** *Active Record* (el ORM de Django: cada modelo hereda de `models.Model`), *Middleware* (procesa la petición antes y la respuesta después de la vista).
- **El problema, en simple:** El ORM es "perezoso": si recorres 100 libros para mostrar el nombre de su autor, Django no hace un solo `JOIN`, hace 1 consulta para los libros y **100 consultas más**, una por cada autor. Esto se llama el **problema N+1** y puede saturar la base de datos si no usas `select_related` o `prefetch_related` para traer todo junto.

### 18. Spring (Java)
- **Paradigma:** Programación Orientada a Aspectos (AOP) combinada con OOP empresarial.
- **Patrones:** *Inversion of Control* (Spring administra el ciclo de vida de todos los objetos, llamados *Beans*), *Dynamic Proxy* (envuelve tus clases para agregar cosas como transacciones o seguridad de forma automática).
- **El problema, en simple:** Toda la magia de anotaciones como `@Transactional` depende de que la llamada pase por un "envoltorio" (proxy) que Spring crea alrededor de tu clase. Pero si un método llama a otro método **de la misma clase** directamente (`this.metodo()`), se salta ese envoltorio y la anotación simplemente no funciona — y encima, **no da ningún error**, solo falla en silencio.

### 19. Prolog
- **Paradigma:** Declarativo lógico. No se escriben pasos, se definen hechos y reglas, y el motor busca soluciones automáticamente.
- **Patrones:** *Pattern Matching/Unificación* (busca cómo emparejar variables entre sí), *Generate and Test* (un predicado genera posibles soluciones y otro las valida).
- **El problema, en simple:** El motor de Prolog explora un árbol de posibilidades probando y retrocediendo (*backtracking*). Si las reglas no están bien escritas o el orden de las condiciones es incorrecto, ese árbol de búsqueda puede crecer sin control (explosión combinatoria), consumiendo toda la memoria disponible incluso para problemas que parecían simples.

### 20. C
- **Paradigma:** Imperativo, procedural y estructurado; muy cercano a cómo funciona la CPU por dentro.
- **Patrones:** *Opaque Pointer/Handle* (ocultar los detalles internos de una estructura, simulando encapsulamiento como en OOP), *Function Tables* (una estructura con punteros a funciones, para simular polimorfismo).
- **El problema, en simple:** C no tiene recolector de basura, ni clases, ni revisión automática de límites en los arreglos. Reservar (`malloc`) y liberar (`free`) memoria depende 100% del programador. Un descuido produce directamente **desbordamientos de memoria (buffer overflows)**, **punteros que apuntan a memoria ya liberada (dangling pointers)** o **fallos de segmentación (segfaults)** — errores que además pueden ser aprovechados como fallas de seguridad graves.

---

## Análisis general

### Orientado a Objetos con herencia fuerte (Java, C++, C#, Spring)
El problema más común es el **abuso de la herencia y de las capas de abstracción**: se crean jerarquías de clases o interfaces más complejas de lo necesario. También aparecen fallas por **estado compartido y sincronización** (deadlocks en Java) o por **mecanismos "mágicos" que dependen de cómo se llama al código** (proxies de Spring que se rompen con llamadas internas).

### Tipado dinámico o débil (Python, JavaScript, Ruby, PHP)
El costo de la flexibilidad es que **los errores aparecen tarde**, solo cuando el código ya se está ejecutando, no antes. Además, como estos lenguajes permiten modificar el comportamiento del sistema en tiempo real (monkey patching en JS, metaprogramación en Ruby), se generan **efectos secundarios difíciles de rastrear** en toda la aplicación.

### Concurrencia y asincronía (Go, Node.js, Rust)
Cada uno resuelve la concurrencia distinto, y cada solución tiene su propia trampa:
- Go puede dejar **goroutines colgadas para siempre**.
- Node.js puede **congelar el único hilo disponible** si algo pesado bloquea el event loop.
- Rust evita bugs de memoria a costa de hacer **muy difíciles las estructuras con referencias cruzadas**.

### Funcional puro (Haskell)
Ganar pureza y previsibilidad tiene el precio de que las operaciones se acumulan sin ejecutarse (evaluación perezosa), lo que puede generar **fugas de memoria por cálculos pendientes** (*space leaks*) si no se fuerza la evaluación a tiempo.

### Frameworks reactivos/basados en componentes (Angular, React, Next.js)
El problema típico aquí es la **gestión del ciclo de vida**: suscripciones que nunca se cancelan (Angular), referencias que cambian en cada render y disparan renders innecesarios (React), o datos que no pueden viajar libremente entre "servidor" y "cliente" (Next.js).

### ORMs y acceso a datos (Rails, Django)
Ambos simplifican tanto el acceso a la base de datos que es fácil perder de vista **cuántas consultas SQL se están ejecutando realmente**: Rails mezcla lógica de negocio con persistencia (Fat Model) y Django puede disparar cientos de consultas sin que el desarrollador lo note (problema N+1).

### Declarativo lógico (Prolog) y bajo nivel (C)
Son paradigmas opuestos, pero ambos "confían mucho" en el programador: Prolog confía en que las reglas estén bien ordenadas (o la búsqueda explota), y C confía en que el programador maneje la memoria perfectamente (o hay corrupción y vulnerabilidades de seguridad).

---

## Conclusión

No existe un paradigma "perfecto"; cada uno **cambia un tipo de problema por otro**:

- La **Orientada a Objetos** da estructura y reutilización, pero puede volverse rígida y sobre-diseñada si se abusa de la herencia.
- El **tipado dinámico** da velocidad para escribir código, pero mueve los errores de "antes de correr el programa" a "mientras se está corriendo".
- La **concurrencia** (goroutines, event loop, hilos) da rendimiento, pero introduce nuevas formas de que algo se quede "atorado" o bloqueado.
- El **funcional puro** da seguridad y previsibilidad, pero exige entender bien cuándo se ejecutan realmente los cálculos.
- Los **frameworks modernos basados en componentes** dan productividad, pero esconden el ciclo de vida de las cosas (suscripciones, renders, memoria), y ese detalle escondido es justo donde aparecen los bugs.
