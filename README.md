# Affichage de la version de docker (client et serveur)
```shell
docker version
```

# Docker, un produit client / server

Le démon docker est lancé en tant que service
```shell
service docker status
```

Le client docker effectue des appels sur l'API REST exposée par le démon sur un socket unix
```shell
curl --unix-socket /var/run/docker.sock http::/containers/json | jq .
```

# ======= Mon premier container ========
```shell
docker run hello-world
```
# Il s'est passé quoi ?
```shell
docker images
docker ps -a
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
docker run --name mongo-server -d mongo:latest
docker run -it --link mongo-server:mongodb --rm mongo mongo --host mongodb test
```
Cela aura pour effet d'ajouter l'entrée `172.17.0.2	mongodb aa0be9536323 mongo-server` dans le fichier `/etc/hosts` du container client


# Le point noir de Docker : la persistance des données

Ecrire des données dans la layer en écriture d'un container n'est pas pérenne du tout !

## Utilisation de volumes

Si l'on prend l'exemple d'une base de donnée :
```shell
docker run --name mongo-server -d mongo:latest
```

Contenu du fichier /tmp/user.json
```json
{name:"Julien"}
```

On importe des données dans la base mongo
```shell
docker run --rm --link mongo-server:mongodb -v ${PWD}/user.json:/tmp/user.json mongo mongoimport --host mongodb --collection users --file /tmp/user.json
```
On vérifie la présence des données en base
```shell
docker run --rm --link mongo-server:mongodb mongo mongo --host mongodb --eval 'db.getCollection("users").find({})'
```

Utilisation d'un volume docker
```shell
docker run --name mongo-server -v mongo-data:/data/db -d mongo
docker volume ls
```

Utilisation d'un montage sur l'hôte
```shell
docker run --name mongo-server -v /tmp/prez/data/db:/data/db -d mongo
ll /tmp/prez/data/db
```


# Docker et la sécurité

## Elevation de privilèges

Préparation d'un fichier secret
```shell
sudo su -
echo "ceci est un contenu secret uniquement accessible a root" > /tmp/secret.txt
chmod 600 /tmp/secret.txt
exit
```

Nous pouvons cependant voir le contenu du fichier avec un user standard

```shell
docker run --rm -v /tmp:/tmp alpine cat /tmp/secret.txt
```

## Attention aux images mal intentionnées !

Il est primordial de respecter certaines règles lors de l'utilisation d'images publiques :
- Utiliser en priorité les images officielles
- Toujours regarder le Dockerfile, et s'assurer que l'image a été générée avec ce Dockerfile
- Vérifier les ressources téléchargées via wget ou curl ou ADD
- Privilégier les images bien notées
- Si possible, se faire sa propre image, éventuellement en s'inspirant d'une existante


# Docker in Docker

## Utilisation du démon docker de l'hôte depuis un container
Même si l'on ne peut par à proprement parler de docker in docker, ce cas d'usage est très souvent utilisé.

```Dockerfile
FROM alpine:3.8
RUN apk update && apk add --no-cache docker
```

On build l'image contenant le client docker
```shell
docker build -t julien:docker-client .
```
On peut ainsi utiliser le client docker dans un container
```shell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock julien:docker-client docker version
```

## Lancement d'un vrai docker dans un container

Ce sujet était extrêment complexe, mais a été grandement simplifié par la communauté par la création d'une image "docker" :
```shell
docker run --privileged -d docker:dind
```
https://hub.docker.com/_/docker/

Mais lancer docker dans docker présente de multiples inconvénients. Pour n'en citer qu'un, le FS de type AUFS ne peut pas être lancé sur un FS AUFS, cela oblige à faire un montage du répertoire /var/lib/docker sur le FS de l'hôte, ou dans un volume docker.

http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
