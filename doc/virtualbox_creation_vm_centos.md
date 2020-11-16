# Création de VMs CentOS dans VirtualBox

## Création de la VM

* Obtenir une iso CentOS : http://isoredirect.centos.org/centos/7/isos/x86_64/
* Choisir un mirroir permettant d'obtenir une iso CentOS minimale
* Créer une nouvelle VM avec un disque virtuel _vdi_ de taille fixe
* Configurer l'interface réseau NAT pour accéder au port 22 du serveur

## Installer CentOS

* Démarrer la VM en _mode normal_
* Choisir l'iso CentOS minimale en disque de démarrage
* Installer CentOS dans la VM
* Activer l'interface _enp0s3_
    * Passer le paramètre _ONBOOT_ à _yes_ dans le fichier _/etc/sysconfig/network-scripts/ifcfg-enp0s3_
    * Activer l'interface

```bash
ifup enp0s3
```

* Mettre à jour le système

```bash
yum upgrade -y
```

## Installer Docker

* Suivre les instructions d'installation depuis le site Docker : https://docs.docker.com/engine/install/centos/
* Installer Docker Compose : https://docs.docker.com/compose/install/


## Installer un conteneur Jenkins

* Suivre les instructions d'installation depuis le site Jenkins : https://www.jenkins.io/doc/book/installing/docker/

> Pour simplifier l'administration de Jenkins, il est commode d'utiliser un docker-compose.

Jenkins ne fournit pas de fichier _docker-compose.yml_ officiel.

Voici ma proposition

```yaml
version: "3.8"
services:
  jenkins-docker:
    image: docker:dind
    privileged: true
    networks:
      jenkins:
        aliases:
          - docker
    environment:
      - DOCKER_TLS_CERTDIR=/certs
    volumes:
      - "jenkins-docker-certs:/certs/client"
      - "jenkins-data:/var/jenkins_home"
    ports:
      - "2376:2376"
  jenkins-blueocean:
    build:
      context: .
      dockerfile: Dockerfile
    image: myjenkins-blueocean:1.1
    depends_on:
      - jenkins-docker
    networks:
      - jenkins
    environment:
      - DOCKER_HOST=tcp://docker:2376
      - DOCKER_CERT_PATH=/certs/client
      - DOCKER_TLS_VERIFY=1
    ports:
      - 8080:8080
      - 50000:50000
    volumes:
      - "jenkins-data:/var/jenkins_home"
      - "jenkins-docker-certs:/certs/client:ro"
networks:
  jenkins:
volumes:
  jenkins-data:
  jenkins-docker-certs:
```
 
On peut ensuite lancer les images avec la commande :
```shell script
docker-compose up -d
```

Et l'arrêter avec :
```shell script
docker-compose down
```

> user : admin
>
> pwd : admin

## Installer un conteneur Jmeter

Apache ne fournit pas de conteneur officiel.
Donc nous allons le construire !

D'abord, créer un _Dockerfile_ pour générer une image.

```Dockerfile
FROM alpine
ARG JAVA_VERSION=openjdk11
ARG JMETER_VERSION=apache-jmeter-5.3
ARG JMETER_URL=https://downloads.apache.org//jmeter/binaries/
ENV JMETER_HOME=/opt/$JMETER_VERSION
ENV JMETER_BIN=$JMETER_HOME/bin/
ENV JMETER_WORKDIR=/var/opt/$JMETER_VERSION
RUN JMETER_FILE=$JMETER_VERSION.tgz
RUN apk update \
    && apk add $JAVA_VERSION
RUN wget -P /tmp $JMETER_URL/$JMETER_FILE \
    && tar -xzvf /tmp/$JMETER_FILE -C /opt \
    && rm /tmp/$JMETER_FILE
RUN freeMem=`awk '/MemFree/ { print int($2/1024) }' /proc/meminfo` \
    && s=$(($freeMem/10*8)) \
    && x=$(($freeMem/10*8)) \
    && n=$(($freeMem/10*2))
ENV JVM_ARGS="-Xmn${n}m -Xms${s}m -Xmx${x}m"
ENV PATH "$PATH:$JMETER_BIN"
RUN mkdir -p $JMETER_WORKDIR/scripts $JMETER_WORKDIR/log $JMETER_WORKDIR/results $JMETER_WORKDIR/data
WORKDIR $JMETER_WORKDIR
CMD /bin/sh
```

> Une source utile : https://www.blazemeter.com/blog/make-use-of-docker-with-jmeter-learn-how

> On notera l'importance d'allouer un espace mémoire adapté à la JVM pour ne pas lever d'erreur du type "There is insufficient memory for the Java Runtime Environment to continue".

```shell script
freeMem=`awk '/MemFree/ { print int($2/1024) }' /proc/meminfo`
s=$(($freeMem/10*8))
x=$(($freeMem/10*8))
n=$(($freeMem/10*2))
export JVM_ARGS="-Xmn${n}m -Xms${s}m -Xmx${x}m"
```

Puis, construire l'image

```shell script
docker build -t buzensb/jmeter:1.0 .
```

Créer un fichier _docker-compose.yml_
```yaml
version: "3.8"
services:
    jmeter:
        image: buzensb/jmeter:1.0
        volumes:
            - "./scripts:/var/opt/apache-jmeter-5.3/scripts"
            - "./log:/var/opt/apache-jmeter-5.3/log"
            - "./results:/var/opt/apache-jmeter-5.3/results"
            - "./data:/var/opt/apache-jmeter-5.3/data"
```