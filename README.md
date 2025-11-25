# PracticaKubernetes
---
En esta práctica vamos a realizar un ejemplo de un sistema de comnicación Cliente-Servidor empleando un Broker, haciendo uso de imagenes de docker en maquinas EC2 de amazon. Para lograr esto haremos uso de kubernetes, para poder trabajar con todo nuestro sistema.

## Configuración de puertos
---
Por un lado tenemos un servidor y un broker, y cada uno de ellos tiene un deployment.yml y un service.yml.

### Broker
````
  Puerto --> 32002
````
### Service
````
Puerto ---> 32001
````
