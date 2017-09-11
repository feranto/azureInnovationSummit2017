# Aplicar contenedores a aplicación SPRING MVC y Java

En este tutorial vamos a mostrarte como crear un contenedor para una aplicación Spring MVC:

1.  Lo primero que debemos hacer es crear una nueva carpeta y un nuevo archivo de tipo "dockerfile"
2.  Dentro de este archivo agregamos el siguiente contenido:
```c
# Dockerfile
FROM demo/maven:3.3-jdk-8
MAINTAINER Author <autor@email.com>
RUN apt-get update && \
    apt-get install -yq --no-install-recommends wget pwgen ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
ENV TOMCAT_MAJOR_VERSION 8
ENV TOMCAT_MINOR_VERSION 8.0.11
ENV CATALINA_HOME /tomcat
```

Acá especificamos lo siguiente:
-   Version de jdk
-   Version de maven
-   Variables de entorno
