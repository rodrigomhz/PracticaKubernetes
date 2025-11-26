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
   nodeSelector:
    rol: broker
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
   nodeSelector:
    rol: server
   containers:
   - name: serverfilemanager-deployment
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

Antes de esto debemos de asignar a nuestros pods unos roles para relacionarlos con los deployments:
````
  kubectl label node k8smaster0.psdi.org rol=broker
  kubectl label node k8sslave1.psdi.org rol=server
````
![nodo](https://github.com/rodrigomhz/PracticaKubernetes/blob/main/Images/nodes.png)

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

Antes de comenzar debemos loguearnos:
````
docker login
````
Te pedirá:

  - Username → tu usuario de Docker Hub

  - Password → tu contraseña de Docker Hub

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

Podemos comprobar que todo ha ido bien con el comando:
````
docker images
````

![nodo](https://github.com/rodrigomhz/PracticaKubernetes/blob/main/Images/dockerImages.png)

## 3. Aplicación de los Deployments en Kubernetes

Con ambas imágenes ya disponibles en Docker Hub, podemos desplegar los servicios en el clúster usando los archivos YAML correspondientes:
````
kubectl apply -f brokerDeployment.yml
kubectl apply -f serverDeployment.yml
````

Esto creará los pods y asignará las imágenes construidas previamente a los Deployments, de forma que cada servicio pueda ejecutarse dentro del clúster Kubernetes

## 4. Aplicacion de los Service en Kubernetes

Ahora que los pods existen, ya puedes exponerlos mediante los servicios:
````
kubectl apply -f brokerService.yml
kubectl apply -f serverService.yml
````
---

## ¿Por qué en este orden?

### 📌 Deployments → crean los pods

  - Sin pods, los services no tienen a quién conectar.

### 📌 Services → exponen los pods

  - Una vez los pods existen, puedes "publicarlos" dentro y fuera del clúster.


## CONFIGURACIÓN AVANZADA 1

### 🟦 Objetivo

Tener varios pods del serverFileManager ejecutándose en el mismo nodo y compartiendo una misma carpeta para que todos los clientes vean los mismos archivos sin importar qué pod atienda la petición.

¿Por qué funciona hostPath aquí?

Porque hostPath monta una carpeta del nodo físico dentro de cada pod.
Como los pods están en el mismo nodo, todos montan la misma carpeta:

````
EC2 nodo esclavo
│
├── /mnt/data   ← carpeta física del nodo
│     ├─ archivo1.txt
│     ├─ archivo2.png
│
├── pod1 → monta /mnt/data en /data
├── pod2 → monta /mnt/data en /data
└── pod3 → monta /mnt/data en /data
````

  ✔ TODOS ven lo mismo

  ✔ TODOS escriben/leen lo mismo

  ✔ NUNCA se borra si un pod cae

  ✔ Cumple exactamente la Configuración Avanzada 1

Esa carpeta /mnt/data hay que crearla en el nodo esclavo, NO en el control-plane

---

### 1. Crear la carpeta compartida en el nodo esclavo

Nos conectamos a la máquina de server:
````
ssh -i labsuser.pem ubuntu@k8sslave2.psdi.org
````

En el nodo donde se ejecutarán los pods del server:
````
sudo mkdir -p /mnt/server-data
sudo chmod 777 /mnt/server-data
````

Comprobamos los nodos:
````
kubectl get nodes
````

Verificamos la etiqueta:
````
kubectl get nodes --show-labels
````

### 2. Deployment del serverFileManager con hostPath + nodeSelector
````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: serverfilemanager-deployment
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: serverfilemanager
  template:
    metadata:
      labels:
        app: serverfilemanager
    spec:
      nodeSelector:
        rol: server
      containers:
      - name: serverfilemanager
        image: docker.io/bitboss629/serverfilemanager:v1
        volumeMounts:
        - name: server-storage
          mountPath: /FileManagerDir
      volumes:
      - name: server-storage
        hostPath:
          path: /home/ubuntu/compartido
          type: DirectoryOrCreate
````
Resumen Conceptos:

  - hostPath.path → carpeta REAL del nodo

  - mountPath → carpeta dentro del contenedor

  - replicas: 3 → 3 pods usando la MISMA carpeta

  - nodeSelector → obliga a que los pods estén en el nodo marcado

---

### 3. Hacer imagen de docker
````
docker build -t bitboss629/serverfilemanager:v1 .
docker push bitboss629/serverfilemanager:v1
````

1. Aplicar el deployment
````
kubectl apply -f serverDeployment.yml
````

2. Comprobar que funciona
````
kubectl get pods -o wide
````
Los 3 pods deben estar en el MISMO nodo.

Entrar en un pod:
````
kubectl exec -it <nombre-pod> -- bash
ls /data
````

Entrar en  otro pod:
````
kubectl exec -it <otro-pod> -- bash
ls /data
````

✔ Deben aparecer los mismos archivos
✔ Lo que suba un cliente al pod1 aparece en pod2 y pod3
✔ Todo se guarda en /mnt/server-data del nodo esclavo

## CONCLUSIÓN
Esto completa la Configuración Avanzada 1 tal y como exige la práctica

🚀 Parte 3: Configuración Avanzada con NFS
Esta sección implementa almacenamiento compartido NFS para alcanzar la máxima calificación (10). Permite tener 3 réplicas del servidor compartiendo los mismos archivos.

¿Por qué NFS?
✅ Alta disponibilidad: 3 réplicas del servidor corriendo simultáneamente
✅ Persistencia: Los archivos se mantienen aunque un pod se reinicie
✅ Compartición: Todas las réplicas ven los mismos archivos en tiempo real
✅ Escalabilidad: Fácil añadir más réplicas sin perder datos
3.1 Instalar y Configurar Servidor NFS
En k8smaster0:

# Actualizar repositorios
sudo apt-get update

# Instalar servidor NFS
sudo apt-get install -y nfs-kernel-server

# Crear directorio compartido
sudo mkdir -p /mnt/nfs-filemanager

# Configurar permisos
sudo chown nobody:nogroup /mnt/nfs-filemanager
sudo chmod 777 /mnt/nfs-filemanager
Configurar exportación NFS:

# Añadir al archivo de exportaciones
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Aplicar configuración
sudo exportfs -ra

# Verificar que el export está activo
sudo exportfs -v
Salida esperada:

/mnt/nfs-filemanager
        <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
Verificar servicio:

sudo systemctl status nfs-kernel-server
⚠️ Problema Común 1: Ruta sin barra inicial

Si ves el error exportfs: Failed to stat mnt/nfs-filemanager: No such file or directory:

# Eliminar línea incorrecta
sudo sed -i '/^mnt\/nfs-filemanager/d' /etc/exports

# Añadir correctamente (con / inicial)
echo "/mnt/nfs-filemanager *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Aplicar
sudo exportfs -ra
3.2 Instalar Cliente NFS en Workers
En k8sslave2 (usando kubectl debug desde k8smaster0):

# Entrar al nodo con chroot
kubectl debug node/k8sslave2.psdi.org -it --image=ubuntu -- chroot /host bash

# Dentro del nodo, instalar cliente NFS
apt-get update
apt-get install -y nfs-common

# Salir
exit
Verificar instalación:

kubectl debug node/k8sslave2.psdi.org -it --image=ubuntu -- chroot /host bash -c "dpkg -l | grep nfs-common"
Salida esperada:

ii  nfs-common  1:2.6.1-1ubuntu1.2  amd64  NFS support files common to client and server
💡 Nota: Los pods de debug temporales pueden eliminarse después:

kubectl delete pod -l app=node-debugger
3.3 Configurar Security Groups de AWS para NFS
Antes de crear los recursos de Kubernetes, debes abrir los puertos NFS en AWS:

Ve a AWS Console → EC2 → Security Groups
Selecciona el security group de k8smaster0
Editar reglas de entrada y añadir:
Tipo	Protocolo	Puerto	Origen	Descripción
TCP personalizado	TCP	2049	172.31.0.0/16	NFS
TCP personalizado	TCP	111	172.31.0.0/16	RPC (portmapper)
⚠️ Problema Común 2: Connection timed out al montar NFS

Si los pods muestran mount.nfs: Connection timed out, es porque el Security Group bloquea los puertos NFS. Asegúrate de añadir las reglas anteriores.

3.4 Crear PersistentVolume NFS
Archivo: pv-nfs.yml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: server-pv-nfs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany  # Permite que múltiples pods lo usen simultáneamente
  nfs:
    server: 172.31.64.84  # IP privada de k8smaster0
    path: /mnt/nfs-filemanager
  storageClassName: nfs
Crear el archivo y aplicarlo:

cd ~/Practica2/SERVIDOR_NFS

# Crear el archivo pv-nfs.yml con el contenido anterior
nano pv-nfs.yml

# Aplicar
kubectl apply -f pv-nfs.yml

# Verificar
kubectl get pv
Salida esperada:

NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
server-pv-nfs   5Gi        RWX            Retain           Available
3.5 Crear PersistentVolumeClaim NFS
Archivo: pvc-nfs.yml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: server-pvc-nfs
spec:
  accessModes:
    - ReadWriteMany  # Debe coincidir con el PV
  resources:
    requests:
      storage: 5Gi  # Solicita 5Gi
  storageClassName: nfs  # Debe coincidir con el PV
Aplicar:

# Crear el archivo
nano pvc-nfs.yml

# Aplicar
kubectl apply -f pvc-nfs.yml

# Verificar que se vinculó al PV
kubectl get pvc
kubectl get pv
Salida esperada:

NAME             STATUS   VOLUME          CAPACITY   ACCESS MODES
server-pvc-nfs   Bound    server-pv-nfs   5Gi        RWX
Estado Bound significa que el PVC encontró el PV y está listo para usar.

3.6 Actualizar Deployment del Servidor con NFS
Ahora modifica el DeploymentServer.yml para usar el volumen NFS y escalar a 3 réplicas:

apiVersion: apps/v1
kind: Deployment
metadata:
 name: server-deployment
 namespace: default
spec:
 replicas: 3  # ← CAMBIO: Escalar de 1 a 3 réplicas
 selector:
  matchLabels:
   app: server-deploy
 template:
  metadata:
   labels:
    app: server-deploy
  spec:
   nodeSelector:
    node-role: server
   containers:
   - name: server-deployment
     image: docker.io/d1n0s/kubernetes-practica2server:v2
     ports:
     - containerPort: 32001
     volumeMounts:  # ← NUEVO: Montar el volumen NFS
     - name: filemanager-storage-nfs
       mountPath: /FileManagerDir  # Donde la app guarda archivos
   volumes:  # ← NUEVO: Definir el volumen desde el PVC
   - name: filemanager-storage-nfs
     persistentVolumeClaim:
       claimName: server-pvc-nfs  # Referencia al PVC creado
Aplicar los cambios:

# Eliminar el deployment anterior
kubectl delete deployment server-deployment

# Aplicar el nuevo con NFS
kubectl apply -f DeploymentServer.yml

# Observar cómo se crean las 3 réplicas
kubectl get pods -w
⚠️ Problema Común 3: ImagePullBackOff

Si los pods muestran ImagePullBackOff, verifica la versión de la imagen:

# Ver el error
kubectl describe pod server-deployment-xxx

# Si dice que no encuentra v3, verifica el deployment
kubectl get deployment server-deployment -o yaml | grep image

# Debe ser v2 (que existe en Docker Hub)
# Si está mal, edita DeploymentServer.yml y vuelve a aplicar
⚠️ Problema Común 4: MountVolume.SetUp failed - Connection timed out

Este es el problema más común al configurar NFS. Los pods quedan en ContainerCreating:

# Ver el error
kubectl describe pod server-deployment-xxx
Causa: Security Group de AWS bloqueando puertos NFS.

Solución: Añadir reglas en Security Group (ver sección 3.3).

Salida esperada cuando todo funciona:

NAME                                 READY   STATUS    NODE
broker-deployment-6fd556654c-jzdsx   1/1     Running   k8sslave1.psdi.org
server-deployment-6bc5f558c5-7vpf9   1/1     Running   k8sslave2.psdi.org
server-deployment-6bc5f558c5-cvltl   1/1     Running   k8sslave2.psdi.org
server-deployment-6bc5f558c5-q9j**   1/1     Running   k8sslave2.psdi.org
✅ Parte 4: Pruebas y Verificación
Esta sección documenta las pruebas realizadas para verificar que el sistema funciona correctamente con NFS.

4.1 Verificar Estado del Cluster
# Ver todos los pods
kubectl get pods -o wide

# Ver servicios
kubectl get svc

# Ver volúmenes
kubectl get pv,pvc
4.2 Prueba de Persistencia de Archivos
Paso 1: Crear un archivo de prueba

cd ~/Practica2
echo "Esto es una prueba" > Prueba.txt
cat Prueba.txt  # Verificar contenido
Paso 2: Subir archivo al sistema

# Conectar al broker
./clientFileManager 172.31.31.30 32002

# Dentro del cliente, subir el archivo
upload Prueba.txt

# Verificar que se subió
lls
Salida esperada:

Enter command:
upload Prueba.txt
Coping file Prueba.txt in to the FileManager path
Reading file: Prueba.txt 19 bytes

Enter command:
lls
Listing files fileManager path
FileManagerDir/Prueba.txt
Paso 3: Salir y reconectar (puede conectar a otra réplica)

# Salir (Ctrl+C si "exit" no funciona)
# Volver a conectar
./clientFileManager 172.31.31.30 32002

# Listar archivos
lls
Resultado esperado: El archivo Prueba.txt debe seguir ahí, confirmando la persistencia.

4.3 Verificar Archivos en el Servidor NFS
En k8smaster0:

# Ver archivos en el directorio NFS
ls -la /mnt/nfs-filemanager/

# Ver contenido del archivo
cat /mnt/nfs-filemanager/Prueba.txt
Salida esperada:

total 12
drwxrwxrwx 2 nobody nogroup 4096 Nov 26 17:45 .
drwxr-xr-x 3 root   root    4096 Nov 26 17:10 ..
-rw-r--r-- 1 nobody nogroup   19 Nov 26 17:45 Prueba.txt
4.4 Verificar Logs de las Réplicas
# Obtener nombres de los pods
kubectl get pods | grep server-deployment

# Ver logs de cada réplica (reemplaza con tus nombres reales)
kubectl logs server-deployment-6bc5f558c5-7vpf9
kubectl logs server-deployment-6bc5f558c5-cvltl
kubectl logs server-deployment-6bc5f558c5-q9j**
Deberías ver que todas las réplicas están registradas en el broker y listas para recibir conexiones.
