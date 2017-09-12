# Aplicar contenedores a aplicación SPRING MVC y Java

En este tutorial vamos a mostrarte como crear un contenedor para una aplicación Spring MVC:

## Creación de Imagen Base
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

Estos nos creará la imagen ejemplo/java:8 en nuestro sistema para poder seguir construyendo con ella.


## Agregamos Maven a la imagen Base

1.  Vamos a utilizar la imagen que creamos recientemente ejemplo/java:8 como base. Y creamos una nueva carpeta y un nuevo dockerfile y agregamos lo siguiente:

```c
# Dockerfile
FROM ejemplo/java:8

ENV MAVEN_VERSION 3.3.9

RUN mkdir -p /usr/share/maven \
  && curl -fsSL http://apache.osuosl.org/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    | tar -xzC /usr/share/maven --strip-components=1 \
  && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

ENV MAVEN_HOME /usr/share/maven

VOLUME /root/.m2

CMD ["mvn"] 
```

2.  Luego construigmos la imagen:

```c
$ docker build -f Dockerfile -t ejemplo/maven:3.3-jdk-8 .
```




## Agregamos Tomcat a la imagen Maven

1.  Lo primero que debemos hacer es crear una nueva carpeta y un nuevo archivo de tipo "dockerfile"
2.  Dentro de este archivo agregamos el siguiente contenido:
```c
# Dockerfile
FROM ejemplo/maven:3.3-jdk-8
MAINTAINER Author <autor@email.com>
RUN apt-get update && \
    apt-get install -yq --no-install-recommends wget pwgen ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
ENV TOMCAT_MAJOR_VERSION 8
ENV TOMCAT_MINOR_VERSION 8.0.11
ENV CATALINA_HOME /tomcat
```


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

5.  Creamos un archivo llamado create_tomcat_admin_user.sh en el mismo directorio con el siguiente contenido:
```c
!/bin/bash

if [ -f /.tomcat_admin_created ]; then
    echo "Tomcat 'admin' user already created"
    exit 0
fi
```

6.  Añadimos las siguientes lineas en donde generamos una password para el usuario
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

7.  Agregamos un archivo más llamado run.sh para crear los usuarios y luego reiniciar el servidor tomcat:

```c
!/bin/bash

if [ ! -f /.tomcat_admin_created ]; then
    /create_tomcat_admin_user.sh
fi

exec ${CATALINA_HOME}/bin/catalina.sh run
```

8.  Construimos y probamos la imagen
```c
$ docker build -f Dockerfile -t ejemplo/spring:maven-3.3-jdk-8 .
```

9.  Cargamos una aplicación de muestra spring mvc
```c
$ git clone  https://github.com/atifsaddique211f/spring-maven-sample.git

$ cd spring-maven-sample
```

10. Ejecutamos el siguiente comando para empaquetar y construir el proyecto:
```c
$ docker run -it --rm -v "$PWD":/app -w /app ejemplo/spring:maven-3.3-jdk-8 mvn clean install
```

El comando anterior creará un archivo war en el directorio target/springwebapp.war

11. Copiamos este war al directorio webapps de Tomcat:
```c
$ docker run -it -d --name spring -p 8080:8080 -v "$PWD":/app ejemplo/spring:maven-3.3-jdk-8 bash -c "cp /app/target/springwebapp.war /tomcat/webapps/ & /tomcat/bin/catalina.sh run"
```
Aca el parametro -d, hará que corra en modo "detached" y también mapea el puerto 8080 del contenedor al puerto 8080 de la maquina.

12. Finalmente accedemos nuestra app con la siguiente URL:
```c
http://localhost:8080/springwebapp/car/add
```

Para chequear el proceso podemos hacer el siguiente comando:
```c
$ docker ps
```

Y para poder cancelarlo podemos usar:
```c
docker rm -vf spring
```




## Guardamos la imagen en un registro

Finalmente lo que haremos será subir esta imagen a un registro, en este caso dockerhub para ello hacemos los siguientes pasos:

1.  Hacemos login con nuestra cuenta de docker hub:
```c
docker login --username=yourhubusername 
```

2.  Luego procedemos a obtener nuestras imagenes locales:
```c
docker images
```

3.  Luego agregamos etiquetas a nuestra imagen
```c
docker tag bb38976d03cf yourhubusername/verse_gapminder:firsttry
```

4.  Luego hacemos push a nuestra imagen
```c
docker push yourhubusername/verse_gapminder
```




ejemplo/srpingapp:1-maven-3.3-jdk-8

docker run -it -d --name spring -p 8080:8080 -v "$PWD":/app ejemplo/srpingapp:1-maven-3.3-jdk-8 bash -c "cp /app/target/springwebapp.war /tomcat/webapps/ & /tomcat/bin/catalina.sh run"

$ docker run -it -d --name spring -p 8080:8080 -v "$PWD":/app ejemplo/spring:maven-3.3-jdk-8 bash -c "cp /app/target/springwebapp.war /tomcat/webapps/ & /tomcat/bin/catalina.sh run"