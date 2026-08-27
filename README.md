# Instituto Tecnológico de Morelia
**Examen Diagnóstico - Fundamentos de Ingeniería de Software**  
**Materia:** Tópicos Selectos de Tecnologías Web y Móvil  
**Profesor:** Jesús Eduardo Alcaraz Chávez  

**Nombre del alumno:** Cesar Enrique Diaz Maldonado  
**Fecha:** 27/08/2026  

---

### Instrucciones
Responde de manera clara y concisa a cada una de las siguientes preguntas abiertas. El propósito de esta evaluación es medir tus conocimientos previos en Ingeniería de Software.

---

### 1. Metodologías
**¿Cuál es la diferencia principal entre una metodología de desarrollo tradicional (como Cascada) y una metodología ágil (como Scrum) frente a los cambios en los requisitos?**

Que la metodología tradicional se utiliza en proyectos donde ya hay requisitos fijos y no van a cambiar y las metodologías agiles es lo contrario es donde el proyecto va a estar en constante cambio y por consiguiente los requisitos siempre o casi siempre van a estar cambiando.

---

### 2. Requerimientos
**Explica la diferencia entre requerimientos funcionales y no funcionales, dando un ejemplo de cada uno aplicable a una plataforma web.**

Los requisitos funcionales nos dices lo que debe de hacer el sistema, todas las funcionalidades en si, las necesidades del cliente y los requisitos no funcionales son el como debe de funcionar el sistema, las necesidades técnicas del sistema, ya sea en cuestiones de rendimiento, calidad, etc.

---

### 3. Arquitectura
**Describe el modelo Cliente-Servidor y explica brevemente cómo se comunican el frontend y el backend en una aplicación web moderna.**

En este modelo el cliente es el que hace la petición al servidor por medio del frontend (cliente) y el backend (servidor) recibe esa petición y a procesa dependiendo del funcionamiento del sistema el cual también se va a conectar a la base de datos, esta comunicación la hacen por medio de una API Rest y con los métodos http.

---

### 4. Bases de Datos
**¿En qué escenarios recomendarías utilizar una base de datos relacional (SQL) frente a una no relacional (NoSQL) para el almacenamiento de datos en una aplicación?**

Una relacional se usa cuando tu sistema tiene las entidades y relaciones claras y no van a cambiar demasiado, además necesitas asegurar que una operación se complete al 100% siempre y una no relacional se usa cuando tienes volúmenes de datos demasiado grandes y tu sistema cambia constantemente.

---

### 5. APIs
**¿Qué es una API REST y qué papel fundamental juega en la integración entre una aplicación móvil y los servidores (backend)?**

Una API Rest es un método con el cual comunicamos el frontend con el backend, en si es una URL con la cual llamamos a nuestro método y lo manipulamos dependiendo la transacción que queramos hacer, usando GET, POST, PUT, PATCH y DELETE que son los métodos HTTP.

---

### 6. Control de Versiones
**Explica la importancia de utilizar Git en un equipo de desarrollo de software y describe brevemente qué es un "merge conflict" (conflicto de fusión).**

Es muy importante porque todos los integrantes del equipo pueden trabajar en el mismo proyecto al mismo tiempo y saber exactamente que es lo que hace cada uno, además de que te da la posibilidad de tener disponibles versiones antiguas del proyecto por si necesitas regresar si fallo algo, también tienes a la vista lo que hacen los demás para que sepas que hizo cada quien y saber que no debes tocar.

El merge conflict es cuando dos o mas personas trabajan o en la misma parte del código e hicieron cambios distintos que se afectan entre si porque cambiaron cosas parecidas o que alguien si lo tiene en su versión de código y alguien mas no.

---

### 7. Pruebas
**¿Qué son las pruebas unitarias (unit testing) y por qué son cruciales para asegurar la calidad del software antes de su paso a producción?**

Es cada test solitario que haces de cada funcion o método del sistema, es como una prueba aislada que nos dice si una pequeña parte del sistema funciona correctamente para que cuando se integre a las demás todo siga funcionando.

---

### 8. POO
**Define los conceptos de encapsulamiento y polimorfismo de la Programación Orientada a Objetos, y menciona cómo ayudan a crear un código más mantenible.**

El polimorfismo es usar una misma clase pero para diferentes cosas y nos ayuda a que nuestro código no tenga que tener clases muy parecidas que solo cambian en un aspecto muy sencillo o básico mientras podríamos usar una misma para varios requisitos.

---

### 9. Patrones de Diseño
**¿Qué es el patrón de arquitectura Modelo-Vista-Controlador (MVC) y cómo ayuda a organizar el código en el desarrollo de software?**

---

### 10. Seguridad
**Explica la diferencia técnica entre "autenticación" y "autorización" en el contexto de seguridad de una aplicación.**

La autenticación es un proceso mediante el cual el sistema verifica la identidad del usuario, dispositivo o API, etc  
Y la autorización es a que funciones le permites entrar y que funciones puede realizar un usuario en el sistema
