# Aplicar contenedores a aplicación SPRING MVC y Java

En este tutorial vamos a mostrarte como crear un contenedor para una aplicación Spring MVC:

0.  Primero seleccionamos una imagen a partir de la cual deseamos partir, en este caso utilizaremos la imagen base minimalista de ubuntu:
```c
phusion/baseimage:0.9.17
```

1.  Luego procedemos a actualizar los repositorios de paquetes:
```c
RUN echo "deb http://archive.ubuntu.com/ubuntu trusty main universe" > /etc/apt/sources.list

RUN apt-get -y update
```

2.  Luego instalamos paquetes que utilizaremos más adelante:
```c
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -q python-software-properties software-properties-common


RUN apt-get -y update
```

3.  Luego instalamos java 8:
```c
ENV JAVA_VER 8
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

RUN echo 'deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main' >> /etc/apt/sources.list && \
    echo 'deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main' >> /etc/apt/sources.list && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C2518248EEA14886 && \
    apt-get update && \
    echo oracle-java${JAVA_VER}-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections && \
    apt-get install -y --force-yes --no-install-recommends oracle-java${JAVA_VER}-installer oracle-java${JAVA_VER}-set-default && \
    apt-get clean && \
    rm -rf /var/cache/oracle-jdk${JAVA_VER}-installer

RUN update-java-alternatives -s java-8-oracle

RUN echo "export JAVA_HOME=/usr/lib/jvm/java-8-oracle" >> ~/.bashrc
```

4.  Hacemos un clean up de APT
```c
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```

5.  Luego utilizamos el inicio del sistema de la imagen base:
```c
CMD ["/sbin/my_init"]
```

Y tenemos nuestro Dockerfile listo:
```c
# Dockerfile

FROM  phusion/baseimage:0.9.17

MAINTAINER  Author Name <author@email.com>

RUN echo "deb http://archive.ubuntu.com/ubuntu trusty main universe" > /etc/apt/sources.list

RUN apt-get -y update

RUN DEBIAN_FRONTEND=noninteractive apt-get install -y -q python-software-properties software-properties-common

ENV JAVA_VER 8
ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

RUN echo 'deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main' >> /etc/apt/sources.list && \
    echo 'deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main' >> /etc/apt/sources.list && \
    apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C2518248EEA14886 && \
    apt-get update && \
    echo oracle-java${JAVA_VER}-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections && \
    apt-get install -y --force-yes --no-install-recommends oracle-java${JAVA_VER}-installer oracle-java${JAVA_VER}-set-default && \
    apt-get clean && \
    rm -rf /var/cache/oracle-jdk${JAVA_VER}-installer

RUN update-java-alternatives -s java-8-oracle

RUN echo "export JAVA_HOME=/usr/lib/jvm/java-8-oracle" >> ~/.bashrc

RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

CMD ["/sbin/my_init"]
```

6.  Ahora procedemos a construir una imagen con nuestro Dockerfile
```c
$ docker build -f Dockerfile -t ejemplo/java:8 .
```



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

3.  Luego procedemos a instalar tomcat:
```c
RUN wget -q https://archive.apache.org/dist/tomcat/tomcat-${TOMCAT_MAJOR_VERSION}/v${TOMCAT_MINOR_VERSION}/bin/apache-tomcat-${TOMCAT_MINOR_VERSION}.tar.gz && \
	wget -qO- https://archive.apache.org/dist/tomcat/tomcat-${TOMCAT_MAJOR_VERSION}/v${TOMCAT_MINOR_VERSION}/bin/apache-tomcat-${TOMCAT_MINOR_VERSION}.tar.gz.md5 | md5sum -c - && \
	tar zxf apache-tomcat-*.tar.gz && \
 	rm apache-tomcat-*.tar.gz && \
 	mv apache-tomcat* tomcat

ADD create_tomcat_admin_user.sh /create_tomcat_admin_user.sh
RUN mkdir /etc/service/tomcat
ADD run.sh /etc/service/tomcat/run
RUN chmod +x /*.sh
RUN chmod +x /etc/service/tomcat/run

EXPOSE 8080
```

4.  Utilizamos el inicio de sistema de baseimage-docker
```c
CMD ["/sbin/my_init"]
```

5.  Creamos el usuario administrador de tomcat
```c
!/bin/bash

if [ -f /.tomcat_admin_created ]; then
    echo "Tomcat 'admin' user already created"
    exit 0
fi
```

6.  Generamos una password para el usuario
```c
PASS=${TOMCAT_PASS:-$(pwgen -s 12 1)}
_word=$( [ ${TOMCAT_PASS} ] && echo "preset" || echo "random" )

echo "=> Creating an admin user with a ${_word} password in Tomcat"
sed -i -r 's/<\/tomcat-users>//' ${CATALINA_HOME}/conf/tomcat-users.xml
echo '<role rolename="manager-gui"/>' >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo '<role rolename="manager-script"/>' >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo '<role rolename="manager-jmx"/>' >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo '<role rolename="admin-gui"/>' >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo '<role rolename="admin-script"/>' >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo "<user username=\"admin\" password=\"${PASS}\" roles=\"manager-gui,manager-script,manager-jmx,admin-gui, admin-script\"/>" >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo '</tomcat-users>' >> ${CATALINA_HOME}/conf/tomcat-users.xml
echo "=> Done!"
touch /.tomcat_admin_created

echo "========================================================================"
echo "You can now configure to this Tomcat server using:"
echo ""
echo "    admin:${PASS}"
echo ""
echo "========================================================================"
```

7.  Agregamos un archivo más para crear los usuarios y luego reiniciar el servidor tomcat:

```c
!/bin/bash

if [ ! -f /.tomcat_admin_created ]; then
    /create_tomcat_admin_user.sh
fi

exec ${CATALINA_HOME}/bin/catalina.sh run
```

8.  Construimos y probamos la imagen
```c
$ docker build -f Dockerfile -t demo/spring:maven-3.3-jdk-8 .
```

9.  Cargamos una aplicación de muestra spring mvc
```c
$ git clone  https://github.com/atifsaddique211f/spring-maven-sample.git

$ cd spring-maven-sample
```

10. Ejecutamos el siguiente comando para empaquetar y construir el proyecto:
```c
$ docker run -it --rm -v "$PWD":/app -w /app demo/spring:maven-3.3-jdk-8 mvn clean install
```

El comando anterior creará un archivo war en el directorio target/springwebapp.war

11. Copiamos este war al directorio webapps de Tomcat:
```c
$ docker run -it -d --name spring -p 8080:8080 -v "$PWD":/app demo/spring:maven-3.3-jdk-8 bash -c "cp /app/target/springwebapp.war /tomcat/webapps/ & /tomcat/bin/catalina.sh run"
```

12. Finalmente accedemos nuestra app con la siguiente URL:
```c
http://localhost:8080/springwebapp/car/add
```