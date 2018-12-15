# Affichage de la version de docker (client et serveur)
docker version

# ======= Concept de CONTAINER ========
# Premier lancement de container
docker run hello-world
# Il s'est passé quoi ?
docker images
docker ps -a
docker pull hello-world

# De quoi est fait hello-world ?
#https://github.com/docker-library/hello-world/blob/master/amd64/hello-world/Dockerfile
```Dockerfile
FROM scratch
COPY hello /
CMD ["/hello"]
```


# ======= Concept d'IMAGES ========

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

# Sauvegarde d'une image
docker save -o alpine-demo-modifiee.tar alpine-demo-modifiee



