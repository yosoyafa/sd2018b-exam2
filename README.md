### Examen 2
**Universidad ICESI**  
**Curso:** Sistemas Distribuidos  
**Docente:** Carlos Andrés Afanador Cabal   
**Tema:** Construcción de artefactos para entrega continua   
**Correo:** carlosafanador97 at gmail.com

### Objetivos
* Realizar de forma autómatica la generación de artefactos para entrega continua.
* Emplear librerías de lenguajes de programación para la realización de tareas específicas.
* Diagnosticar y ejecutar de forma autónoma las acciones necesarias para corregir fallos en la infraestructura.

### Desarrollo

#### Registry  
Para tener un control centralizado propio de las imagenes de Docker a usar en nuestros clientes de Docker, se crea un Docker Registry privado. En primera instancia, en la carpeta *docker_data/certs* se crean certificados para darle seguridad al sistema, de la sigueinte forma:
```
$ mkdir docker_data
$ mkdir docker_data/certs
$ cd docker_data/certs

$ sudo openssl req -newkey rsa:4096 -nodes -sha256 -keyout `pwd`/docker_data/certs/domain.key -x509 -days 365 -out `pwd`/docker_data/certs/domain.crt
```
Este proceso arrojará la siguiente salida:

**llaves.png**

A continuación
