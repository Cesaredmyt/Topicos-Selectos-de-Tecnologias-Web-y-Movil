# Misión 1 y Misión 2 — Patrones de software para aplicaciones web

## Cesar Enrique Diaz Maldonado

**Tema:** Composición de patrones en un portal universitario (caso de estudio)

---

## Misión 1 — El portal no es un patrón

El portal no se describe con una sola etiqueta ("es MVC", "es hexagonal"). Cada síntoma del relato corresponde a un conflicto distinto, en una capa distinta, y se resuelve con una estructura distinta.

| # | Problema | Capa | Patrón(es) | Por qué ese y no el vecino | Cuándo no aplicaría |
|---|----------|------|------------|------------------------------|----------------------|
| 1 | Cada trámite es un archivo suelto, se copian sesión, bitácora y encabezado en decenas de archivos | Políticas transversales | **Front Controller + Chain of Responsibility** | La cadena puede **cortar** la petición si falla una validación (por ejemplo, sin sesión → 401). Decorator, en cambio, siempre envuelve y deja pasar, no está pensado para detener el flujo | Si solo hubiera 2 o 3 rutas fijas sin previsión de crecer, centralizar la entrada sería más ceremonia que beneficio |
| 2 | `pagar.php` es un switch de 200 líneas, el banco habla de "créditos" y códigos 00/01, el reglamento habla de estados propios | Aplicación/dominio + Integración | **Strategy** (elegir el método de cobro) **+ Adapter** (traducir el vocabulario del banco) | Factory decide **qué clase instanciar**, aquí el método ya fue elegido por el estudiante y lo que varía es el algoritmo de cobro. Facade simplifica un subsistema **propio**, aquí el protocolo es ajeno, por eso Adapter | Si existiera un solo método de pago sin previsión de otro, un `if` simple sería suficiente, armar una interfaz con una sola implementación es sobreingeniería |
| 3 | La plantilla del kardex ejecuta SQL, las constancias duplican esas consultas con otro formato | Presentación (servidor) + Datos | **Repository + Service Layer** | Nombrar "MVC" no dice qué falta, el problema puntual es que la vista habla SQL directamente. Repository nombra la pieza exacta que se necesita | Una consulta trivial usada una sola vez, sin duplicarse en ningún otro reporte, no justifica una capa de repositorio completa |
| 4 | Al acreditar el pago, el mismo script llama a control escolar, correo y caja, si el correo falla a veces no se registra el alta, el doble clic genera doble cargo | Aplicación/dominio + Datos + Integración | **Observer** (desacoplar notificaciones) **+ Unit of Work** (atomicidad) **+ idempotencia** | Observer resuelve **quién se entera** del cambio, no si el cambio fue atómico. Por eso no sustituye a Unit of Work, ambas fuerzas coexisten en el mismo párrafo del relato | Si solo existiera un interesado en el evento (sin correo ni caja de por medio), bastaría una llamada directa sin necesidad de Observer |
| 5 | La app pide un JSON mínimo, el kiosco pide HTML con escudo y tabla, hoy la app hace doce peticiones | Integración (borde) | **BFF** | El API Gateway aplica políticas **iguales** para todos los clientes (TLS, autenticación, cupo), aquí el conflicto es que cada cliente necesita una **agregación distinta** de datos, que es justo lo que resuelve BFF | Si todos los clientes necesitaran los mismos datos, un solo recurso REST bien agregado bastaría, sin necesidad de un servicio por cliente |
| 6 | El banco y el validador de CURP fallan seguido, mientras no responden, ni el kardex (que no depende de ellos) puede consultarse | Integración | **Circuit Breaker + Timeout + Bulkhead** | Reintentar (Retry) contra un servicio caído de forma sostenida no resuelve el problema, solo aumenta la carga. Bulkhead es la pieza que evita que el colapso de un colaborador arrastre a otro módulo que no depende de él | Si el colaborador externo fuera confiable y estuviera siempre disponible, instalar estas protecciones sería complejidad sin beneficio |

---

## Misión 2 — Una petición, varios patrones

Caso de uso: **pagar la inscripción**, desde el clic hasta la persistencia y notificación. El recorrido sigue el esquema de composición de una petición HTTP.

```
Cliente (MVC/MVVM/flujo unidireccional)
   │ representación HTTP
   ▼
Borde (API Gateway, BFF, Proxy, caché)
   │
   ▼
Front Controller
   │ Chain of Responsibility (autenticación, cupo, registro)
   ▼
Controlador de aplicación (Page Controller o recurso)
   │
   ▼
Service Layer (caso de uso)
   │ Strategy, Factory, Facade, Observer de dominio
   ▼
Repository + Unit of Work
   │ Adapter hacia el motor de persistencia o un sistema ajeno
   ▼
Almacén / servicio externo
```

| Paso | Mecanismo / objeto | Patrón que realiza |
|------|---------------------|----------------------|
| 1 | El cliente arma el estado de la pantalla de pago (folio, monto, método) | MVC / MVVM / flujo unidireccional |
| 2 | La petición sale como representación HTTP hacia el borde del sistema | — |
| 3 | En el borde se agregan datos según el tipo de cliente (app o kiosco), o se aplican políticas comunes de entrada | BFF (agregación por cliente) / API Gateway (políticas comunes) |
| 4 | La petición `POST /pagos` entra por el único punto de entrada del portal | Front Controller |
| 5 | Antes de llegar al controlador, se revisan sesión, cupo y bitácora, en orden, con posibilidad de cortar la petición | Chain of Responsibility (middleware) |
| 6 | El controlador deserializa el body y llama al caso de uso, sin decidir reglas de negocio | Page Controller (o recurso) |
| 7 | Se invoca el caso de uso `pagarInscripcion(estudianteId, metodo, monto)` | Service Layer |
| 8 | Dentro del Service Layer, se elige el algoritmo de cobro según el método (tarjeta, SPEI, ventanilla) | Strategy |
| 9 | Si el caso de uso orquesta varias operaciones internas (validar saldo, calcular cargo, verificar plazo) y se expone como una sola operación simple | Facade |
| 10 | Una vez confirmado el pago dentro de esta capa, se emite el evento correspondiente para quienes deban reaccionar | Observer de dominio |
| 11 | Se revisa una clave de idempotencia antes de ejecutar el cargo, para que un doble clic no genere doble cobro | Idempotencia (patrón de integración) |
| 12 | El cargo y el alta de materias se persisten como una sola transacción, mediante una colección abstracta del dominio | Repository + Unit of Work |
| 13 | La escritura o consulta hacia el motor de base de datos, o la comunicación con el banco externo, se traduce en su vocabulario propio | Adapter |
| 14 | El dato queda registrado en el almacén, o el cobro queda procesado por el servicio externo | Almacén / servicio externo |
| 15 | Si el banco o el validador de CURP tardan o fallan de forma reiterada, la espera se acota y se deja de insistir sin bloquear el resto del sistema | Timeout + Circuit Breaker |
| 16 | La respuesta se arma en el formato que corresponde al cliente (JSON mínimo o HTML completo), con datos ya calculados | Template / Transform View |

---
