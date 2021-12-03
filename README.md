# Docker-PortFolio - *Déploiement site web personnel avec Docker*
> Année: 2021
> Autheur: Morgan MINBIELLE
> Temps de lecture: 3 minutes

# Docker
---
## Pré-requis

### *Projet*
> Cloner le répertoire git [`PortFolioV3`](github.com/GerlariMin/PortFolioV3/) sur le répertoire courant dans lequel on va exécuter les fichier dockerfile.
```console
user@host:~# apt-get update
user@host:~# apt-get install git
user@host:~# git clone "https://github.com/GerlariMin/PortFolioV3"
```

### *Paquets*

> Installer [`php` et `composer`](https://www.howtoforge.com/how-to-install-php-composer-on-debian-11/#:~:text=How%20to%20Install%20PHP%20Composer%20on%20Debian%2011,...%204%20Testing%20the%20PHP%20Composer%20Installation.%20) sur la machine (pour pouvoir mettre à jour les dépendances du projet).  
[`Guide détaillé ici (anglais)`](https://www.howtoforge.com/how-to-install-php-composer-on-debian-11/#:~:text=How%20to%20Install%20PHP%20Composer%20on%20Debian%2011,...%204%20Testing%20the%20PHP%20Composer%20Installation.%20)  
*Intallation de PHP*  
```console
user@host:~# apt-get update
user@host:~# apt-get upgrade
user@host:~# apt install php -y
user@host:~# apt install curl git unzip wget php-common php-zip php-cli php-xml php-gd php-mysql php-bcmath php-imap php-curl php-intl php-mbstring -y
```  
*Installation de composer*  
```console
user@host:~# php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"'
user@host:~# php composer-setup.php --install-dir=/usr/local/bin --filename=composer
user@host:~# chmod +x /usr/local/bin/composer
```  
*Mise à jour des dépendances*  
```console
user@host:~# cd PortFolioV3/ressources/
user@host:~# composer update
```

### *httpd.conf*

> Le projet contient un fichier `httpd.conf` configuré pour le projet [`PortFolioV3`](github.com/GerlariMin/PortFolioV3/).
> Si ce fichier ne convient pas, il suffit de créer un fichier `httpd.conf` à partir d'un fichier `httpd.conf` de base ou avec une autre configuration et ajouter les modification suivantes:   

```html
# Deny access to the entirety of your server's filesystem. You must
# explicitly permit access to web content directories in other
# <Directory> blocks below.
#
<Directory />
    AllowOverride all # Remplacer le none par all
    # Enlever, si présent, la ligne Require all denied
</Directory>
```

```html
# DirectoryIndex: sets the file that Apache will serve if a directory
# is requested.
#
<IfModule dir_module>
    DirectoryIndex index.html index.php # Ajouter le index.php
</IfModule>
```  

```html
LoadModule proxy_module modules/mod_proxy.so # Décommenter cette ligne
```

```html
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so # Décommenter cette ligne
```

```html
# Bloc de code à ajouter
<VirtualHost *:80>
    # Configuration personnalisee pour accéder au conteneur de nom conteneur_php qui va traiter le code PHP
	ProxyPassMatch "^/(.*\.php(/.*)?)$" "fcgi://conteneur_php:9000/var/www/" enablereuse=on
</VirtualHost>
```

## Dockerfiles

### *DockerfileApache*

```dockerfile
FROM httpd:2.4

COPY ./PortFolioV3/ /usr/local/apache2/htdocs/
COPY ./conf/httpd.conf /usr/local/apache2/conf/httpd.conf
```

### *DockerfilePhp*

```dockerfile
FROM php:7.4-fpm

RUN apt-get update && apt-get install -y \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) gd
    
COPY ./PortFolioV3/ /var/www/
```

## Création des images

```console
user@host:~# docker build -t image_apache -f DockerfileApache .
```

```console
user@host:~# docker build -t image_php -f DockerfilePhp74 .
```

## Vérification des images docker

```console
user@host:~# docker image ls
```

## Création d'un réseau

```console
user@host:~# docker network create -d bridge connexion_custom
```

## Vérification des réseaux docker

```console
user@host:~# docker network ls
```

## Création des conteneurs

```console
user@host:~# docker run -d --name conteneur_php --network connexion_custom --rm image_php
```

```console
user@host:~# docker run -d --name conteneur_apache --network connexion_custom -p 8888:80 --rm image_apache
```

## Vérification des conteneurs docker

```console
user@host:~# docker container ls
```

## Vérification des processus docker en fonctionnement

```console
user@host:~# docker ps --all
```

## Test des conteneurs
> Sur le navigateur, saisir l'[`URL`](127.0.0.1:8888) suivante: [`127.0.0.1:8888`](127.0.0.1:8888).  

# Docker-compose
---
## Pré-requis
### *docker-compose*
> Installer [docker-compose](https://docs.docker.com/compose/). [`Guide complet d'installation ici`](https://docs.docker.com/compose/install/).
```console
user@host:~# sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

> Donner les droits d'exécution de `docker-compose` à l'ensemble des utilisateurs.
```console
user@host:~# sudo chmod +x /usr/local/bin/docker-compose
```

> Vérifier que le paquet est bien installé et sa version.
```console
user@host:~# docker-compose --version
```

## YAML
### *docker-compose.yml*
```yml
version: '3'

services:
    conteneur_php:
        image: image_php
        networks:
            - network_custom
        volumes:
            - ./PortFolioV3:/var/www/
        restart: always
    conteneur_apache:
        image: image_apache
        depends_on:
            - conteneur_php
        ports:
            - "8080:80"
        networks:
            - network_custom
        volumes:
            - ./PortFolioV3:/usr/local/apache2/htdocs/
            - ./conf/httpd.conf:/usr/local/apache2/conf/httpd.conf
        restart: always

networks:
  network_custom:
    driver: bridge
```

## Lancement de l'orchestrateur
```console
user@host:~# docker-compose up -d
```

## Vérification des conteneurs docker

```console
user@host:~# docker container ls
```

## Vérification des processus docker en fonctionnement

```console
user@host:~# docker ps --all
```