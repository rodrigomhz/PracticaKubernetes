# Practica_Kubernetes
---
## Objetivo de la Práctica:

El objetivo de esta práctica es desplegar una aplicación distribuida utilizando Kubernetes y Docker en un clúster de instancias EC2:

En Kubernetes, un clúster está formado por:

  - Master node (Control-Plane): Es la máquina que se encarga de coordinar todo el trabajo. Supervisa las tareas, decide dónde poner los servicios y coordina los demás nodos.

  - Worker nodes (Nodos esclavos): Son las máquinas donde realmente ejecutan las aplicaciones (en nuestro caso, los servicios como brokerFileManager y serverFileManager). En estos nodos se crean los pods que ejecutan tus servicios, vamos a desglosarlos un poco más.:

    - **brokerFileManager**: El broker que gestiona la comunicación entre el servidor y el cliente.

    - **serverFileManager**: El servidor que proporciona acceso a los archivos.

A continuación, desglosaré lo que hemos hecho hasta ahora, explicando cada paso y su propósito:

---

## 1. Preparación del Entorno:

### Despliegue en EC2:

. Tienes 3 instancias EC2:

  - Broker: Esta instancia ejecuta el servicio del brokerFileManager.

  - Control-Plane: Es la máquina que gestiona el control de Kubernetes.

  - Server: Esta instancia ejecuta el servicio del serverFileManager.

Estas instancias están en ejecución, y cada una tiene una función clave dentro de la infraestructura.

. Kubernetes:

  - Kubernetes se usará para gestionar los contenedores Docker que ejecutarán los servicios de brokerFileManager y serverFileManager.

## 2. Imágenes Docker:

### Dockerfile (serverFileManager):

Para crear los contenedores que ejecutarán los servicios, necesitamos un Dockerfile para cada servicio. Este archivo contiene las instrucciones sobre cómo crear la imagen Docker para un servicio en particular. A continuación, explicamos cómo funciona el Dockerfile que hemos configurado para cada uno:

DockerFileS (serverFileManager):
````
# Usar una imagen base oficial de Ubuntu
FROM ubuntu:20.04

# Actualizar los repositorios e instalar dependencias
RUN apt-get update
RUN apt-get install -y software-properties-common 
RUN apt-get install -y curl

# Exponer el puerto 32001
EXPOSE 32001

COPY serverFileManager /
RUN chmod +x /serverFileManager
RUN mkdir FileManagerDir
COPY resolv.conf /
CMD cp resolv.conf /etc/resolv.conf && /serverFileManager 172.31.31.163 32002 $(curl -s https://api.ipify.org) 32001
````
La IP debe ser la del Broker.

### DockerfileB (brokerFileManager):
````
# Usar una imagen base oficial de Ubuntu
FROM ubuntu:20.04

# Actualizar los repositorios e instalar dependencias
RUN apt-get update
RUN apt-get install -y software-properties-common 
RUN apt-get install -y curl

# Exponer el puerto 32002
EXPOSE 32002

# Queremos haccer el dockerFile del Broker

# Copiar el ejecutable al contenedor
COPY brokerFileManager /brokerFileManager

# Dar permisos de ejecuacion
RUN chmod +x /brokerFileManager

# Ejecutar brokerFileManager cuando inicie el contenedor
CMD /brokerFileManager
````

serverFileManager necesita conectarse al brokerFileManager para obtener información sobre cómo conectarse a otros servidores. Esto se hace utilizando el puerto 32002, que es el puerto en el que el broker está esperando las conexiones.

## Deployments

En Kubernetes, Deployment es un objeto que gestiona la creación y actualización de pods de manera automática. Un Deployment asegura que siempre haya el número adecuado de pods ejecutándose, incluso en el caso de fallos o actualizaciones. En este caso, tenemos dos Deployments: uno para brokerFileManager y otro para serverFileManager.

### brokerDeployment:
````
apiVersion: apps/v1
kind: Deployment
metadata:
 name: brokerfilemanager-deployment
 namespace: default
spec:
 replicas: 1
 selector:
  matchLabels:
   app: brokerfilemanager
 template:
  metadata:
   labels:
    app: brokerfilemanager
  spec:
   containers:
   - name: brokerfilemanager-deployment
     image: docker.io/bitboss629/brokerfilemanager:v1  
````

### serverDeployment:
````
apiVersion: apps/v1
kind: Deployment
metadata:
 name: serverfilemanager-deployment
 namespace: default
spec:
 replicas: 1
 selector:
  matchLabels:
   app: serverfilemanager
 template:
  metadata:
   labels:
    app: serverfilemanager
  spec:
   containers:
   - name: serverFilemanager-Deployment
     image: docker.io/bitboss629/serverfilemanager:v1    
````
---
````
image: docker.io/bitboss629/brokerfilemanager:v1
````

ESO significa:

  - “Kubernetes buscará la imagen bitboss629/brokerfilemanager:v1 en Docker Hub”

  - Si no existe → FALLA

  - Si existe → Kubernetes la descarga y levanta los pods
  - 
## Servicios

En Kubernetes, los Services se utilizan para exponer los puertos de los pods de manera consistente y permitir la comunicación entre ellos. Existen diferentes tipos de servicios, pero en este caso estamos usando NodePort, lo que significa que los servicios estarán accesibles desde fuera del clúster a través de un puerto específico.

### brokerService.
````
apiVersion: v1
kind: Service
metadata:
 name: brokerservice
 namespace: default
spec:
 type: NodePort
 selector:
  app: brokerfilemanager
 ports:
 # Enlazamos el puerto 32002 interno al externo 32002
 #Virtual
  - port: 32002 # Puerto al que se accederá dentro del contenedor
    targetPort: 32002 # El puerto dentro del pod al que el servicio debe redirigir
    # Fisico
    nodePort: 32002 # Puerto expuesto externamente en el nodo
 externalTrafficPolicy: Cluster
````

### serverService
````
apiVersion: v1
kind: Service
metadata:
 name: serverservice
 namespace: default
spec:
 type: NodePort
 selector:
  app: serverfilemanager
 ports:
  - port: 32001 # Puerto dentro del clúster para el servidor
    targetPort: 32001 # El puerto dentro del pod al que el servicio debe redirigir
    nodePort: 32001 # Puerto expuesto externamente en el nodo para el servicio
 externalTrafficPolicy: Cluster
````
Resumen Conceptos:

  1. name: Es el nombre del servicio. Como si le pones un nombre a la puerta del servicio, para poder llamarla.
  
  2. app: Es una etiqueta para identificar a qué grupo de casitas (pods) pertenece este servicio. Es como si dijeras: "Estas casitas son del tipo broker".
  
  3. port: Es el puerto dentro del clúster que otros servicios usan para acceder a tu servicio. Es la puerta que los servicios dentro de Kubernetes usarán para conectarse.
  
  4. targetPort: Es el puerto dentro del contenedor donde el servicio realmente está esperando las conexiones. Es como la puerta dentro de la casita.
  
  5. nodePort: Es el puerto en los nodos del clúster que se expone a fuera del clúster. Es como si pusieras una puerta de entrada en la muralla del barrio para que la gente desde fuera pueda entrar.

# Ejecucuión
---
Antes de desplegar nuestros servicios en Kubernetes, es necesario construir las imágenes Docker de ambos componentes: brokerFileManager y serverFileManager.
Estas imágenes son las que luego utilizarán los Deployments.

## 1. Construcción y subida de la imagen del Broker

Primero, desde la carpeta que contiene el Dockerfile del broker, construimos su imagen:

````
docker build -t bitboss629/brokerfilemanager:v1 .
````

Una vez generada, la subimos a Docker Hub para que Kubernetes pueda descargarla:
````
docker push bitboss629/brokerfilemanager:v1
````

## 2. Construcción y subida de la imagen del Server

A continuación, repetimos el proceso para el servidor, situándonos en la carpeta donde está su Dockerfile:
````
docker build -t bitboss629/serverfilemanager:v1 .
````

Subimos también esta imagen a Docker Hub:
````
docker push bitboss629/serverfilemanager:v1
````

## 3. Aplicación de los Deployments en Kubernetes

Con ambas imágenes ya disponibles en Docker Hub, podemos desplegar los servicios en el clúster usando los archivos YAML correspondientes:
````
kubectl apply -f brokerDeployment.yaml
kubectl apply -f serverDeployment.yaml
````

Esto creará los pods y asignará las imágenes construidas previamente a los Deployments, de forma que cada servicio pueda ejecutarse dentro del clúster Kubernetes

## 4. Aplicacion de los Service en Kubernetes

Ahora que los pods existen, ya puedes exponerlos mediante los servicios:
````
kubectl apply -f brokerService.yaml
kubectl apply -f serverService.yaml
````
---

## ¿Por qué en este orden?

### 📌 Deployments → crean los pods

  - Sin pods, los services no tienen a quién conectar.

### 📌 Services → exponen los pods

  - Una vez los pods existen, puedes "publicarlos" dentro y fuera del clúster.
