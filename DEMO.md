# Affichage de la version de docker (client et serveur)
```shell
docker version
```

# Docker, un produit client / server

Le client docker effectue des appels sur l'API REST exposée par le démon sur un socket unix
```shell
curl --unix-socket /var/run/docker.sock http::/containers/json | jq .
```

Le démon docker est lancé en tant que service
```shell
service docker status
```
# ======= Mon premier container ========
```shell
docker run hello-world
```
# Il s'est passé quoi ?
```shell
docker images
docker ps -a
docker pull hello-world
```

# De quoi est fait hello-world ?
https://github.com/docker-library/hello-world/blob/master/amd64/hello-world/Dockerfile
```Dockerfile
FROM scratch
COPY hello /
CMD ["/hello"]
```
![Layers](/images/layers.png)

# D'ou vient l'image hello-world ?
# ======= DockerHub : la registry de Docker Inc. ========
https://hub.docker.com/_/hello-world

- Repository public d'images docker, et registry par defaut lors de l'installation de docker.
- Propose à tous détenteur de comptes de pouvoir uploader ses propres images, de façon public ou privée.
- Système de notation communautaire.
- Branchement avec GitHub et BitBucket.


La recherche d'image est disponibles en ligne de commande :
```shell
docker search nginx
```

On récupère des images existantes simplement avec la commande `docker pull nginx`

On push des images personnelles sur son espace simplement avec la commande `docker push mon-espace/mon-image`

On se loggue sur une registry docker avec la commande `docker login`

L'utilisation de registry autre que DockerHub est évidement possible (registry interne à l'entreprise ou cloud)

# ======= Concept de CONTAINER ========

Une image est un ensemble de layers en lecture seule, plus des metadonnées
Un container est un ensemble de thread et ressources complètement isolés au moyen des namespace et cgroup linux, qui s'exécutent sur le FS d'une image surplombée par une layer en modification.


## Présentation des système de fichier en couche (UnionFS, AuFS, Btrfs, zfs, overlay, overlay2, devicemapper)

## Pull d'une image
```shell
docker pull alpine:3.8
```

## Lancement d'un container basé sur l'image alpine:3.8
```shell
docker run alpine ls /
docker ps -a
docker run -it alpine
```
- le FS du container est isolé de celui de l'hôte
-

## Modifications dans un container et commit dans une image
```shell
docker run alpine touch /tmp/mon-fichier.txt
docker run alpine ls /tmp
docker ps -a
docker diff <id container>
```

![LayersModifs](/images/layers-modifs.png)

```shell
docker commit <id container> alpine-modifiee
docker images
docker history alpine-modifiee
```
![LayersCommit](/images/layers-commit.png)

Chaque layer contient le différentiel par rapport aux layers sous jacentes.
Si l'on modifie le fichier `/etc/passwd`, la layer ne contiendra pas le fichier entier, mais seulment le patch.

## Sauvegarde d'une image
```shell
docker save -o alpine-modifiee.tar alpine-modifiee
mkdir alpine-modifiee
tar xvf alpine-modifiee.tar -C alpine-modifiee
ll alpine-modifiee
```

## Création d'une image à partir d'un Dockerfile
```shell
cd build
```

```Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
```

Le fichier index.html
```html
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Docker for life!</h1>
<p><em>index.html</em></p>
</body>
</html>
```

```shell
docker build -f Dockerfile.step1 -t nginx-demo .
docker run -d -p 88:80 nginx-demo
docker history nginx-demo
```
## Anatomie d'un Dockerfile

`FROM`: indique de quelle image l'on part

`COPY`: copie un fichier depuis le contexte de build vers l'image finale

`RUN`: execute une commande lors du build de l'image

`ENTRYPOINT`: défini la commande executée lors du démarrage d'un container basé sur cette image

`CMD`: permet de donner des valeurs par défaut à l'ENTRYPOINT

`ENV`: défini une variable d'environnement, disponible pour le build mais aussi lors de l'exécution du container

`VOLUME`: permet d'indiquer l'utilisation d'un volume docker à l'exécution

`EXPOSE`: permet d'indiquer explicitement que l'on veut exposer un port

`WORKDIR`: positionne un répertoie comme répertoire courant pour la suite des instructions du Dockerfile


## Exemple d'un Dockerfile plus étoffé : prometheus

https://github.com/prometheus/prometheus/blob/master/Dockerfile

```Dockerfile
FROM        quay.io/prometheus/busybox:latest
LABEL maintainer="The Prometheus Authors <prometheus-developers@googlegroups.com>"

COPY prometheus                             /bin/prometheus
COPY promtool                               /bin/promtool
COPY documentation/examples/prometheus.yml  /etc/prometheus/prometheus.yml
COPY console_libraries/                     /usr/share/prometheus/console_libraries/
COPY consoles/                              /usr/share/prometheus/consoles/

RUN ln -s /usr/share/prometheus/console_libraries /usr/share/prometheus/consoles/ /etc/prometheus/ && \
    mkdir -p /prometheus && \
    chown -R nobody:nogroup etc/prometheus /prometheus && \
    ln -s /prometheus /etc/prometheus/data

USER       nobody
EXPOSE     9090
VOLUME     [ "/prometheus" ]
WORKDIR    /etc/prometheus
ENTRYPOINT [ "/bin/prometheus" ]
```


## Utilisation des builds multi-stage
Permet de s'absoudre des tout l'environnement de build dans l'image finale, et ne conserver que l'application compilée. Voici un exemple en Golang.

hello.go
```golang
package main

import "fmt"

func main() {
	fmt.Println("Hello world!")
}
```

```Dockerfile
# build stage
FROM golang:alpine AS build-env
RUN mkdir /src
ADD hello.go /src
RUN cd /src && go build -o goapp

# final stage
FROM alpine
WORKDIR /app
COPY --from=build-env /src/goapp /app/
ENTRYPOINT ./goapp
```
Lancement de la construction de l'image docker
```shell
cd multistage-build
docker build -f Dockerfile -t multistage/hello .
docker run --rm multistage/hello
docker history multistage/hello
docker images
```
L'image golang:alpine fait 287Mo, quand l'image finale de l'application fait 5,71Mo


## Création d'images Docker sans Docker

Les images docker répondent à une spécification OCI (https://github.com/opencontainers/image-spec).
Il existe d'autres outils que Docker pour construire des images :

- JIB : utilisable depuis Maven, Gradle ou encore en java (https://github.com/GoogleContainerTools/jib)
- ...

# La face cachée de Docker : le réseau

## Ports et redirection de port

```shell
docker run -d -p 88:80 nginx-demo
```
On ne peut pas lancer deux fois cette commande, car `Bind for 0.0.0.0:88 failed: port is already allocated.`

Le container nginx de tout à l'heure tourne encore en tâche de fond, et monopolise le port 88 de la machine hôte sur son interface eth0, on ne peut donc pas en lancer un autre.

```shell
docker network ls
```
Le réseau nommé bridge fait le pont entre l'interface réseau eth0 et docker0 sur la machine hôte.

```shell
docker run -d -p 89:80 nginx-demo
```
On peut cependant changer de port (le 89 par exemple) et toujours rediriger dans le container sur le port 80.

## Réseaux docker
Il existe plusieurs types de réseau sous docker
- none : attaché à un container, celui-ci ne possèdera aucune interface réseau
- host : attaché à un container, celui-ci ne sera pas isolé et sera directement attaché à eth0
- bridge : créé un réseau isolé. tout container attaqué à ce réseau pourra voir un autre container de ce même réseau. il peut en exister plusieur par machine
- overlay : réseau prévu pour exister sur plusieurs machines. nécessite une base clé valeur (type etcd)

```shell
docker network create -d bridge my-bridge-network
```

```shell
docker run --rm alpine ifconfig
docker run --rm --network none alpine ifconfig
docker run --rm --network my-bridge-network alpine ifconfig
```

Il est possible de linker des containers ensembles :

```shell
docker run --name some-mongo -d mongo:latest
docker run -it --link some-mongo:mongo --rm mongo mongo --host mongo test
```
Cela aura pour effet d'ajouter l'entrée `172.17.0.2	mongo aa0be9536323 some-mongo` dans le fichier `/etc/hosts` du container client


# Le point noir de Docker : la persistance des données

Ecrire des données dans la layer en écriture d'un container n'est pas pérenne du tout !

## Utilisation de volumes


# Docker et la sécurité

## Elevation de privilèges

## Attention aux images mal intentionnées !
