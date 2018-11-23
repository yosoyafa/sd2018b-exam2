### Examen 2 
**Universidad ICESI**  
**Curso:** Sistemas Distribuidos  
**Docente:** Carlos Andrés Afanador Cabal :colombia:   
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

![][1]

A continuación se procede a implementar el artefacto por medio de la herramienta Compose de Docker, incluyendo los servicios de **integración contínua (CI)** para realizar el proceso de decisión al integrar las ramas en el repositorio, **Registry** para el control privado de las imagenes de Docker para el proceso de integración, y **ngrok** para poder acceder a los servicios desplegados desde internet. Al archivo de Compose para el artefacto, *docker-compose.yml*, es el siguiente:
```
version: '3'
services:
    registry:
        image: registry:2
        restart: always
        container_name: Registry_Server
        volumes:
            - './docker_data/certs:/certs'
        environment:
            - 'REGISTRY_HTTP_ADDR=0.0.0.0:443'
            - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt
            - REGISTRY_HTTP_TLS_KEY=/certs/domain.key
        ports:
            - '443:443'
    ci_server:
        build: ci_server
        container_name: ci_server
        volumes:
          - //var/run/docker.sock:/var/run/docker.sock
        environment:
          - 'CI_SERVER_HTTP_ADDR=0.0.0.0:8080'
        ports:
          - '8080:8080'
    ngrok:
        image: wernight/ngrok
        ports:
          - '0.0.0.0:4040:4040'
        links:
          - ci_server
        environment:
NGROK_PORT: ci_server:8080
```
Para el despliegue del servidor de integración se utiliza el siguiente Dockerfile:
```
FROM python:3.6

COPY . /handler_endpoint

WORKDIR /handler_endpoint

RUN pip3.6 install --upgrade pip
RUN pip3.6 install connexion[swagger-ui]
RUN pip3.6 install --trusted-host pypi.python.org -r requirements.txt

RUN ["chmod", "+x", "/handler_endpoint/deploy.sh"]

CMD ./deploy.sh
```

Mediante Swagger, se implementa el API del servidor de integración:
```
swagger: '2.0'
info:
  title: User API
  version: "0.1.0"
paths:
  /ciserver/updates:
    post:
      x-swagger-router-controller: gm_analytics
      operationId: handlers.repository_changed
      summary: The repository has changed.
      responses:
        200:
          description: Successful response.
          schema:
            type: object
            properties:
              command_return:
                type: string
                description: The information is procesing
  /:
    post:
      x-swagger-router-controller: gm_analytics
      operationId: handlers.hello
      summary: The repository has changed.
      responses:
        200:
          description: Successful response.
          schema:
            type: object
            properties:
              command_return:
                type: string
description: The information is procesing
```
El proceso gobernado por el API se ve expresado a través de un método de python3.6 que recibe y analiza los *pull requests* hechos al repositorio, y decide si hace la integración o no. 
```python
import os
import logging
import requests
import json
import docker
from flask import Flask, request, json

def hello():
    result = {'command_return': 'work'}
    return result

def repository_changed():
    result_swagger   = ""
    post_json_data   = request.get_data()
    string_json      = str(post_json_data, 'utf-8')
    json_pullrequest = json.loads(string_json)
    branch_merged = json_pullrequest["pull_request"]["merged"]
    if branch_merged:
        pullrequest_sha  = json_pullrequest["pull_request"]["head"]["sha"]
        json_image_url     = "https://raw.githubusercontent.com/yosoyafa/sd2018b-exam2/" + pullrequest_sha + "/images.json"
        response_image_url = requests.get(json_image_url)
        image_data    =  json.loads(response_image_url.content)
        for service in image_data:
            service_name = service["service_name"]
            image_type = service["type"]
            image_version = service["version"]
            if image_type == 'Docker':
                dockerfile_image_url = "https://raw.githubusercontent.com/yosoyafa/sd2018b-exam2/" + pullrequest_sha + "/" + service_name + "/Dockerfile"
                file_response = requests.get(dockerfile_image_url)
                file = open("Dockerfile","w")
                file.write(str(file_response.content, 'utf-8'))
                file.close()
                image_tag  = "Registry_Server:443/" + service_name + ":" + image_version
                client = docker.DockerClient(base_url='unix://var/run/docker.sock')
                client.images.build(path="./", tag=image_tag)
                client.images.push(image_tag)
                client.images.remove(image=image_tag, force=True)
                result_swagger = image_tag + " - Image built - " + result_swagger
            else:
                out = {'command return' : 'JSON file have an incorrect format'}
        out = {'cammand return' : result_swagger}
    else:
        out= {'command_return': 'Pull request was not merged'}
return out
```
Para ejecutar t





[1]: images/llaves.png
[2]: images/build1.png
[3]: images/build2.png
[4]: images/ngrokstatus.png
[5]: images/webhook.png
[6]: images/200cli.png
[7]: images/200ngrok.png
