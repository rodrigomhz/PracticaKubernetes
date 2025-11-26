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
