---
layout: post
title: "Configuration de Lizmap et QGIS Server sur nginx"
date: 2022-02-22
---

# Installation et configuration de Lizmap avec Nginx sur Debian 11 

Ce mémo décrit l'installation de QGIS-Server 3.16 et Lizmap 3.4 sur Debian 11 "Bullseye".

Installation effectuée en janv. 2022, avec la version 7.4 de PHP.

---



## Qu'allons nous installer ?

Nous allons installer :

- nginx - le serveur web
- PHP-FPM 7.4 - pour faire fonctionner les applis PHP sur le serveur web
- PostgreSQL et PostGIS - le système de base de données et son module spatial
- QGIS Server - le serveur OGC
- le client web Lizmap 3.4 - l'appli cartographique qui s'appuie sur PHP et QGIS Server
- [Facultatif] Maria-DB et WordPress 5 pour tester l'installation d'un CMS

### Différence entre Apache et nginx pour le fonctionnement de PHP

### Particularité de QGIS-Server avec nginx

Nous verrons comment générer des sockets QGIS Server

### Liens utiles : 

- <https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mariadb-php-lemp-stack-on-debian-10>
- <http://nginx.org/en/docs/beginners_guide.html>
- <https://docs.qgis.org/3.16/fr/docs/server_manual/getting_started.html>
- <https://oslandia.com/en/2018/11/23/deploying-qgis-server-with-systemd/>

---




## Installation et configuration NGINX + PHP-FPM

### Packages dans Debian 11

Nous installons Nginx, PHP-FPM 7, Maria-DB.

	sudo apt install nginx
	sudo apt install php-fpm php-mysql php-pgsql


### Test et rechargement de la configuration nginx

Pour vérifier le fonctionnement des nouveaux services :

	sudo apt systemctl status nginx
	sudo apt systemctl status php7.4-fpm

Pour arrêter nginx (stop = fast shutdown, quit = graceful shutdown) :

	sudo nginx -s quit

Pour démarrer nginx :

	sudo systemctl start nginx

Pour activer un nouveau virtual host : on crée un lien symbolique, on teste, on fait relire la configuration à nginx

	sudo ln -s /etc/nginx/sites-available/wp_local  /etc/nginx/sites-enabled/
	sudo nginx -t
	sudo systemctl reload nginx

**Remarque sur l'utilisateur www-data :** par défaut l'utilisateur de nginx est `www-data`, tout comme celui de PHP. (vérifier le propriétaire du fichier `/var/run/php/php7.4-fpm.sock ` : c'est bien www-data)


### Où sont situés les fichiers de nginx dans Debian ?

> Le **fichier de configuration principal** de nginx se trouve dans `/etc/nginx/nginx.conf`

Dans ce fichier est défini : 

- l'utilisateur de nginx = www-data
- nb de workers_connection = 768
- chemin fichiers log
- chemin des virtual hosts

> La configuration des virtual hosts se trouve dans `/etc/nginx/sites-avalaible`

> La page d'accueil (provisoire) de Nginx se trouve dans `/var/www/html`. 

> Les **fichiers de log nginx** sont situées dans `/var/log/nginx/access.log` et `/var/log/nginx/error.log` par défaut


### Structure des fichiers de configuration

**Directives simples VS directives bloc**. Les directives bloc commencent et se terminent avec des {}. Les directives simples se présentent sous la format d'un nom + paramètre et se terminent par un ';', elles sont inclues dans les directives bloc et de ce fait situées dans un _contexte_ particulier.

Hiérarchie des contextes :

	main
	  - events
	  - http
	    - server
	      - location


### Les processus de nginx

nginx a 1 processus **master** et N processus **worker**. Rôle du master : lire, évaluer la configuration et diriger les workers. Rôle des workers : traiter les requêtes. 

Nb de workers : ajustable, il est défini dans les fichiers de configuration (=768)

### Comment nginx traite les requêtes ?

Le bloc `http` peut contenir 1 ou N directives `server` (1 directive par port ...)

Dans le bloc `server`, on trouve la directive listen qui donne le port écouté ainsi que le nom du server.

nginx choisit quel `server` va traiter la requête avec `server_name`. Si la requête ne correspond à aucun nom de server, elle est dirigée vers le server _default_.

Cf. <https://nginx.org/en/docs/http/request_processing.html>


### Ordre d'interprétation des URI

nginx teste l'URI spécifiée dans l'entête de la requête http **avec les paramètres de la directive `location`**

Quand plusieurs `location` figurent dans le fichier de conf, nginx commence par tester :

1. les préfixes les + longs (exemple : `/images`, puis 
2. ceux avec une expression régulière (avec une tilde qui signale l'expression, exemple `~ \.(gif|jpg|png)$`) et termine par 
3. celui sans préfixe (avec juste `/`). 

La directive `root` qui indique le chemin des fichiers sur le disque peut se trouver dans le bloc **location** (lorsque plusieurs locations existent) ou dans le bloc **server** (lorsque 1 seule location : /).


### Exemple de config 1 : servir du contenu statique

Contenu statique = servir des fichiers (ex: images, html)


### Exemple de config 2 : le virtual host par défaut

Un exemple de configuration pour du contenu statique nous est donné par le virtual host par défaut.

**Virtual hosts par défaut** : défini dans `/etc/nginx/sites-avalaible/default`

Dans ce fichier, on peut voir que la page d'accueil (provisoire) de Nginx se trouve dans `/var/www/html`. 


extrait du virtual hosts par défaut :

```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;

	# Add index.php to the list if you are using PHP
	index index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
	}
```


### Exemple de config 3 : nginx comme simple proxy

proxy = recevoir les requêtes et les passer aux serveurs "proxied". Renvoyer leur réponse au client.

Exemple : 

```nginx
server {
    location / {
        proxy_pass http://localhost:8080;
    }
}
```

Dans ce cas, les requêtes reçues sur le port 80 sont routées vers 8080


### nginx comme proxy pour FastCGI

C'est le cas où nginx est utilisé pour router les requêtes vers une application serveur FastCGI (par exemple PHP)

La config la + basique inclut les directives : 

- `fastcgi_pass` pour router (au lieu de proxy_pass)
- `fastcgi_param` pour régler les paramètres envoyés à l'appli FastCGI

#### Création d'un hôte virtuel pour une appli PHP

Voici la configuration type pour une appli PHP telle que **Wordpress**.

Dans l'exemple suivant, lorsque l'URI se termine par '.php', les requêtes sont dirigées vers PHP 7.4. Remarquer également la direction include avec le fichier de directives `snippets/fastcgi-php.conf`

Nous créons un nouvel hôte virtuel : http://wp.localhost/

	sudo mkdir /var/www/wp_local
	sudo touch /etc/nginx/sites-available/wp_local

Contenu du fichier **wp_local**

``` nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/wp_local;
    index index.php index.html index.htm;

    server_name wp.localhost;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }
}
```

Test et rechargement de la configuration nginx

	sudo ln -s /etc/nginx/sites-available/wp_local  /etc/nginx/sites-enabled/
	sudo nginx -t
	sudo systemctl reload nginx

Ajout de wp.localhost dans /etc/hosts

	sudo nano /etc/hosts

Ajout d'un fichier avec phpinfo() pour tester le fonctionnement de PHP

	sudo nano /var/www/wp_local/info.php
	sudo chown -R www-data:www-data /var/www/wp_local

Pour tester, nous ouvrons dans le navigateur la page <http://wp.localhost/info.php>


### Snippets

La configuration adoptée pour tester PHP dans notre exemple est issue du fichier `/etc/nginx/snippets/fastcgi-php.conf`, qui lui-même inclut les directives de `/etc/nginx/fastcgi.conf`

Fichier **/etc/nginx/snippets/fastcgi-php.conf**

```nginx
# regex to split $uri to $fastcgi_script_name and $fastcgi_path
fastcgi_split_path_info ^(.+?\.php)(/.*)$;

# Check that the PHP script exists before passing it
try_files $fastcgi_script_name =404;

# Bypass the fact that try_files resets $fastcgi_path_info
# see: http://trac.nginx.org/nginx/ticket/321
set $path_info $fastcgi_path_info;
fastcgi_param PATH_INFO $path_info;

fastcgi_index index.php;
include fastcgi.conf;
```

Fichier **/etc/nginx/fastcgi.conf**

```nginx
fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;

fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  REQUEST_SCHEME     $scheme;
fastcgi_param  HTTPS              $https if_not_empty;

fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  REMOTE_USER        $remote_user;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;

# PHP only, required if PHP was built with --enable-force-cgi-redirect
fastcgi_param  REDIRECT_STATUS    200;
```

---




## Installation et configuration de QGIS Server

Cf. <https://qgis.org/en/site/forusers/alldownloads.html#debian-ubuntu> pour installer des paquets QGIS LTR sur Debian et Ubuntu

Liens utiles :

	sudo apt install python-qgis qgis-server

Pour tester l'installation :

	/usr/lib/cgi-bin/qgis_mapserv.fcgi


### Données pour tester QGIS Server

Avec Lizmap, il faudra choisir 1 ou plusieurs répertoires dans lequels seront rangés les projets QGIS et les données. Peu importe l'emplacement, le principal est que les droits d'accès soient bien configurés pour l'utilisateur `www-data`.

Dans ce tutoriel, nous créons le répertoire : /var/data/lizmap/. Nous configurerons 2 dépôts dans Lizmap (_repositories_), qui correspondront aux dossiers physiques : `/var/data/lizmap/public` et `/var/data/lizmap/prive`

Pour tester le bon fonctionnement de QGIS Server, nous copions dès maintenant 1 projet QGIS avec ses données : `/var/data/lizmap/public/fra_physical.qgs`.


### Nginx et FastCGI

Nginx ne créé pas automatiquement les processus FastCGI. Il faut les démarrer séparément. 

Remarque : le processus PHP est un cas spécial, il est auto-généré. Cf le script init : `/etc/init.d/php7.4-fpm`

Pour QGIS, 2 options se présentent à nous. 

- option 1 : **spawn-fcgi** ou **fcgiwrap**. Cas d'utilisation : notamment quand il n'y a pas de serveur X. Eviter fcgiwrap qui est plus facile à configurer mais beaucoup moins performant car il lance 1 processus à chaque requête.
- option 2 : **SystemD**. Requiert un serveur X pour fonctionner.


### Générer un socket QGIS Server

Pour QGIS Server, il y a 2 manières de créer un service basé sur un socket.

**Méthode spawn-fcgi** 

Il faut au préalable installer le paquet `spawn-fcgi`

	sudo apt install spawn-fcgi

Dans un 1er temps, si on veut simplement tester, on peut créer le processus qgis-server avec cette ligne de commande :

	spawn-fcgi -s /var/run/qgis-server.socket \
					-U www-data -G www-data -n \
					/usr/lib/cgi-bin/qgis_mapserv.fcgi

Si on souhaite installer qgis-server en tant que service, il faudra éditer un fichier de service `/etc/systemd/system/qgis-server.service` :

```service
[Unit]
Description=QGIS Server Service
After=network.target

[Service]
;; set env var as needed
;Environment="LANG=en_EN.UTF-8"
;Environment="QGIS_SERVER_PARALLEL_RENDERING=1"
;Environment="QGIS_SERVER_MAX_THREADS=12"
;Environment="QGIS_SERVER_LOG_LEVEL=0"
;Environment="QGIS_SERVER_LOG_STDERR=1"
;; or use a file:
EnvironmentFile=/etc/qgis-server/env

ExecStart=spawn-fcgi -s /var/run/qgis-server.socket -U www-data -G www-data -n /usr/lib/cgi-bin/qgis_mapserv.fcgi

[Install]
WantedBy=multi-user.target
```

Puis on active le service avec la commande : 

	sudo systemctl enable --now qgis-server

Voir aussi ci-dessous pour la création d'un fichier Environnement `/etc/qgis-server/env` pour QGIS Server.

### Un virtual host pour QGIS Server (via spawn-fcgi) et Lizmap

```nginx
server {
    listen 80;
    listen [::]:80;

    root /opt/lizmap-web-client/lizmap/www;
    index index.php index.html index.htm;

    server_name lizmap.localhost;

    location / {
        try_files $uri $uri/ =404;
    }

    location /qgis-server {
        gzip off;
        include fastcgi_params;
        fastcgi_pass unix:/var/run/qgis-server.socket;
    }    

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }
}
```


**Méthode SystemD**

> Création d'un fichier `/etc/systemd/system/qgis-server@.socket`

```service
[Unit]
Description=QGIS Server Listen Socket (instance %i)

[Socket]
Accept=false
ListenStream=/var/run/qgis-server-%i.sock
SocketUser=www-data
SocketGroup=www-data
SocketMode=0600

[Install]
WantedBy=sockets.target
```

> Création d'un fichier `/etc/systemd/system/qgis-server@.service`

```service
[Unit]
Description=QGIS Server Service (instance %i)

[Service]
User=www-data
Group=www-data
StandardOutput=null
StandardError=journal
StandardInput=socket
ExecStart=/usr/lib/cgi-bin/qgis_mapserv.fcgi
EnvironmentFile=/etc/qgis-server/env

[Install]
WantedBy=multi-user.target
```

> Création d'un fichier `/etc/qgis-server/env` pour les variables d'environnement de QGIS-Server

```conf
QGIS_PROJECT_FILE=/etc/qgis/myproject.qgs
QGIS_SERVER_LOG_STDERR=1
QGIS_SERVER_LOG_LEVEL=3
```

> Activer les services

	for i in 1 2 3 4; do systemctl enable --now qgis-server@$i.socket; done
	for i in 1 2 3 4; do systemctl enable --now qgis-server@$i.service; done

Après avoir redémarré, on peut consulter l'état des services de cette manière :

	sudo systemctl status qgis-server@1
	sudo systemctl status qgis-server@2
	sudo systemctl status qgis-server@3
	sudo systemctl status qgis-server@4

> Déactiver les services 

	for i in 1 2 3 4; do systemctl disable --now qgis-server@$i.service; done
	for i in 1 2 3 4; do systemctl disable --now qgis-server@$i.socket; done


### Un virtual host pour QGIS Server (via systemd) et Lizmap

```nginx
upstream qgis-server_backend {
    server unix:/var/run/qgis-server-1.sock;
    server unix:/var/run/qgis-server-2.sock;
    server unix:/var/run/qgis-server-3.sock;
    server unix:/var/run/qgis-server-4.sock;
}

server {
    listen 80;
    listen [::]:80;

    root /var/www/lizmap_local;
    index index.php index.html index.htm;

    server_name lizmap.localhost;

    location / {
        try_files $uri $uri/ =404;
    }

    location /qgis-server {
        gzip off;
        include fastcgi_params;
        fastcgi_pass qgis-server_backend;
    }    

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
    }
}
```

Pour tester le bon fonctionnement de QGIS Server : 

<http://lizmap.localhost/qgis-server?SERVICE=WMS&VERSION=1.3.0&MAP=/var/data/lizmap/public/fra_physical.qgs&REQUEST=GetCapabilities>

## Installation et configuration PostgreSQL / PostGIS

    sudo apt install postgis

Création d'un rôle superutilisateur admin_bdd / bdd_admin. 
Puis, création de la base de données `exemple`.

Pour se connecter en tant que `postgres`

    sudo -i -u postgres
    createuser -P -s admin_bdd
    createdb -O admin_bdd exemple
    psql -h localhost -U admin_bdd -c "CREATE EXTENSION postgis;"

Tester la configuration avec PHP ?


## Installation du client web Lizmap

### Liens utiles 

Les détails de l'installation figurent sur cette page : 

- <https://github.com/3liz/lizmap-web-client/blob/master/INSTALL.md>
- <https://docs.lizmap.com/next/fr/install/linux.html>

### Packages Debian

Installer `curl` ainsi que les extensions PHP requises par Lizmap.

	apt install curl php-sqlite3 php-gd php-xml php-curl

### Client web Lizmap 

Pour notre installation, nous téléchargeons la version 3.4.9 sur la page des releases :

<https://github.com/3liz/lizmap-web-client/releases>

Nous dézippons les fichiers et les copions dans `/opt/lizmap-web-client-3.4.9`, et nous créons un lien symbolique sans le numéro de version :

	ln -s /opt/lizmap-web-client-3.4.9 /opt/lizmap-web-client

**Remarque** : les fichiers de l'application sont donc accessibles à cet emplacement : `/opt/lizmap-web-client/lizmap/www`

	cd /opt/lizmap-web-client-3.4.9/lizmap/var/config
	cp lizmapConfig.ini.php.dist lizmapConfig.ini.php
	cp localconfig.ini.php.dist localconfig.ini.php
	cp profiles.ini.php.dist profiles.ini.php

Dans le fichier `lizmapConfig.ini.php`, modification de cette ligne :

	wmsServerURL="http://lizmap.localhost/qgis-server"

Puis nous lançons l'installation après avoir redéfini les droits :

	cd /opt/lizmap-web-client-3.4.9
	sudo lizmap/install/set_rights.sh www-data www-data
	sudo php lizmap/install/installer.php	


### Un virtual host pour QGIS Server (via spawn-fcgi) et Lizmap

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name lizmap.localhost;
    root /opt/lizmap-web-client/lizmap/www;
    index index.php index.html index.htm;

    # compression setting
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 5;
    gzip_min_length 100;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript text/json;

    location / {
        try_files $uri $uri/ =404;
    }

    location /qgis-server {
        gzip off;
        include fastcgi_params;
        fastcgi_pass unix:/var/run/qgis-server.socket;
    }    

    location ~ [^/]\.php(/|$) {
       fastcgi_split_path_info ^(.+\.php)(/.*)$;
	   # because of bug http://trac.nginx.org/nginx/ticket/321
       set $path_info $fastcgi_path_info; 
       try_files $fastcgi_script_name =404;
       include fastcgi_params;

       fastcgi_index index.php;
       fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       fastcgi_param PATH_INFO $path_info;
       fastcgi_param PATH_TRANSLATED $document_root$path_info;
       fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
       fastcgi_param SERVER_NAME $http_host;
    }
}
```

### Configuration Lizmap

Rendez-vous à l'URL : http://lizmap.localhost/. La page d'accueil de Lizmap doit s'afficher, mais aucune carte n'est encore configurée.

Il faut dans un premier temps changer le mot de passe **admin**.

Se connecter à l'interface d'administration avec `admin` / `admin`, puis modifier le mot de passe.

L'étape suivante consiste à configurer un répertoire (_repositories_) dans Lizmap.

Se connecter à l'interface d'administration dans la partie _Gestion des cartes_ (<http://lizmap.localhost/admin.php/admin/maps/>) et cliquer sur _Ajouter un répertoire_ :

	Identifiant = public
	Nom affiché = Tutoriel
	Chemin vers le répertoire local = /var/data/lizmap/public

Définition des droits sur le nouveau répertoire :

	Visualiser les projets du répertoire = anonymous, admins, users

---




## Installation et configuration d'un CMS [Facultatif]

### Maria DB

	sudo apt install mariadb-server
	sudo mysql_secure_installation

> entrer 'N' pour les sockets puis 'N' pour le nouveau mot de passe root. Puis entrez 'Y' pour les questions suivantes (supprimer BD test et utilisateur anomyne).

Se connecter à MariaDB pour créer une nouvelle base **wp_local** et un nouvel utilisateur **bernard**

	CREATE DATABASE wp_local;
	GRANT ALL ON wp_local.* TO 'admin_bdd'@'localhost' IDENTIFIED BY 'bdd_admin' WITH GRANT OPTION;
	FLUSH PRIVILEGES;

### Installation de Wordpress 5.8.3

Pour tester le bon fontionnement de PHP et MariaDB, nous installons le CMS Wordpress. Les fichiers sont téléchargées sur wordpress.org et décompressés.

- Installation des fichiers dans le dossier /var/www/wp_local, avec la base de données précédemment créée dans Maria DB (wp_local)
- Titre du site : Mon site WP
- Identifiant : admin_wp
- Mot de passe : wp_admin
- Adresse email : cyril-jb à laposte.net



