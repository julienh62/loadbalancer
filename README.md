Ce fichier documente les étapes pour déployer deux instances WordPress (wordpress1 et wordpress2) derrière un serveur Nginx configuré en tant que reverse proxy avec redirection HTTPS. Il utilise Docker pour la gestion des services.
Prérequis
    rangé dans /srv/docker-easy   serveur VPS2
    Serveur avec Docker et Docker Compose installés.
    Nginx configuré avec les certificats SSL (Let's Encrypt ou auto-signés).
    Domaines ou sous-domaines configurés pour pointer vers votre serveur (par exemple jhennebo.be).
    
- Assurez-vous qu'aucune autre instance de Nginx ne tourne sur le serveur. Vous pouvez vérifier cela avec la commande suivante :
  ```bash
  sudo systemctl status nginx

Si Nginx est actif, vous devez l'arrêter avant de commencer :
ps aux | grep nginx

bash

sudo systemctl stop nginx
Structure du projet

root@ubuntu:/srv/docker-easy# tree -d -L 2 -p
[drwxr-xr-x]  .
├── [drwxr-xr-x]  html
├── [drwxr-xr-x]  mariadb_data
│   ├── [drwxr-xr-x]  mysql
│   ├── [drwxr-xr-x]  performance_schema
│   ├── [drwxr-xr-x]  sys
│   ├── [drwxr-xr-x]  wordpress1
│   └── [drwxr-xr-x]  wordpress2
├── [drwxr-xr-x]  nginx
│   ├── [drwxr-xr-x]  conf.d
│   ├── [drwxr-xr-x]  logs
│   └── [drwxr-xr-x]  ssl
└── [drwxr-xr-x]  wordpress
    ├── [drwxr-xr-x]  wp1
    └── [drwxr-xr-x]  wp2

14 directories
root@ubuntu:/srv/docker-easy# 
root@ubuntu:/srv/docker-easy# ls -l
total 40
-rw-r--r-- 1 www-data www-data 12299 Oct 26 08:36 README.md
-rw-r--r-- 1 www-data www-data  1070 Oct 26 07:59 docker-compose.yaml
drwxr-xr-x 2 www-data www-data  4096 Oct 26 07:59 html
drwxr-xr-x 7 lxd      docker    4096 Oct 26 08:19 mariadb_data
drwxr-xr-x 5 www-data www-data  4096 Oct 26 07:59 nginx
-rw-r--r-- 1 www-data www-data    12 Oct 26 07:59 readme.txt
drwxr-xr-x 4 www-data www-data  4096 Oct 26 08:14 wordpress
root@ubuntu:/srv/docker-easy# 
root@ubuntu:/srv/docker-easy/nginx# ls -l
total 12
drwxr-xr-x 2 www-data www-data 4096 Oct 26 07:59 conf.d
drwxr-xr-x 2 www-data www-data 4096 Oct 26 07:59 logs
drwxr-xr-x 2 www-data www-data 4096 Oct 26 07:59 ssl
root@ubuntu:/srv/docker-easy/nginx# cd ssl
root@ubuntu:/srv/docker-easy/nginx/ssl# ls -l
total 24
-rw-r--r-- 1 www-data www-data 6278 Oct 26 07:59 full_chain.pem
-rw-r--r-- 1 www-data www-data 2134 Oct 26 07:59 intermediate1.cer
-rw-r--r-- 1 www-data www-data 1938 Oct 26 07:59 intermediate2.cer
-rw-r--r-- 1 www-data www-data 1678 Oct 26 07:59 jhennebo.be_private_key.key
-rw-r--r-- 1 www-data www-data 2204 Oct 26 07:59 jhennebo.be_ssl_certificate.cer
root@ubuntu:/srv/docker-easy/nginx/ssl# cd ..
root@ubuntu:/srv/docker-easy/nginx# 

Étapes d'installation
1. Créer le fichier docker-compose.yaml

Le fichier docker-compose.yaml permet de gérer les conteneurs WordPress, MariaDB et Nginx.

yaml

version: '3'

services:
  db:
    image: mariadb:latest
    container_name: db
    environment:
      MYSQL_ROOT_PASSWORD: my_root_password
    volumes:
      - ./mariadb_data:/var/lib/mysql
    user: "www-data"
    networks:
      - wp-network

  wordpress1:
    image: wordpress:latest
    container_name: wordpress1
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp1
      WORDPRESS_DB_PASSWORD: wordpress1234
      WORDPRESS_DB_NAME: wordpress1
    volumes:
      - ./wordpress/wp1:/var/www/html
    depends_on:
      - db
    networks:
      - wp-network

  wordpress2:
    image: wordpress:latest
    container_name: wordpress2
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wp2
      WORDPRESS_DB_PASSWORD: wordpress1234
      WORDPRESS_DB_NAME: wordpress2
    volumes:
      - ./wordpress/wp2:/var/www/html
    depends_on:
      - db
    networks:
      - wp-network

  nginxrp:
    image: nginx:latest
    container_name: nginxrp
    ports:
      - "443:443"
      - "8080:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
      - ./html:/usr/share/nginx/html
    depends_on:
      - wordpress1
      - wordpress2
    networks:
      - wp-network

networks:
  wp-network:
    driver: bridge

2. Configuration Nginx

Créer un fichier de configuration pour Nginx dans nginx/conf.d/test.conf:

nginx

# Redirection HTTP vers HTTPS
server {
    listen 80;
    server_name jhennebo.be www.jhennebo.be;

    # Redirection vers HTTPS
    return 301 https://$host$request_uri;
}

# Configuration pour HTTPS et Reverse Proxy
server {
    listen 443 ssl;
    server_name jhennebo.be www.jhennebo.be;

    # Certificats SSL
    ssl_certificate /etc/nginx/ssl/full_chain.pem;
    ssl_certificate_key /etc/nginx/ssl/jhennebo.be_private_key.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Proxy pour wordpress1
    location /wordpress1/ {
        proxy_pass http://wordpress1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Proxy pour wordpress2
    location /wordpress2/ {
        proxy_pass http://wordpress2:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Serveur web statique par défaut (pour tester Nginx)
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

3. Générer les certificats SSL
créer le fichier full_chain.pem en utilisant les certificats telechargés. Étant donné que tu as plusieurs fichiers, tu vas combiner le certificat principal et les certificats intermédiaires dans le bon ordre.
Étapes pour Créer full_chain.pem

    Vérifier l’Ordre des Certificats : Assure-toi d'avoir les fichiers suivants :
        Certificat principal : jhennebo.be_ssl_certificate.cer
        Certificats intermédiaires : intermediate1.cer et intermediate2.cer
        Clé privée : jhennebo.be_private_key.key (non incluse dans full_chain.pem, mais nécessaire pour la configuration Nginx)

    Créer full_chain.pem : Utilise la commande cat pour concaténer le certificat principal avec les certificats intermédiaires. Voici comment faire :

    bash

cat jhennebo.be_ssl_certificate.cer intermediate1.cer intermediate2.cer >
Placer les certificats SSL dans nginx/ssl/ :

    full_chain.pem : Chaîne complète du certificat
    jhennebo.be_private_key.key : Clé privée associée au certificat


4. Gestion des Permissions dans Docker
    4.1. Permissions sur l'Hôte

    Assure-toi que les fichiers et répertoires que tu souhaites partager avec tes conteneurs Docker ont les bonnes permissions. Voici quelques commandes utiles :

    Vérifier les permissions d'un répertoire :

    bash

    ls -l /chemin/vers/ton/répertoire

   Changer le propriétaire d'un répertoire (par exemple, pour donner la propriété à l'utilisateur www-data qui est souvent utilisé par Nginx et PHP) :

   bash

   sudo chown -R www-data:www-data /chemin/vers/ton/répertoire

  chown -R www-data:www-data
   Modifier les permissions d'un répertoire (par exemple, pour donner des droits de lecture, écriture et exécution à l'utilisateur et aux groupes) :

   bash

  sudo chmod -R 755 /chemin/vers/ton/répertoire

  Pour des fichiers spécifiques, tu peux faire :

  bash

    sudo chmod 644 /chemin/vers/ton/fichier

  4.2 Permissions à l'Intérieur des Conteneurs

  Lorsque tu exécutes des conteneurs Docker, il est souvent nécessaire de s'assurer que les permissions sont correctes à l'intérieur du conteneur, notamment pour les applications web comme WordPress.

    Accéder à un conteneur en cours d'exécution :

    bash

   docker exec -it nom_du_conteneur bash

  Vérifier les permissions d'un répertoire à l'intérieur d'un conteneur :

  bash

  ls -l /chemin/vers/ton/répertoire

  Changer le propriétaire d'un répertoire à l'intérieur d'un conteneur (par exemple, pour un conteneur WordPress) :

  bash

  chown -R www-data:www-data /var/www/html

  Modifier les permissions d'un répertoire à l'intérieur d'un conteneur :

  bash

    chmod -R 755 /var/www/html

  4.3. Utilisation de Docker Compose

  Si tu utilises Docker Compose, tu peux définir les permissions directement dans ton fichier docker-compose.yaml :

  yaml

  services:
    wordpress:
      image: wordpress
      volumes:
        - ./wordpress/wp1:/var/www/html
      user: "www-data"  # Exécute le conteneur en tant qu'utilisateur www-data

   Cela permet de s'assurer que le conteneur WordPress s'exécute avec les permissions appropriées.
    
   4.4 Vérification des Permissions après Redémarrage

    Après avoir modifié les permissions ou redémarré les conteneurs, assure-toi de vérifier que tout fonctionne comme prévu. Tu peux le faire en consultant les logs des conteneurs :

    bash

    docker-compose logs





5. Démarrer les conteneurs

Dans le répertoire racine du projet, exécuter la commande suivante pour démarrer tous les services :

bash

docker-compose up -d

6. Configuration des bases de données WordPress

Se connecter à MariaDB pour créer les bases de données et utilisateurs pour les deux WordPress :

bash

docker exec -it db mysql -u root -p

# Créer la base de données et l'utilisateur pour wordpress1
CREATE DATABASE wordpress1;
CREATE USER 'wp1'@'%' IDENTIFIED BY 'wordpress1234';
GRANT ALL PRIVILEGES ON wordpress1.* TO 'wp1'@'%';

# Créer la base de données et l'utilisateur pour wordpress2
CREATE DATABASE wordpress2;
CREATE USER 'wp2'@'%' IDENTIFIED BY 'wordpress1234';
GRANT ALL PRIVILEGES ON wordpress2.* TO 'wp2'@'%';

7. Installer WordPress

    Accéder à https://jhennebo.be/wordpress1/wp-admin/install.php et suivre les instructions pour installer WordPress.
    Répéter pour https://jhennebo.be/wordpress2/wp-admin/install.php.

8. Ajuster les URLs WordPress (si nécessaire)

Vérifier les URL dans la base de données pour chaque instance WordPress et s'assurer qu'elles pointent vers les bons chemins :

bash

# Se connecter à MariaDB pour wordpress1
docker exec -it db mysql -u root -p

USE wordpress1;
SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl', 'home');

# Si nécessaire, les corriger
UPDATE wp_options SET option_value = 'https://jhennebo.be/wordpress1' WHERE option_name = 'siteurl' OR option_name = 'home';


9.modifier la pondération entre les sites 
# Groupe de serveurs pour l'équilibrage de charge avec pondération
upstream wordpress_backend {
    server wordpress1:80 weight=3; # Priorité plus élevée (plus de trafic)
    server wordpress2:80 weight=1; # Moins de trafic
}

# Redirection HTTP vers HTTPS
server {
    listen 80;
    server_name jhennebo.be www.jhennebo.be;

    # Redirection vers HTTPS
    return 301 https://$host$request_uri;
}

# Configuration pour HTTPS
server {
    listen 443 ssl;
    server_name jhennebo.be www.jhennebo.be;

    # Certificats SSL
    ssl_certificate /etc/nginx/ssl/full_chain.pem;
    ssl_certificate_key /etc/nginx/ssl/jhennebo.be_private_key.key;

    # Paramètres SSL recommandés
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Site principal
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }

    # Proxy vers le groupe de serveurs équilibrés (wordpress1 et wordpress2)
    location /wordpress1/ {
        proxy_pass http://wordpress_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /wordpress2/ {
        proxy_pass http://wordpress_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

Explication

    Pondération avec weight : La ligne server wordpress1:80 weight=3; signifie que wordpress1 recevra trois fois plus de trafic que wordpress2, qui a un poids de 1.
    Si vous souhaitez que le trafic soit 100% dirigé vers wordpress1 sauf en cas de panne, vous pouvez simplement mettre un poids très élevé pour wordpress1 et très bas pour wordpress2.

Résultat

    Avec cette configuration, le reverse proxy va privilégier wordpress1 en lui attribuant une proportion de trafic plus importante par rapport à wordpress2.

Vous avez dit :
Pondération avec weight : La ligne server wordpress1:80 weight=3; signifie que wordpress1 recevra trois fois plus de trafic que wordpress2, qui a un poids de 1. oui mais aprés ça dépend aussi des equetes des clients.   si le site wordpress 2 a plus de demande, que se passe t il?
ChatGPT a dit :
ChatGPT

C’est une bonne question ! Dans un scénario d’équilibrage de charge avec pondération, comme celui configuré avec weight, voici comment cela fonctionne en pratique :

    Distribution initiale du trafic :
        Avec server wordpress1:80 weight=3; et server wordpress2:80 weight=1;, NGINX va essayer de diriger environ 75% des requêtes vers wordpress1 et 25% des requêtes vers wordpress2.
        Cela signifie que pour chaque 4 requêtes, 3 seront envoyées à wordpress1 et 1 sera envoyée à wordpress2, en moyenne.
