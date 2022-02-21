---
layout: post
title: "Configuration de Lizmap et QGIS Server sur nginx"
date: 2022-02-22
---

# Installation et configuration de Lizmap avec Nginx sur Debian 11 

Ce mémo décrit l'installation de QGIS-Server 3.16 et Lizmap 3.4 sur Debian 11 "Bullseye".

Installation effectuée en janv. 2022, avec la version 7.4 de PHP.




## Qu'allons nous installer ?

Nous allons installer :

- PostgreSQL et PostGIS
- QGIS et QGIS Server
- nginx, le serveur web
- PHP-FPM 7.4
- et enfin, le client web Lizmap 3.4
- [Facultatif] Maria-DB et WordPress 5 pour tester l'installation d'un CMS

### Différence entre Apache et nginx pour le fonctionnement de PHP

### Particularité de QGIS-Server avec nginx

Nous verrons comment générer des sockets QGIS Server

### Liens utiles : 

- <https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mariadb-php-lemp-stack-on-debian-10>
- <http://nginx.org/en/docs/beginners_guide.html>





## Installation de QGIS et QGIS Server

Cf. <https://qgis.org/en/site/forusers/alldownloads.html#debian-ubuntu> pour installer des paquets QGIS LTR sur Debian et Ubuntu

Liens utiles :

- <https://docs.qgis.org/3.16/fr/docs/server_manual/getting_started.html>
- <https://oslandia.com/en/2018/11/23/deploying-qgis-server-with-systemd/>

	sudo apt install python-qgis qgis-server

Pour tester l'installation :

	/usr/lib/cgi-bin/qgis_mapserv.fcgi




## Installation et configuration PostgreSQL / PostGIS

    sudo apt install postgis

Création d'un rôle superutilisateur admin_bdd / bdd_admin. 
Puis, création de la base de données `exemple`.

Pour se connecter en tant que `postgres`

    sudo -i -u postgres
    createuser -P -s admin_bdd
    createdb -O admin_bdd exemple
    psql -h localhost -U admin_bdd -c "CREATE EXTENSION postgis;"

Tester la configuration avec PHP.





## Installation et configuration Maria DB [Facultatif]

	sudo apt install mariadb-server
	sudo mysql_secure_installation

> entrer 'N' pour les sockets puis 'N' pour le nouveau mot de passe root. Puis entrez 'Y' pour les questions suivantes (supprimer BD test et utilisateur anomyne).

Se connecter à MariaDB pour créer une nouvelle base **wp_local** et un nouvel utilisateur **bernard**

	CREATE DATABASE wp_local;
	GRANT ALL ON wp_local.* TO 'admin_bdd'@'localhost' IDENTIFIED BY 'bdd_admin' WITH GRANT OPTION;
	FLUSH PRIVILEGES;


## Installation et configuration NGINX + PHP-FPM

### Packages dans Debian 11

Nous installons Nginx, PHP-FPM 7, Maria-DB.

	sudo apt install nginx
	sudo apt install php-fpm php-mysql php-pgsql

Vérifier le fonctionnement :

	sudo apt systemctl status nginx
	sudo apt systemctl status php7.4-fpm

### Où sont situés les fichiers dans Debian ?

**Fichier de configuration de nginx** : `/etc/nginx/nginx.conf`

Dans ce fichier est défini : 

- l'utilisateur de nginx = www-data
- nb de workers_connection = 768
- chemin fichiers log
- chemin virtual hosts

La configuration des virtualhosts se trouve dans `/etc/nginx/sites-avalaible`

**Virtual hosts par défaut** : défini dans `/etc/nginx/sites-avalaible/default`

Dans ce fichier, on peut voir que la page d'accueil (provisoire) de Nginx se trouve dans `/var/www/html`. 

**Remarque sur l'utilisateur www-data :** l'utilisateur de nginx est `www-data` tout comme celui de PHP. (Vérification faite, le propriétaire du fichier `/var/run/php/php7.4-fpm.sock ` est bien www-data)

**Fichiers de log nginx** : par défaut ils sont situées dans `/var/log/nginx/access.log` et `/var/log/nginx/error.log`

### Comment fonctionne nginx ?

nginx a 1 processus **master** et N processus **worker**.

Rôle du master : lire, évaluer la configuration et diriger les workers. 

Rôle des workers : traiter les requêtes. 

Nb de workers : défini dans les fichiers de configuration

### Structure fichier configuration

**Directives simples VS directives bloc**. Les directives bloc commencent et se terminent avec des {}.

Les directives simples inclues dans les directives bloc sont de ce fait situées dans un _contexte_ particulier.

Hiérarchie des contextes :

	main
	  - events
	  - http
	    - server
	      - location

### Servir du contenu statique

contenu statique = servir des fichiers (ex: images, html)

Processus :

1. nginx choisit quel `server` va traiter la requête
2. il teste l'URI spécifiée dans l'entête de la requête http **avec les paramètres de la directive `location`**

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

### Ordre d'interprétation des URI

Le bloc `http` peut contenir 1 ou N directives `server` (1 directive par port ...)

Dans le bloc `server`, on trouve la directive listen qui donne le port écouté.

Quand plusieurs `location` figurent dans le fichier de conf, nginx commence par tester les préfixes les + longs (exemple : `/images`, puis ceux avec une expression régulière (avec une tilde qui signale l'expression, exemple `~ \.(gif|jpg|png)$`) et termine par celui sans préfixe (avec juste `/`). 

La directive `root` peut se trouver dans le bloc **location** (lorsque plusieurs locations existent) ou dans le bloc **server** (lorsque 1 seule location : /).


### nginx comme simple proxy

proxy = recevoir les requêtes et les passer aux serveurs "proxied". Renvoyer leur réponse au client.

Exemple : 

```nginx
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

### nginx comme proxy pour FastCGI

C'est le cas où nginx est utilisé pour router les requêtes vers une application serveur FastCGI (par exemple PHP)

La config la + basique inclut les directives : 

- `fastcgi_pass` pour router (au lieu de proxy_pass)
- `fastcgi_param` pour régler les paramètres envoyés à l'appli FastCGI


### Création d'un hôte virtuel

> La page d'accueil (provisoire) de Nginx se trouve dans `/var/www/html`. 

> La configuration des virtualhosts se trouve dans `/etc/nginx/sites-avalaible`

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

Pour démarrer nginx :

	sudo systemctl start nginx

Pour arrêter nginx (stop = fast shutdown, quit = graceful shutdown) :

	sudo nginx -s quit

Ajout de wp.localhost dans /etc/hosts

	sudo nano /etc/hosts

Ajout d'un fichier avec phpinfo() pour tester le fonctionnement de PHP

	sudo nano /var/www/wp_local/info.php
	sudo chown -R www-data:www-data /var/www/wp_local

Pour tester, nous ouvrons dans le navigateur la page <http://wp.localhost/info.php>

### Exemple : nginx comme simple proxy

proxy = recevoir les requêtes et les passer aux serveurs "proxied". Renvoyer leur réponse au client.

Exemple : 

```nginx
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

### Exemple : nginx comme proxy pour FastCGI

C'est le cas où nginx est utilisé pour router les requêtes vers une application serveur FastCGI (par exemple PHP)

La config la + basique inclut les directives : 

- `fastcgi_pass` pour router (au lieu de proxy_pass)
- `fastcgi_param` pour régler les paramètres envoyés à l'appli FastCGI


### Snippets

La configuration adoptée pour tester PHP dans notre exemple est issue du fichier `/etc/nginx/snippets/fastcgi-php.conf`, qui lui-même inclut les directives inclues dans `/etc/nginx/fastcgi.conf`

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

### Installation de Wordpress 5.8.3

Pour tester le bon fontionnement de PHP et MariaDB, nous installons le CMS Wordpress. Les fichiers sont téléchargées sur wordpress.org et décompressés.

- Installation des fichiers dans le dossier /var/www/wp_local, avec la base de données précédemment créée dans Maria DB (wp_local)
- Titre du site : Mon site WP
- Identifiant : admin_wp
- Mot de passe : wp_admin
- Adresse email : cyril-jb à laposte.net




## Configuration de Nginx pour QGIS Server

### Nginx et FastCGI

Nginx ne créé pas automatiquement les processus FastCGI. Il faut les démarrer séparément. 

Remarque : le processus PHP est un cas spécial, il est auto-généré. Cf le script init : `/etc/init.d/php7.4-fpm`

Il y a 2 manières de remédier à cela.

- option 1 : utiliser **spawn-fcgi** ou **fcgiwrap**. Notamment quand il n'y a pas de serveur X. fcgiwrap est plus facile à configurer mais beaucoup moins performant.
- option 2 : **SystemD**. Requiert un serveur X pour fonctionner.

**Méthode spawn-fcgi** 

spawn-fcgi -s /var/run/qgis-server.socket \
                -U www-data -G www-data -n \
                /usr/lib/cgi-bin/qgis_mapserv.fcgi

**Méthode SystemD**

Création d'un fichier `/etc/systemd/system/qgis-server@.socket`

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

Création d'un fichier `/etc/systemd/system/qgis-server@.socket`

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

Création d'un fichier `/etc/qgis-server/env`

```conf
QGIS_PROJECT_FILE=/etc/qgis/myproject.qgs
QGIS_SERVER_LOG_STDERR=1
QGIS_SERVER_LOG_LEVEL=3
```

	for i in 1 2 3 4; do systemctl enable --now qgis-server@$i.socket; done
	for i in 1 2 3 4; do systemctl enable --now qgis-server@$i.service; done

Après avoir redémarré, on peut consulter l'état des services de cette manière :

	sudo systemctl status qgis-server@1
	sudo systemctl status qgis-server@2

Test 

<http://lizmap.localhost/qgis-server?SERVICE=WMS&VERSION=1.3.0&MAP=/var/data/lizmap/public/fra_physical.qgs&REQUEST=GetCapabilities>

Déactiver les services 

	for i in 1 2 3 4; do systemctl disable --now qgis-server@$i.service; done
	for i in 1 2 3 4; do systemctl disable --now qgis-server@$i.socket; done
