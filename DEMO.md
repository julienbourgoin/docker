#!/bin/bash

# Affichage de la version de docker (client et serveur)
docker version

# Pull d'une image
docker pull alpine:3.8

# Création d'une image
docker build -f Dockerfile -t alpine-demo .

# Liste des images en cache local
docker images

# Lancement d'un container basé sur notre image
docker run alpine-demo ls /tmp

# Lancement d'un container basé sur notre image avec une console TTY
docker run -ti alpine-demo
/ # ls /tmp
mon-fichier.txt
/ # touch /tmp/test.txt

# Création d'une nouvelle image à partir d'un container
docker ps -a
docker commit <id container> alpine-demo-modifiee

docker run alpine-demo-modifiee ls /tmp


