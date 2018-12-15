# Affichage de la version de docker (client et serveur)
```shell
docker version
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

# ======= Concept de CONTAINER ========
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

## Modifications dans un container et commit dans une image
```shell
docker run alpine touch /tmp/mon-fichier.txt
docker run alpine ls /tmp
docker ps -a
docker diff <id container>

docker commit <id container> alpine-modifiee
docker images
docker history alpine-modifiee
```
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
docker search nginx
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
