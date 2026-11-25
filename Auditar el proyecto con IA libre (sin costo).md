# Auditar el proyecto con IA libre (sin costo)

Proyecto 1 — Mascotas / veterinaria
Cesar Enrique Díaz Maldonado — 22121331

## Herramientas libres o sin costo

Todo en mi computadora con Ollama, nada salió a internet.

| IA | Modelo | Tipo |
| --- | --- | --- |
| A | `gemma3:4b` | general |
| B | `gemma4:e4b` | general |
| C | `qwen2.5-coder:7b` | de código |

## Paso 1 — Armar un paquete de contexto (sin secretos)

Revisé que no hubiera `.env`, `key.properties` ni llaves, y no hay.

Este es el paquete completo que le pasé a las IAs:

<details>
<summary>Paquete de contexto completo (clic para abrir o cerrar)</summary>

````
# Paquete de contexto para la auditoría (sin secretos)

## Dominio y arquitectura
Proyecto individual de Tópicos Web y Móvil: veterinaria. Un recomendador decide el nivel de atención de una mascota (leve, normal, prioritaria, urgente) con un modelo .joblib (scikit-learn) y reglas de seguridad que solo pueden subir el nivel.

Arquitectura: microservicios con gateway. Flutter (web y móvil, mismo código) solo habla con el gateway nginx (:8080).
- gateway (nginx): manda /mascotas/recomendar/ al servicio recomendar y el resto de /mascotas/ al servicio pedidos; /salud/pedidos y /salud/recomendar revisan cada servicio; sirve la app web si está construida.
- pedidos (Django + PostgreSQL con Docker, SQLite sin Docker): catálogo, solicitudes con folio (MAS-XXXX) y asistente (clasificador de intenciones). No tiene el modelo de urgencia: se lo pide a recomendar por HTTP (3 s de espera). Si falla, plan B: guarda la solicitud con nivel "urgente" y plan_b=true.
- recomendar (Flask + gunicorn): tiene los .joblib (arbol, bosque, red) y las reglas. Valida los 8 campos.
- db: PostgreSQL 16.

## Endpoints (todos por el gateway)
- GET /mascotas/productos/?q=  | GET /mascotas/productos/<id>/
- POST /mascotas/recomendar/  (lo contesta recomendar, no guarda nada)
- POST /mascotas/pedidos/  (pedidos llama a recomendar, guarda, devuelve folio; 503 con folio si plan B)
- GET /mascotas/pedidos/<folio>/
- POST /mascotas/asistente/
- GET /salud/pedidos, GET /salud/recomendar

## Árbol de carpetas
backend/docker-compose.yml
backend/gateway/nginx.conf
backend/pedidos/ (manage.py, plataforma/settings.py, mascotas/{models,repositorio,caso,cliente_recomendar,asistente,views,urls,tests}.py, mascotas/modelo/modelo_asistente.joblib)
backend/recomendar/ (app.py, caso.py, reglas.py, recomendador.py, tests.py, modelo/*.joblib)
backend/dataset-urgencia/, backend/dataset-asistente/ (scripts de entrenamiento)
app/lib/ (main.dart, tema.dart, modelos/, servicios/api_mascotas.dart, pantallas/, widgets/)
docs/ (documentación)

## Fragmento 1: backend/gateway/nginx.conf
server {
    listen 80;

    # DNS interno de Docker. Con esto nginx busca cada servicio en cada petición y no al arrancar:
    # si "recomendar" o "pedidos" están apagados, el gateway sigue vivo y solo esa ruta falla.
    resolver 127.0.0.11 valid=5s ipv6=off;

    set $pedidos http://pedidos:8000;
    set $recomendar_sugerir http://recomendar:8002/sugerir;
    set $pedidos_salud http://pedidos:8000/mascotas/salud/;
    set $recomendar_salud http://recomendar:8002/salud;

    # La app Flutter web (flutter build web) se sirve desde aquí. Si no está construida, solo falta esta página.
    root /usr/share/nginx/html;
    index index.html;

    # Si un servicio no contesta, el gateway se rinde pronto.
    proxy_connect_timeout 2s;

    # CORS: Flutter en Chrome (flutter run) corre en otro puerto que el gateway, y sin estos
    # encabezados el navegador bloquea la respuesta. "always" los pone también en los errores.
    add_header Access-Control-Allow-Origin "*" always;
    add_header Access-Control-Allow-Methods "GET, POST, OPTIONS" always;
    add_header Access-Control-Allow-Headers "Content-Type" always;

    # Cuando el servicio no existe o no contesta, el cliente recibe un JSON claro y no una página de nginx.
    error_page 502 504 /_servicio_caido;
    location = /_servicio_caido {
        internal;
        default_type application/json;
        return 502 '{"error": "un servicio no responde (gateway)"}';
    }

    location / {
        try_files $uri /index.html;
    }

    # Solo el recomendador: el servicio recomendar tiene el modelo y las reglas.
    # El cliente no llama a la IP de ningún servicio, solo a esta puerta.
    location = /mascotas/recomendar/ {
        # El navegador pregunta con OPTIONS antes del POST; se contesta aquí.
        if ($request_method = OPTIONS) {
            return 204;
        }
        proxy_pass $recomendar_sugerir;
    }

    # Todo lo demás de la API (catálogo, solicitudes con folio, asistente) es del servicio pedidos.
    location /mascotas/ {
        if ($request_method = OPTIONS) {
            return 204;
        }
        proxy_pass $pedidos;
        # Django responde con su propio CORS (para cuando se corre sin gateway); aquí lo pone el gateway.
        proxy_hide_header Access-Control-Allow-Origin;
        proxy_hide_header Access-Control-Allow-Methods;
        proxy_hide_header Access-Control-Allow-Headers;
        # Django solo acepta el nombre "pedidos" (ALLOWED_HOSTS), no importa desde dónde llegue el cliente.
        proxy_set_header Host pedidos;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Para revisar que cada servicio responde (la app los muestra como pastillas).
    location = /salud/pedidos {
        proxy_pass $pedidos_salud;
        proxy_hide_header Access-Control-Allow-Origin;
        proxy_hide_header Access-Control-Allow-Methods;
        proxy_hide_header Access-Control-Allow-Headers;
        proxy_set_header Host pedidos;
    }

    location = /salud/recomendar {
        proxy_pass $recomendar_salud;
    }
}

## Fragmento 2: backend/docker-compose.yml
# Cuatro piezas, cada una en su contenedor:
#   gateway    nginx, la única puerta de entrada (puerto 8080)
#   pedidos    Django: catálogo, solicitudes con folio y asistente. Guarda en PostgreSQL
#   recomendar Flask: el modelo (.joblib) y las reglas de seguridad. No toca la base de datos
#   db         PostgreSQL
# Además "migrar" corre una sola vez: crea las tablas y carga los productos de prueba.

x-django: &django
  build: ./pedidos
  image: mascotas-pedidos
  environment:
    DJANGO_DEBUG: "0"
    DJANGO_SECRET_KEY: cambiar-en-produccion
    DJANGO_ALLOWED_HOSTS: pedidos,localhost,127.0.0.1
    POSTGRES_HOST: db
    POSTGRES_DB: mascotas
    POSTGRES_USER: mascotas
    POSTGRES_PASSWORD: mascotas
    RECOMENDAR_URL: http://recomendar:8002

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mascotas
      POSTGRES_USER: mascotas
      POSTGRES_PASSWORD: mascotas
    volumes:
      - datos:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mascotas"]
      interval: 2s
      retries: 30

  migrar:
    <<: *django
    command: sh -c "python manage.py migrate --noinput && python manage.py loaddata productos.json"
    depends_on:
      db:
        condition: service_healthy

  pedidos:
    <<: *django
    depends_on:
      migrar:
        condition: service_completed_successfully

  recomendar:
    build: ./recomendar

  gateway:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    volumes:
      - ./gateway/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      # La app Flutter web, si se construyó con "flutter build web" (opcional).
      - ../app/build/web:/usr/share/nginx/html:ro
    depends_on:
      - pedidos
      - recomendar

volumes:
  datos:

## Fragmento 3: backend/pedidos/mascotas/views.py (crear_pedido y plan B)
import json
from dataclasses import asdict

from django.conf import settings
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_GET, require_POST

from . import cliente_recomendar, repositorio
from .asistente import responder
from .caso import leer_caso

# Si recomendar no responde, el nivel más alto nunca queda por debajo de lo que exigen las reglas
# (igual que en el proyecto de entregas la camioneta es el medio que nunca rompe una regla).
NIVEL_POR_DEFECTO = "urgente"


def _json(datos, status=200):
    return JsonResponse(datos, status=status, json_dumps_params={"ensure_ascii": False})

def crear_pedido(request):
    caso, modelo, error = leer_caso(request.body)
    if error:
        return _json({"servicio": "pedidos", **error}, 400)

    recorrido = ["gateway", "pedidos"]

    try:
        sugerencia = cliente_recomendar.sugerir(caso, modelo)
    except cliente_recomendar.RecomendarNoDisponible as causa:
        return _plan_b(caso, modelo, recorrido, str(causa))
    except cliente_recomendar.RecomendarRechazo as rechazo:
        # El caso viene mal armado: es error de quien lo mandó. El plan B escondería el problema,
        # así que NO se crea la solicitud.
        return _error("recomendar rechazó el caso", rechazo.estado, detalle=rechazo.detalle)

    resultado = {**asdict(sugerencia), "plan_b": False}
    pedido = repositorio.crear_pedido(caso, modelo, resultado)
    respuesta = _pedido_a_dict(pedido)
    respuesta.update({
        "servicio": "pedidos",
        "modelo_nombre": sugerencia.modelo_nombre,
        "razones_reglas": sugerencia.razones_reglas,
        "ranking": sugerencia.ranking,
        "proceso": sugerencia.proceso,
        "recorrido": recorrido + ["recomendar"],
    })
    return _json(respuesta, 201)


def _plan_b(caso, modelo, recorrido, causa):
    motivo = f"{causa}; se asignó el nivel {NIVEL_POR_DEFECTO} por precaución, sin evaluar el caso"
    pedido = repositorio.crear_pedido(caso, modelo, {
        "nivel": NIVEL_POR_DEFECTO,
        "confianza": 0.0,
        "corregido_por_reglas": False,
        "preferido_por_modelo": "",
        "motivo": motivo,
        "plan_b": True,
    })
    respuesta = _pedido_a_dict(pedido)
    respuesta.update({
        "servicio": "pedidos",
        "error": "el servicio recomendar no estuvo disponible",
        "recorrido": recorrido + ["recomendar (sin respuesta)"],
    })
    # 503 avisa que recomendar falló; la solicitud sí quedó registrada, con su folio.
    return _json(respuesta, 503)


@require_GET
def pedido(request, folio):

## Fragmento 4: backend/pedidos/mascotas/cliente_recomendar.py (sugerir)
def sugerir(caso, modelo):
    """Le pide a recomendar el nivel de atención. Lanza RecomendarNoDisponible o RecomendarRechazo."""
    cuerpo = {**caso, "modelo": modelo}
    try:
        respuesta = requests.post(
            f"{settings.RECOMENDAR_URL}/sugerir",
            json=cuerpo,
            timeout=settings.RECOMENDAR_ESPERA_SEGUNDOS,
        )
    except requests.RequestException as error:
        raise RecomendarNoDisponible("recomendar no respondió a tiempo") from error

    if respuesta.status_code >= 500:
        raise RecomendarNoDisponible(f"recomendar respondió con error {respuesta.status_code}")

    try:
        datos = respuesta.json()
    except ValueError as error:
        raise RecomendarNoDisponible("recomendar respondió algo que no es JSON") from error

    if respuesta.status_code != 200:
        raise RecomendarRechazo(respuesta.status_code, datos)

    try:
        return Sugerencia.desde_json(datos)
    except KeyError as error:
        raise RecomendarNoDisponible(f"recomendar respondió sin el campo {error}") from error

## Fragmento 5: app/lib/servicios/api_mascotas.dart (crearPedido y URL base)
import '../modelos/recomendacion.dart';

// La app solo habla con el gateway (una sola puerta). Nunca con la IP de pedidos o recomendar.
// Chrome: 127.0.0.1 | emulador Android: 10.0.2.2 | celular físico: IP de la PC en el WiFi.
// Se cambia al correr: flutter run --dart-define=URL_BASE=http://10.0.2.2:8080
const urlBase = String.fromEnvironment('URL_BASE', defaultValue: 'http://127.0.0.1:8080');

class ErrorApi implements Exception {
Future<Recomendacion> crearPedido(Caso caso, String modelo) async {
  final (estado, datos) = await _llamarCrudo(() => http.post(
        Uri.parse('$urlBase/mascotas/pedidos/'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode(caso.toJson(modelo: modelo)),
      ));
  final planB = estado == 503 && datos['plan_b'] == true;
  if (estado == 201 || planB) {
    return Recomendacion.fromJson(datos);
  }
  throw ErrorApi('Error $estado: ${datos['error'] ?? 'respuesta inesperada'}');
}

## Fragmento 6: backend/pedidos/plataforma/settings.py (hosts, base de datos)
# Quick-start development settings - unsuitable for production
# See https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "solo-para-desarrollo")
DEBUG = os.environ.get("DJANGO_DEBUG", "1") == "1"
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")

# Dónde está el servicio recomendar (en Compose, su nombre de servicio) y cuánto se le espera.
RECOMENDAR_URL = os.environ.get("RECOMENDAR_URL", "http://localhost:8002")
RECOMENDAR_ESPERA_SEGUNDOS = 3
if os.environ.get("POSTGRES_HOST"):
    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.postgresql",
            "HOST": os.environ["POSTGRES_HOST"],
            "NAME": os.environ["POSTGRES_DB"],
            "USER": os.environ["POSTGRES_USER"],
            "PASSWORD": os.environ["POSTGRES_PASSWORD"],
        }
    }
else:
    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.sqlite3",
            "NAME": BASE_DIR / "db.sqlite3",
        }
    }


# Password validation
# https://docs.djangoproject.com/en/5.2/ref/settings/#auth-password-validators

AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',

## Fragmento 7: backend/pedidos/plataforma/cors.py
from django.http import HttpResponse


class CorsMiddleware:
    """Deja que Flutter en Chrome (otro puerto) lea las respuestas de la API."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # El navegador pregunta con OPTIONS antes de un POST con JSON.
        if request.method == "OPTIONS":
            respuesta = HttpResponse(status=204)
        else:
            respuesta = self.get_response(request)
        respuesta["Access-Control-Allow-Origin"] = "*"
        respuesta["Access-Control-Allow-Methods"] = "GET, POST, OPTIONS"
        respuesta["Access-Control-Allow-Headers"] = "Content-Type"
        return respuesta
````

</details>

## Paso 2 — Probar que la herramienta responde

Le hice una pregunta corta a `llama3.2:3b` y contestó bien. Después bajé `qwen2.5-coder:7b`.

## Paso 3 — Prompt de auditoría (con evidencias)

Este fue el prompt. Después de él pegué el paquete completo del paso 1:

````
Actúa como revisor de un proyecto universitario (Django + Flutter, con nginx como gateway, un recomendador con modelo .joblib y un asistente).
Abajo viene el contexto del equipo y fragmentos de código. Solo puedes usar lo que está en este texto.
Tareas:
1) Da hallazgos numerados. Cada uno con: severidad (alta/media/baja), archivo o módulo sospechoso, evidencia tomada del texto, riesgo si fuera cierto y cómo verificarlo un humano en 5 minutos.
2) Separa "hecho con evidencia" de "hipótesis sin evidencia".
3) Señala contradicciones con los requisitos: el modelo vive en el servidor, el mismo front para web y móvil, sin secretos en el repo.
4) No reescribas el proyecto. No inventes archivos, rutas ni servicios que no se mencionaron; si no lo ves en el texto, no lo afirmes.
5) Termina con las 5 comprobaciones manuales más útiles antes de la demo.
Responde en español y de forma breve.
````

Respuesta completa de la IA A (`gemma3:4b`):

<details>
<summary>Respuesta completa de la IA A (gemma3:4b)</summary>

````
## Hallazgos de Auditoría del Proyecto Veterinario

Aquí tienes los hallazgos de la auditoría, clasificados por severidad y con evidencia del texto:

**1. Alta Severidad: Falta de Cumplimiento de Requisitos de Arquitectura**

*   **Hecho con Evidencia:** "El modelo vive en el servidor" y "el mismo front para web y móvil, sin secretos en el repo".
*   **Riesgo:**  Alta.  La separación de la lógica de recomendación en un servicio independiente (recomendar) es fundamental para la escalabilidad, el mantenimiento y la seguridad.  El uso del mismo código para web y móvil es un riesgo de incompatibilidad y limita las funcionalidades específicas de cada plataforma. La ausencia de secretos en el repositorio es un riesgo de seguridad.
*   **Verificación Humana (5 minutos):**  Confirmar que el servicio `recomendar` realmente aloja el modelo `.joblib` y que no hay código específico para web o móvil en el repositorio.  Revisar la configuración de Docker para asegurar que el modelo se expone correctamente.  Verificar que no hay claves API, contraseñas o información sensible en el repositorio.

**2. Media Severidad: Problemas de Conectividad y Manejo de Errores en el Gateway**

*   **Hecho con Evidencia:** "server { listen 80; resolver 127.0.0.11 valid=5s ipv6=off; proxy_connect_timeout 2s; error_page 502 504 /_servicio_caido".
*   **Riesgo:** Media. El uso de `resolver` con `valid=5s` puede causar retrasos significativos si el servicio `recomendar` no está disponible. El manejo de errores con `/_servicio_caido` es aceptable, pero la respuesta JSON es demasiado genérica.
*   **Verificación Humana (5 minutos):**  Simular la caída del servicio `recomendar` y verificar que el gateway responde correctamente con la página `/_servicio_caido`.  Analizar el tiempo de respuesta del resolver.  Considerar una estrategia de tiempo de espera más agresiva para el `proxy_connect_timeout`.  Mejorar la respuesta JSON de error para proporcionar más información sobre el fallo.

**3. Media Severidad:  Problemas de CORS y Seguridad en el Gateway**

*   **Hecho con Evidencia:** "add_header Access-Control-Allow-Origin \"*\" always; add_header Access-Control-Allow-Methods \"GET, POST, OPTIONS\" always; add_header Access-Control-Allow-Headers \"Content-Type\" always;".
*   **Riesgo:** Media.  Permitir `Access-Control-Allow-Origin "*" ` es un riesgo de seguridad, ya que permite cualquier origen.  Aunque se incluyen los encabezados CORS, la configuración es demasiado permisiva.
*   **Verificación Humana (5 minutos):**  Revisar la configuración de CORS y restringir el origen a `http://localhost:8080` (o el dominio correcto de la aplicación Flutter).  Asegurarse de que los encabezados CORS se establecen correctamente en todas las rutas.

**4. Baja Severidad:  Dependencia de `settings.py` para Hosts Permitidos**

*   **Hecho con Evidencia:** "ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS", "localhost,127.0.0.1").split(",")".
*   **Riesgo:** Baja.  La dependencia de `settings.py` para la lista de hosts permitidos es un punto de fallo.
*   **Verificación Humana (5 minutos):**  Asegurarse de que la lista de hosts permitidos se define de forma consistente en `settings.py` y en el gateway.  Considerar el uso de un mecanismo más flexible para la gestión de hosts permitidos.

**5. Baja Severidad:  Falta de Validación de Datos en `crear_pedido`**

*   **Hecho con Evidencia:**  `crear_pedido` en `backend/pedidos/views.py`.
*   **Riesgo:** Baja. No se valida que los datos recibidos en el cuerpo de la petición sean del tipo esperado.
*   **Verificación Humana (5 minutos):**  Añadir validación de datos en la función `crear_pedido` para asegurar que los datos recibidos son del tipo esperado.

## Comprobaciones Manuales Finales (Antes de la Demo)

1.  **Verificar la Disponibilidad del Servicio `recomendar`:**  Asegurarse de que el contenedor `recomendar` está en ejecución y respondiendo a las peticiones.
2.  **Probar el Flujo de Recomendación:**  Realizar una petición a `/mascotas/recomendar/` y verificar que el servicio `recomendar` devuelve una respuesta JSON con la recomendación.
3.  **Verificar la Integración con el Servicio `pedidos`:**  Realizar una petición a `/mascotas/pedidos/` y verificar que el servicio `pedidos` recibe la petición, llama al servicio `recomendar` y devuelve una respuesta JSON con el folio del pedido.
4.  **Probar el Asistente:**  Realizar una petición a `/mascotas/asistente/` y verificar que el servicio `pedidos` devuelve una respuesta JSON con la respuesta del asistente.
5.  **Verificar la Respuesta de Error:**  Simular la caída del servicio `recomendar` y verificar que el gateway devuelve la respuesta JSON correcta en la página `/_servicio_caido`.

---

**Nota:** Esta auditoría se basa únicamente en la información proporcionada en el contexto.  Se recomienda una auditoría más exhaustiva para identificar posibles problemas adicionales.
````

</details>

Inventó una ruta que no existe (`backend/pedidos/views.py`; la real es `backend/pedidos/mascotas/views.py`) y la anoté como falso positivo.

## Paso 4 — Segunda opinión

Le pasé el mismo prompt y paquete a B y C. Sus respuestas completas:


<details>
<summary>Respuesta completa de la IA B (gemma4:e4b)</summary>

````
## 🔎 Revisión de Proyecto Universitario

### 1. Hallazgos de Auditoría

#### Hechos con Evidencia (Arquitectura Confirmada)
*   **Modelo en Servidor:** El modelo `.joblib` reside en `backend/recomendar/` y es consumido por `backend/pedidos/` a través de una llamada HTTP.
*   **Front Único:** El código Flutter (`app/lib/`) es el mismo para web y móvil, y el gateway (`nginx.conf`) sirve la versión web (`../app/build/web`).
*   **Gateway Centralizado:** Todo el tráfico pasa por `gateway/nginx.conf` (puerto 8080), que dirige las peticiones a `pedidos` o `recomendar`.

#### Hipótesis sin Evidencia (Riesgos Potenciales)
*   *No hay hipótesis de alto riesgo sin evidencia. Los hallazgos se centran en configuraciones de seguridad y robustez.*

---

#### 🚨 Hallazgos Numerados

**1. Configuración de Seguridad (Alto)**
*   **Archivo/Módulo Sospechoso:** `backend/pedidos/plataforma/settings.py` y `backend/docker-compose.yml`.
*   **Evidencia:** `SECRET_KEY = os.environ.get("DJANGO_SECRET_KEY", "solo-para-desarrollo")` (settings.py) y `DJANGO_SECRET_KEY: cambiar-en-produccion` (docker-compose.yml).
*   **Riesgo:** El uso de valores por defecto o *hardcoded* para claves secretas (como `solo-para-desarrollo`) en el repositorio expone el proyecto a riesgos de seguridad si se utiliza la configuración por defecto en un entorno real.
*   **Verificación (5 min):** Revisar el `.gitignore` y confirmar que las variables de entorno (como `DJANGO_SECRET_KEY`) no están siendo cargadas desde archivos `.env` que puedan estar en el repositorio.

**2. Exposición de CORS (Media)**
*   **Archivo/Módulo Sospechoso:** `backend/gateway/nginx.conf`.
*   **Evidencia:** `add_header Access-Control-Allow-Origin "*" always;`
*   **Riesgo:** Permitir `*` (cualquier origen) en producción puede exponer la API a ataques CSRF o permitir que sitios maliciosos consuman los endpoints sin restricciones de dominio.
*   **Verificación (5 min):** Intentar acceder a un endpoint de la API desde un navegador en un dominio diferente al esperado (ej. `http://evil.com/mascotas/pedidos/`).

**3. Fallo de Servicio y Plan B (Media)**
*   **Archivo/Módulo Sospechoso:** `backend/pedidos/mascotas/views.py` (`_plan_b`).
*   **Evidencia:** Si `recomendar` falla, se activa `_plan_b` y se asigna `NIVEL_POR_DEFECTO = "urgente"`.
*   **Riesgo:** El sistema está diseñado para asumir el peor escenario (urgente) si el servicio de recomendación falla. Si bien esto es una regla de seguridad, debe ser documentado como un *fallo de diseño* (fail-safe) y no como un comportamiento normal, ya que puede generar falsos positivos de urgencia.
*   **Verificación (5 min):** Detener el contenedor `recomendar` y ejecutar una petición POST a `/mascotas/pedidos/` para confirmar que el Plan B se activa y el estado de respuesta es 503.

**4. Timeout Agresivo del Gateway (Baja)**
*   **Archivo/Módulo Sospechoso:** `backend/gateway/nginx.conf`.
*   **Evidencia:** `proxy_connect_timeout 2s;`
*   **Riesgo:** Un timeout de 2 segundos es muy corto para una llamada de microservicio. Si el servicio `recomendar` está bajo carga o experimenta latencia, el gateway lo cortará prematuramente, activando innecesariamente el Plan B en `pedidos`.
*   **Verificación (5 min):** Simular una latencia alta en el servicio `recomendar` (ej. usando `sleep` en Flask) y medir el tiempo de respuesta del gateway.

**5. Configuración de Debug por Defecto (Media)**
*   **Archivo/Módulo Sospechoso:** `backend/pedidos/plataforma/settings.py`.
*   **Evidencia:** `DEBUG = os.environ.get("DJANGO_DEBUG", "1") == "1"`.
*   **Riesgo:** Si la variable de entorno `DJANGO_DEBUG` no se establece, Django por defecto operará en modo `DEBUG=True`, exponiendo información sensible del servidor (traza de errores, variables de entorno) en producción.
*   **Verificación (5 min):** Ejecutar el proyecto sin definir la variable `DJANGO_DEBUG` y revisar la respuesta de un error 500 para ver si se expone información interna.

---

### 2. Contradicciones con Requisitos Declarados

| Requisito Declarado | Estado en el Código | Contradicción / Observación |
| :--- | :--- | :--- |
| El modelo vive en el servidor. | **Cumplido.** El modelo `.joblib` está en `recomendar` y es llamado por `pedidos`. | Ninguna. |
| El mismo front para web y móvil. | **Cumplido.** Se usa `app/lib/` (Flutter) y el gateway sirve la versión web. | Ninguna. |
| Sin secretos en el repo. | **Violado.** `settings.py` y `docker-compose.yml` contienen valores por defecto de secretos (`solo-para-desarrollo`, `cambiar-en-produccion`). | Los secretos están presentes como *fallbacks* en el código fuente. |

---

### 3. 🚀 5 Comprobaciones Manuales Antes de la Demo

1.  **Prueba de Resiliencia (Plan B):** Detener el servicio `recomendar` y ejecutar `POST /mascotas/pedidos/`. Verificar que el gateway devuelva 503 y que la respuesta contenga `plan_b: true` y el nivel "urgente".
2.  **Prueba de Flujo Completo:** Ejecutar `POST /mascotas/pedidos/` con un caso de prueba conocido. Verificar que el servicio `pedidos` haya llamado a `recomendar` y que el resultado final sea 201, incluyendo `razones_reglas` y `modelo_nombre`.
3.  **Prueba de Gateway (Rutas):** Intentar acceder a `/mascotas/pedidos/` y luego a `/mascotas/recomendar/` desde el cliente. Verificar que el gateway dirija correctamente a los servicios internos sin exponer IPs.
4.  **Prueba de CORS:** Acceder a un endpoint de la API desde una herramienta externa (Postman o cURL) y verificar que los encabezados `Access-Control-Allow-*` estén presentes en la respuesta.
5.  **Prueba de Fallo de Validación:** Enviar una petición a `POST /mascotas/pedidos/` con un caso incompleto (que falle en `leer_caso`). Verificar que el servicio `pedidos` devuelva un 400 y el mensaje de error esperado.
````

</details>

<details>
<summary>Respuesta completa de la IA C (qwen2.5-coder:7b)</summary>

````
### Hallazgos

1. **Severidad: Alta**
   - **Archivo o módulo sospechoso:** `backend/pedidos/mascotas/views.py`
   - **Evidencia:** `DJANGO_SECRET_KEY: cambiar-en-produccion`
   - **Riesgo:** Exposición de la clave secreta en el código fuente.
   - **Verificación:** Buscar la clave secreta en el código fuente y asegurarse de que no sea "cambiar-en-produccion".

2. **Severidad: Media**
   - **Archivo o módulo sospechoso:** `backend/pedidos/mascotas/views.py`
   - **Evidencia:** `settings.RECOMENDAR_URL = os.environ.get("RECOMENDAR_URL", "http://localhost:8002")`
   - **Riesgo:** El modelo de urgencia se ejecuta en el servidor, lo que podría ser inseguro si el servidor no es confiable.
   - **Verificación:** Verificar que el servidor que ejecuta el modelo de urgencia es seguro y confiable.

3. **Severidad: Baja**
   - **Archivo o módulo sospechoso:** `app/lib/servicios/api_mascotas.dart`
   - **Evidencia:** `const urlBase = String.fromEnvironment('URL_BASE', defaultValue: 'http://127.0.0.1:8080');`
   - **Riesgo:** La aplicación Flutter usa una URL base que puede ser cambiada en tiempo de ejecución, lo que podría ser una vulnerabilidad si no se maneja adecuadamente.
   - **Verificación:** Verificar que la URL base se maneja adecuadamente y no se modifica sin autorización.

### Hecho con evidencia

1. **Modelo vive en el servidor:** El modelo de urgencia se ejecuta en el servidor `recomendar`, lo que es correcto.
2. **Mismo front para web y móvil:** La aplicación Flutter web y móvil comparte el mismo código, lo que es correcto.
3. **Sin secretos en el repo:** La clave secreta está marcada como "cambiar-en-produccion", lo que es correcto.

### Hipótesis sin evidencia

1. **Reglas de seguridad:** No se proporciona información sobre las reglas de seguridad específicas implementadas en el proyecto.

### Contradicciones con los requisitos

1. **Modelo vive en el servidor:** El modelo de urgencia se ejecuta en el servidor `recomendar`, lo que es correcto.
2. **Mismo front para web y móvil:** La aplicación Flutter web y móvil comparten el mismo código, lo que es correcto.
3. **Sin secretos en el repo:** La clave secreta está marcada como "cambiar-en-produccion", lo que es correcto.

### Comprobaciones manuales

1. **Verificar la clave secreta:** Buscar la clave secreta en el código fuente y asegurarse de que no sea "cambiar-en-produccion".
2. **Verificar la seguridad del servidor de recomendar:** Verificar que el servidor que ejecuta el modelo de urgencia es seguro y confiable.
3. **Verificar la URL base en la aplicación Flutter:** Verificar que la URL base se maneja adecuadamente y no se modifica sin autorización.
4. **Verificar el funcionamiento del modelo de urgencia:** Probar el funcionamiento del modelo de urgencia y asegurarse de que no haya errores.
5. **Verificar el funcionamiento del asistente:** Probar el funcionamiento del asistente y asegurarse de que no haya errores.
````

</details>

| Hallazgo | IA A | IA B | IA C | ¿Lo vimos en el código? | Qué haremos |
| --- | --- | --- | --- | --- | --- |
| `CORS: *` en `nginx.conf` | sí | sí | no | parcial | se queda en desarrollo |
| `DEBUG` en `1` por defecto | no | sí | no | parcial | se queda |
| Valores tipo secreto en el repo | no | sí | sí | parcial | cambiarlos antes de producción |
| Falta validación en `crear_pedido` | sí | no | no | no | nada, ya valida |
| `proxy_connect_timeout 2s` muy corto | no | sí | no | no | nada |
| Plan B con `urgente` es fallo de diseño | no | sí | no | no | nada, es decisión documentada |
| `URL_BASE` se cambia en ejecución | no | no | sí | no | nada |
| `RECOMENDAR_URL` en `views.py` y modelo inseguro | no | no | sí | no | nada |

Ninguno lo dijeron las tres. Los que coinciden en dos (CORS y secretos) salieron parciales.

## Paso 5 — Cómo analizar el resultado

### Herramientas y modelos

`gemma3:4b`, `gemma4:e4b` y `qwen2.5-coder:7b`, los tres locales con Ollama.

### Qué le pasé y qué omití

Le pasé el resumen del paso 1 con los siete fragmentos. Omití el historial de Git, datasets, `.joblib`, la interfaz de Flutter, las pruebas y la documentación.

### Tabla de hallazgos verificados

| Hallazgo | Quién lo dijo | Lo que vi en el código | Qué hice |
| --- | --- | --- | --- |
| `CORS: *` | A, B | Cierto, pero la API no usa cookies ni sesiones | Se queda en desarrollo |
| `DEBUG` en `1` por defecto | B | Cierto en `settings.py`, pero el Compose pone `DJANGO_DEBUG: "0"` | Se queda |
| Valores tipo secreto | B, C | Son de desarrollo, no hay secretos reales | Se queda |
| **Al apk le falta `INTERNET` y el tráfico `http`** | ninguna | `AndroidManifest.xml` no lo tenía. Lo encontré yo | **Corregido** |
| **Contrato de los 8 campos duplicado** en `pedidos` y `recomendar` | ninguna | Si cambia un campo hay que cambiarlo en los dos. Lo encontré yo | Queda anotado, se unifica si el proyecto crece |

### Falsos positivos (con referencia al código)

1. **A dijo «falta validación en `crear_pedido`» (en `backend/pedidos/views.py`).** En `mascotas/views.py`, `crear_pedido` empieza con `leer_caso`, que ya valida todo, y las pruebas `CasoTests` lo comprueban. Esa ruta ni existe.
2. **B dijo que `proxy_connect_timeout 2s` es muy corto.** En `nginx.conf` solo es la espera para conectarse. La espera a `recomendar` está en `settings.py` (`RECOMENDAR_ESPERA_SEGUNDOS = 3`).
3. **B dijo que el plan B con `urgente` es un fallo de diseño.** En `views.py` es una decisión puesta a propósito (`NIVEL_POR_DEFECTO`) y la pantalla dice «sin evaluar».
4. **C dijo que `URL_BASE` se cambia en ejecución.** En `api_mascotas.dart` es `String.fromEnvironment`, se fija al compilar.
5. **C dijo que `RECOMENDAR_URL` está en `views.py` y que el modelo en el servidor es inseguro.** Está en `settings.py`, y el modelo en el servidor es justo lo que pide el enunciado.

### Cambios hechos después

No hice commits. Arreglé `AndroidManifest.xml` (permiso `INTERNET`, `http` permitido, nombre `Mascotas`).

### Qué quedó pendiente y por qué

- Probar la app en un celular Android: todavía no lo hago, así que no sé si el arreglo del permiso de internet funciona.
- Dejar los datos del formulario en un solo lugar: ahorita están repetidos en dos partes, pero para el Proyecto 1 no estorba.
- Ajustar la seguridad (contraseñas y permisos de prueba): solo hace falta cuando se publique de verdad.

Conclusión: ningún hallazgo de las IAs era un bug real, los dos de verdad los encontré yo abriendo los archivos. Sirven de segunda opinión, pero hay que comprobar cada cosa en el código. El modelo de código no salió mejor que los generales.

## Paso 6 — Cierre antes de la demo

`flutter analyze` salió limpio y las 6 pruebas de Flutter pasan. Probé recomendación, folio y chat con todo levantado y funcionaron. El build web está hecho y el apk no lo probé.

## Errores frecuentes

| Qué pasa | ¿Me pasó? | Qué hice |
| --- | --- | --- |
| Inventa cosas que no existen | Sí, la IA A se inventó un archivo | Lo anoté como error de la IA |
| Todo le parece grave | Sí, la IA B marcó como grave algo que no lo era | Yo decidí qué tan grave era |
| La compu se queda sin memoria | Sí, el programa falló una vez | Le bajé la cantidad de texto que lee a la vez |
| Pegar contraseñas en el chat | No | Antes revisé que no hubiera |
| Que la IA escriba el informe de patrones | No | No se lo pedí, ese lo reviso y lo firmo yo |
