# EXAMEN — MICROSERVICIOS, GATEWAY Y EL MONOLITO CON API

**Tópicos Selectos de Tecnologías Web y Móvil**
**Eduardo Alcaraz**

**Nombre:** Cesar Enrique Diaz Maldonado  **Calificación:** _________

---

## Indicaciones

Este examen *no* se aprueba de memoria. Casi todas las preguntas tienen una respuesta que «suena bien» y está mal, o dos opciones defendibles donde se califica el argumento.

**Reglas:**

- Expliquen el *porqué*. Una conclusión sin razonamiento vale la mitad (o menos).
- Cuando digan «depende», digan *de qué*.
- No hace falta código que compile; sí hace falta que el diseño cierre (quién llama a quién, qué pasa si algo falla, qué ve el cliente).
- Pueden dibujar. Un dibujo contradictorio con el texto resta.

**Puntaje:** *100 puntos* (10 × 10).

---

## 1 — La frase del compañero (10 puntos)

En el pizarrón quedó:

> La API separa clientes del servidor; los microservicios separan servidores entre sí.

Un compañero dice: «Entonces, si mi monolito Django expone una API y además lo meto en Docker, ya cumplí las dos mitades de la frase.»

> **a)** Desarmen su argumento. ¿Qué parte de la frase cree cumplir con la API? ¿Qué parte cree cumplir con Docker? ¿En qué se equivoca en cada una?

**Respuesta:**

Exponer una API cumple la primera mitad de la frase, y eso está bien dicho: la API separa lo que ve el cliente (el front) del back, para eso sirve. Pero ahí se queda, es nada más el contrato entre cliente y servidor y no dice cuántos servicios hay detrás. Con Docker se cree cumplida la segunda mitad (“separar servidores entre sí”), y ahí está el error, porque Docker solo mete un proceso en su propio entorno aislado. Si adentro del contenedor sigue todo el Django junto, no hay varios servicios distribuidos, es uno solo metido en una caja.

> **b)** Den un contraejemplo mínimo: mismo código Django, *con* API y *con* Docker, que *sigamos* pudiendo llamar monolito. ¿Qué tendría que añadir (o partir) para que deje de serlo?

**Respuesta:**

El mismo Django, con API y con Docker, sigue siendo monolito mientras todos los módulos estén en la misma imagen, corran en el mismo proceso y se desplieguen juntos, porque se siguen importando unos a otros. Para que deje de serlo, habría que separar esos módulos en imágenes distintas, de modo que cada uno corra como proceso independiente, se despliegue solo y pueda fallar sin tumbar a los demás. En vez de importarse, se conectarían con una llamada por red al servicio que se separó. Ahí ya hay servidores distintos que se hablan entre sí, que es lo que dice la segunda mitad de la frase.

> **c)** ¿La frase del pizarrón habla de *tecnologías* (Docker, REST) o de *límites de proceso / despliegue*? Defiendan su lectura con lo que vieron en clase.

**Respuesta:**

La frase habla de **límites de proceso y despliegue**, no de tecnologías. Docker y REST se pueden usar tanto en un monolito como en microservicios, así que no definen la arquitectura. Lo que la define es qué corre por separado, se despliega por separado y se comunica por red. Además, usar microservicios depende del tamaño del proyecto, de cuánto va a crecer y de los recursos disponibles, porque si no hacen falta solo añaden complejidad.

---

## 2 — Tres rutas, una trampa (10 puntos)

El equipo muestra este `urls.py` de un solo proyecto Django:

```python
path("usuarios/", ...),
path("pedidos/", ...),
path("recomendar/", ...),
```

y este `docker-compose.yml`:

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]
    deploy:
      replicas: 3
```

Afirman: «Tres rutas + tres réplicas = microservicios con alta disponibilidad.»

> **a)** Separen las dos ideas que mezclaron (rutas vs réplicas). ¿Qué problema resuelve cada una y cuál *no* resuelve?

**Respuesta:**

Las **rutas** solo organizan el acceso a distintas partes del proyecto. No resuelven la separación de servicios ni la disponibilidad, porque siguen siendo un solo Django. Las **réplicas** resuelven la disponibilidad y reparten la carga, pero no la separación, porque son la misma copia corriendo varias veces en puertos distintos. Microservicios sería que cada pieza corra, se despliegue y falle por separado. Aquí se confunde porque si una réplica se cae las otras siguen funcionando y parece lo mismo, pero no es así.

> **b)** Si una de las tres réplicas se cae a mitad de un `POST /pedidos`, ¿el sistema dejó de ser monolito? ¿Por qué sí o por qué no?

**Respuesta:**

No, sigue siendo monolito. Que se caiga una réplica es un tema de disponibilidad, no de arquitectura. La acción se cancela y se reintenta en otra réplica, pero todas tienen el mismo código completo desplegado junto, así que nada se separó.

> **c)** Rediseñen *solo con palabras* el `compose` para que sí ilustre microservicios (nombres de servicios y responsabilidades; no hace falta YAML perfecto). ¿Cuántas *imágenes distintas* mínimo y por qué esa cantidad?

**Respuesta:**

Haría falta un **gateway** que reciba todas las peticiones y las mande a donde corresponde, y un servicio por separado para cada cosa: **usuarios** (cuentas), **pedidos** (crear y gestionar pedidos) y **recomendar** (dar la recomendación). Mínimo **4 imágenes distintas** (gateway, usuarios, pedidos y recomendar), porque cada servicio tiene su propio código y dependencias y debe construirse y desplegarse por separado. Si fuera la misma imagen repetida, seguirían siendo copias de un monolito.

---

## 3 — El gateway que «hace de más» (10 puntos)

Les proponen este diseño «para no complicar»:

> Un solo contenedor nginx+Python que: valida el JWT, elige el medio con el `.joblib`, guarda el pedido en SQLite, manda el correo y devuelve el HTML.

Lo llaman «API Gateway».

> **a)** ¿Es un gateway? Si no, ¿qué es? Usen la analogía del recepcionista (o otra) y digan qué oficios mezclaron.

**Respuesta:**

No es un gateway, es un monolito disfrazado. Un gateway de verdad solo se encarga de la entrada: recibe a quien llega, checa que sea quien dice ser y lo manda a donde corresponde. Es como un recepcionista, que te recibe, revisa tu gafete y te dice a qué oficina ir. Aquí el recepcionista además hace los cálculos, guarda los pedidos y manda los correos. O sea, mezcla un montón de trabajo distinto: seguridad, elegir el medio de transporte con el modelo, guardar los datos, avisar por correo y mostrar la página.

> **b)** Mencionen *tres* consecuencias malas de ese diseño cuando el modelo pesado se reentrena cada semana y el correo empieza a fallar.

**Respuesta:**

1. Cada semana que se actualiza el modelo hay que apagar y volver a prender todo junto, y mientras tanto nada funciona, ni la entrada.
2. Si el correo falla, puede trabar o tumbar el pedido completo, aunque el pedido ya estuviera bien, porque todo va revuelto en el mismo lugar.
3. No se puede arreglar ni hacer crecer una cosa sin mover todo lo demás. El modelo es pesado y la entrada debería ser ligera, pero como van juntos, una falla en una afecta a todas.

> **c)** Partan ese contenedor en piezas. Para cada pieza digan: ¿queda en el borde (gateway) o detrás? No inventen diez servicios: el mínimo que separe borde, logística e (si hace falta) aviso.

**Respuesta:**

- **Gateway:** revisa quién entra, muestra la página y manda las peticiones hacia adentro. Se queda en la entrada.
- **Logística (pedidos + modelo):** elige el medio de transporte y guarda el pedido. Va atrás del gateway.
- **Aviso:** manda el correo. Va atrás también, y si falla no debe afectar al pedido.

---

## 4 — `localhost`, la trampa favorita (10 puntos)

Pedidos corre en un contenedor. Recomendar en otro, misma Compose. Alguien «arregla» la conexión así:

```
RECOMENDAR_URL=http://127.0.0.1:8002
```

En su laptop, *sin* Docker, ese `curl` a `127.0.0.1:8002` sí funcionaba.

> **a)** Expliquen por qué en Compose ese arreglo suele fallar *aunque* Recomendar esté sano. ¿A qué proceso apunta `127.0.0.1` desde dentro de Pedidos?

**Respuesta:**

Cada contenedor tiene su propia red interna, así que `127.0.0.1` desde dentro de Pedidos apunta al mismo contenedor de Pedidos, no a la laptop ni a Recomendar. Como ahí en el puerto 8002 no hay nada escuchando, la conexión se rechaza aunque Recomendar esté sano en otro contenedor. En la laptop sin Docker sí funcionaba porque todo corría en la misma máquina.

> **b)** ¿Por qué `http://recomendar:8002` sí tiene chance de funcionar? ¿Quién «inventa» el nombre `recomendar`?

**Respuesta:**

Porque `recomendar` es el nombre del servicio, y ese nombre lo “inventa” Docker Compose: crea una red interna y le pone a cada servicio un nombre igual al que se definió en el `docker-compose.yml`. Así, Pedidos puede encontrar a Recomendar por ese nombre, siempre que estén en la misma red de Compose.

> **c)** Ahora Recomendar se muda a la laptop de otro alumno (`192.168.0.57`). Escriban la URL que pondrían *y* dos cosas que tienen que ser verdad en la WiFi para que no sea teatro (puertos, firewall, etc.). Si solo cambian la URL y nada más, ¿qué puede seguir fallando?

**Respuesta:**

La URL sería `http://192.168.0.57:8002`. Para que funcione de verdad, tienen que cumplirse dos cosas:

1. Que Recomendar esté publicado en el puerto 8002 y escuchando hacia afuera, no solo en el localhost de esa laptop.
2. Que el firewall de esa laptop deje pasar el puerto 8002 y que la WiFi permita que los equipos se vean entre sí, estando todos en la misma red.

Si solo se cambia la URL y nada más, puede seguir fallando porque el firewall bloquea el puerto, porque el servicio solo escucha en localhost, porque la WiFi aísla a los equipos o porque la IP cambió (la asigna el router y puede ser otra después).

---

## 5 — Fallo parcial con números (10 puntos)

Escenario medido en laboratorio (inventado pero coherente):

- 1000 `POST /pedidos` por hora en promedio.
- Recomendar tarda 200 ms cuando está bien.
- El 5 % del tiempo Recomendar no responde (timeout 3 s).
- Pedidos, si hay timeout, usa plan B (moto) y responde 200 en 50 ms.
- El gateway añade 5 ms.

> **a)** En la hora «normal» (Recomendar bien), ¿el cliente espera ~200 ms o ~3 s? ¿Quién pone el tiempo dominante?

**Respuesta:**

El cliente espera ~200 ms, y el tiempo dominante lo pone Recomendar, porque es la llamada más lenta de todas. Los 3 s del timeout no cuentan aquí, solo aplican cuando Recomendar no responde.

> **b)** En el 5 % malo, con plan B: ¿el cliente ve error o pedido creado? ¿Qué «mentira útil» están aceptando (medio quizá no óptimo)?

**Respuesta:**

El cliente ve pedido creado, no error, porque en vez de rechazar la petición se usa el plan B (la moto). Pero espera unos 3 s, porque primero tiene que terminar el timeout y luego se responde en 50 ms. La mentira útil es que el pedido sale bien aunque el medio quizá no sea el óptimo, solo el más equilibrado.

> **c)** Un compañero dice: «Mejor que Pedidos *reintente* 10 veces hasta que Recomendar vuelva; así nunca usamos plan B.» ¿Qué le pasa a la cola de peticiones y al gateway si Recomendar está caído 10 minutos? ¿Plan B siempre es peor que reintentar? Argumenten.

**Respuesta:**

Reintentar 10 veces no conviene. Si Recomendar está caído 10 minutos, cada petición queda colgada esperando y las que llegan se van acumulando. La cola se llena de peticiones que no van a salir y el gateway también se satura, así que se afectan hasta las peticiones que sí se podían atender rápido. El plan B no siempre es peor: a veces la moto incluso es la respuesta correcta, y no hay garantía de que Recomendar regrese, así que es mejor responder con algo razonable que dejar todo trabado.

---

## 6 — Apagan el «edificio equivocado» (10 puntos)

Tres contenedores: gateway, pedidos, recomendar. Hacen tres experimentos *por separado*:

1. `docker compose stop gateway`
2. `docker compose stop pedidos`
3. `docker compose stop recomendar`

Para *cada* experimento digan qué ve el navegador en `http://IP:8080/` (página / crear pedido) y qué sigue vivo por dentro. No basta «no jala»: digan *qué* no jala y qué sí podría seguir respondiendo si alguien hablara por dentro de la red.

Luego: ¿en cuál de los tres el sistema «parece» más muerto hacia afuera aunque más piezas sigan arriba? ¿Qué les enseña eso del borde?

**Respuesta:**

**1) Se apaga el gateway.** El navegador no puede ni cargar la página, sale error de que no se puede acceder. Por dentro, pedidos y recomendar siguen vivos y responderían si alguien les hablara directo desde la red interna, pero nadie de afuera llega a ellos.

**2) Se apaga pedidos.** La página sí carga, porque la sirve el gateway. El problema aparece al crear un pedido, ahí sale error de que no se encuentra el servicio o de timeout. Por dentro siguen vivos el gateway y recomendar, pero como recomendar se usa desde pedidos, tampoco se llega a usar.

**3) Se apaga recomendar.** Es el caso más favorable. La página carga y el pedido se crea con el plan B, aunque tarda unos 3 s por el timeout y quizá el medio no sea el mejor. Por dentro siguen vivos gateway y pedidos.

**El que más muerto parece hacia afuera es el 1**, aunque sean más las piezas que siguen arriba. Eso enseña que el borde es la puerta de todo: si cae, el sistema se ve caído aunque por dentro esté sano, por eso conviene cuidarlo más que al resto.

---

## 7 — WiFi del salón, conflicto de roles (10 puntos)

Cuatro alumnos:

- Ana: gateway + HTML
- Beto: pedidos
- Carla: recomendar
- Diana: «yo también levanté un gateway en mi laptop por si Ana falla»

Todos en la misma WiFi. El cliente (otro equipo) configura a veces la IP de Ana y a veces la de Diana.

> **a)** ¿Qué problema de *diseño* introdujo Diana aunque «todo responda 200»? (Piensen en una sola puerta vs dos puertas.)

**Respuesta:**

El problema es que ya no hay una sola puerta de entrada. Con dos gateways, el cliente tiene que escoger a cuál llamarle y no hay un solo punto donde aplicar seguridad y revisar qué pasa. Si no están configurados igual, el mismo cliente recibe respuestas distintas según la puerta que use, aunque todo responda 200. Lo ideal es una sola puerta como principal y la otra de respaldo, pero detrás de una sola dirección para que el cliente no tenga que escoger.

> **b)** Beto configuró `RECOMENDAR_URL=http://localhost:8002` porque en su máquina desarrolló Pedidos y Recomendar juntos. En el salón Recomendar está en la laptop de Carla. ¿Qué falla y cómo lo corrigen *sin* juntar otra vez los dos servicios en la laptop de Beto?

**Respuesta:**

Falla porque `localhost` en la laptop de Beto apunta a su propia máquina, y ahí ya no está Recomendar. Se corrige cambiando la URL a la IP de la laptop de Carla con el puerto, algo como `http://IP_DE_CARLA:8002`. Conviene que sea una variable de configuración para cambiarla sin tocar el código, y que el puerto esté abierto y las laptops en la misma red. Así no hace falta juntar otra vez los dos servicios.

> **c)** La WiFi del campus aísla clientes («AP isolation»). `ping` entre alumnos falla. ¿Pueden hacer la práctica tal cual? Si no, propongan *un* plan B de laboratorio que preserve la lección «procesos separados» (aunque no sea WiFi del salón).

**Respuesta:**

No se puede hacer tal cual, porque con el aislamiento los equipos no se pueden ver entre sí y los servicios no se podrían hablar. Un plan B es armar una red propia, por ejemplo con el hotspot del celular de un compañero o un router aparte, y conectar todos ahí. Así se conserva lo importante: procesos separados que se hablan por red.

---

## 8 — El front que conoce demasiado (10 puntos)

El HTML hace esto al enviar el formulario:

```js
await fetch("http://192.168.1.20:8002/sugerir", { ... }); // Recomendar
await fetch("http://192.168.1.30:8001/pedidos", { ... }); // Pedidos
```

El autor dice: «Más rápido: me salto el gateway.»

> **a)** Listen *tres* problemas (seguridad, cambio de IP, orden de llamadas, consistencia del folio, CORS, etc.).

**Respuesta:**

1. **Seguridad:** el navegador se brinca la validación del JWT, y las IPs internas quedan a la vista, así que cualquiera podría llamar directo a los servicios.
2. **Cambio de IP:** las IPs están escritas en el front, y si cambian hay que modificarlo y volverlo a publicar.
3. **Orden y consistencia:** el navegador hace de coordinador de dos llamadas, y si la primera sale bien y la segunda falla, el pedido queda a medias.

> **b)** Rediseñen la secuencia correcta con gateway: *una* llamada desde el navegador. ¿Quién habla con Recomendar y por qué no el navegador?

**Respuesta:**

El navegador hace una sola llamada al gateway. El gateway valida el JWT y pasa la petición a Pedidos. Pedidos le habla a Recomendar (con timeout y plan B), guarda el pedido y la respuesta regresa por el gateway. El navegador no le habla a Recomendar porque no debe conocer los servicios internos ni manejar sus fallas, eso le toca a Pedidos.

> **c)** El selector de modelo (`arbol=|=bosque=|=red`) va en el JSON. Si el front llama directo a Recomendar *y* a Pedidos, ¿quién debería crear el folio para no duplicar lógica? ¿Qué invariante rompen si los dos «crean» algo?

**Respuesta:**

El folio lo debe crear Pedidos, porque es el dueño del pedido, y Recomendar solo recomienda. Si los dos crean algo, se rompe la regla de que cada pedido tenga un solo folio y una sola fuente de verdad, y salen folios duplicados o distintos para el mismo pedido.

---

## 9 — El LLM vendedor (10 puntos)

Producto: tienda + envíos.

- Modelo `.joblib`: medio de transporte.
- LLM: texto para convencer al cliente en la ficha del producto.

Un arquitecto propone *un solo* microservicio `cerebro` que: recibe el carrito, elige el medio con el árbol *y* genera el pitch de ventas con el LLM, todo en la misma petición `/inteligente`.

> **a)** Critiquen el nombre y el corte. ¿Qué dos oficios distintos mezclaron? ¿Qué falla cuando el LLM tarda 8 s y el despacho necesita responder en < 500 ms?

**Respuesta:**

Mezclaron dos oficios distintos: elegir el medio de transporte con el modelo (logística) y escribir el texto de venta con el LLM (marketing). Si el LLM tarda 8 s y el despacho necesita responder en menos de 500 ms, como van en la misma petición, el despacho se queda esperando y se pasa por mucho. Además, si el LLM falla, también se cae la elección del medio, aunque no tengan nada que ver. Y el nombre `cerebro` es poco descriptivo, no dice qué hace.

> **b)** Propongan dos endpoints (o dos servicios) y digan cuál usa el *front de tienda* y cuál el *flujo de pedido/envío*.

**Respuesta:**

Dos servicios: `recomendar` (el modelo, rápido), que lo usa el flujo de pedido/envío, y `pitch` (el LLM, lento), que lo usa el front de la tienda para la ficha del producto. Así cada uno tiene su propio tiempo, plan B y escalado.

> **c)** El LLM vive en un server con GPU en otra red. El front de tienda ¿apunta a la GPU o al gateway? Si mañana cambian de proveedor LLM (otro host), ¿qué archivo/config quieren tocar *una sola vez*?

**Respuesta:**

El front apunta al gateway, no a la GPU, porque no debe conocer la dirección del LLM y es el gateway quien valida y manda la petición hacia adentro. Si cambian de proveedor, se toca una sola vez la configuración del servicio `pitch`, la variable con la URL del LLM (en el `.env` o el `docker-compose.yml`). El front y el gateway no cambian, ni saben que cambió algo.

---

## 10 — El inversionista y la semana imposible (10 puntos)

Hoy:

- Panel interno: monolito Django (funciona, poco tráfico).
- App clientes: gateway + pedidos + recomendar.
- Idea: LLM de ventas (aún no).
- Equipo: 5 personas.
- Inversionista: «Todo a microservicios en 7 días, incluido el panel, o no hay dinero.»

> **a)** Escriban la respuesta al inversionista (máx. 8 líneas) con postura *y* riesgo concreto si obedecen a ciegas.

**Respuesta:**

El panel interno funciona bien, tiene poco tráfico y no depende de otros servicios, así que partirlo en una semana no nos da ningún beneficio. Si lo hacemos a ciegas con 5 personas, el riesgo es dejar el panel roto o a medias y meter problemas nuevos de red y de fallos que hoy no tenemos. Preferimos invertir esa semana en lo que sí lo necesita: la app de clientes, donde está el tráfico. Ahí sí aplicamos microservicios y vamos a reforzar seguridad y estabilidad.

> **b)** Ordenen *estas cuatro* iniciativas de más a menos urgente para *esta* semana y justifiquen el orden (no el catálogo soñado):
>
> 1. Partir el panel Django en 6 microservicios.
> 2. Sacar el reentrenamiento del `.joblib` a un pipeline que no tumbe Pedidos.
> 3. Meter auth + rate limit en el gateway público.
> 4. Subir el LLM de ventas a producción sin timeouts ni plan B.

**Respuesta:**

1. **Iniciativa 3 (auth + rate limit en el gateway):** es seguridad básica, y el gateway público está expuesto a atacantes.
2. **Iniciativa 2 (sacar el reentrenamiento):** el modelo se reentrena cada semana y Pedidos no puede apagarse cada vez, debe estar siempre disponible.
3. **Iniciativa 4 (LLM):** solo después de pensarlo bien, con timeouts y plan B, porque así como está puede tumbar todo.
4. **Iniciativa 1 (partir el panel):** es lo menos urgente, porque es largo, no cabe en una semana y el panel ya funciona.

> **c)** Inventen *una* métrica que les diga «ya toca partir X del monolito» (no «porque está de moda»). Tiene que ser observable (tiempo, errores, despliegues, equipo…).

**Respuesta:**

Si un mismo módulo se redespliega más de 3 veces por semana y cada vez tumba o frena a los demás, ya toca partirlo. Se mide fácil con el historial de despliegues y el tiempo caído después de cada uno. Otra forma de verlo es que un solo módulo consuma más del 50 % del CPU o de los errores del sistema y haya que escalarlo aparte, porque ahí el costo de tenerlo junto ya es mayor que el de separarlo.

> **d)** Cierren con una frase propia (no copien el apunte) que capture: ¿microservicios son el objetivo o la herramienta?

**Respuesta:**

Los microservicios son una herramienta, no el objetivo. No siempre conviene migrar, y hacerlo solo porque está de moda nada más complica el sistema. Sirven cuando el sistema o el equipo crece mucho, o se sabe que va a crecer, y ahí sí valen la pena.
