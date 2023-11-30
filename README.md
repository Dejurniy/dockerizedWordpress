# Setup Wordpress using Docker Compose

**Key components of the project**

+ Configuring **Docker Compose** file
+ Configuring **NGINX**
+ Generateing SSL/TLS certificates for **HTTP** to **HTTPS** redirection
+ Configuring Basic Authentication for **wp-admin** page

<ins>**Make sure you have installed Docker and Docker Compose of the latest versions!**</ins> 

<ins>**Also all the created configuration files will after be mounted into services in docker-compose.yaml file!**</ins>

## Instructions

### 1. Setup environment variables 

Copy **[.env.example](.env.example)** to **.env** in the project main directory and assign your values to variables. After of course execute **`source .env`** command to actually declare the variables in your environment.

**Example.**
 ```
 # Variables for MySQL
DB_ROOT_PASSWORD=mysql_root
DB_NAME=wordpress
DB_USER=wpressuser
DB_PASSWORD=wpressuser_

# Variables for Wordpress
WP_HOST=mysql_db:3306
WP_NAME=wordpress
WP_USER=wpressuser
WP_PASSWORD=wpressuser_
 ```

### 2. Create Certifications fro HTTP to HTTPS redirection.
<details>
 <summary>Option 1 #Create manually</summary>

##### You can refer to this **[link](https://devopscube.com/create-self-signed-certificates-openssl/)** to create SSL/TLS certificates manually for **HTTPS** redirection.
</details>

<details>
 <summary>Option 2</summary>

  You also can generate certificates by using **`mkcert`** command.

 Here are the steps how to generate certificates.

 1. First install **`mkcert`** into your system (For example I am using ubuntu:22.04)
 ```
  sudo apt update
  sudo apt install mkcert -y
 ```

 2. After installing **`mkcert`**, execute following commands to generate certifications. It will create your certificate and private key in your current directory.

 ```
 mkcert -install # Generates local CA certificate.
 mkcert <domain> <IP Address> # Creates certificates with your domain and IP Address.
 ```
</details>

### 3. Configure NGINX
After creating certificates, you need to configure your **NGINX**. First install necessary packages to create **.htpasswd** file for configuring wp-admin page.
+ Install **`apache2-utils`**
```
sudo apt update
sudo apt install apache2-utils -y
```
+ Create **`.htpasswd`** file
```
sudo htpasswd -c /path/to/.htpasswd <your_username>
# It will promt you to write a password for it.
```

+ After creating .htpasswd file, create the [nginx.conf](nginx.conf) file and do the magic.
```
  server {
      listen 80;
      listen [::]:80;
      server_name <your_domain> www.<your_domain>;
      return 301 https://$server_name$request_uri;

  }

  server {
      listen 443 ssl;

      ssl_certificate /path/to/certificate;
      ssl_certificate_key /path/to/private/key;

      server_name <your_domain> www.<your_domain>;
      access_log /var/log/nginx/access.log;
      error_log /var/log/nginx/errors.log;

      root /var/www/html/;
      index index.php index.html index.htm;

      location / {
          try_files $uri $uri/ /index.php$is_args$args;
      }

      location ~ \.php$ {
         include fastcgi_params;
         fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
         # Here we configured fastcgi_pass and gave it the wordpress service name and port of the wordpress image that we will use in the docker-compose file.
         fastcgi_pass wordpress:9000; 
      }

      location /wp-admin {
         try_files $uri $uri/ =404;
         auth_basic "WP Admin page";
         auth_basic_user_file /etc/nginx/.htpasswd;
      }
  }
```
+ After configuring NGINX, write your domain with IP Address in **/etc/hosts** file.
```
<your_domain> <IP_Address>
# For example
vikss.com 192.168.78.3
```


### 4. Configure Docker Compose

After configuring NGINX Finally you can configure **Docker Compose**.

+ Create **`docker-compose.yaml`** file.
```
version: '3'

services:
  wordpress:
    image: wordpress:6.4.1-php8.2-fpm
    container_name: wordpress
    environment:
      WORDPRESS_DB_HOST: ${WP_HOST}
      WORDPRESS_DB_NAME: ${WP_NAME}
      WORDPRESS_DB_USER: ${WP_USER}
      WORDPRESS_DB_PASSWORD: ${WP_PASSWORD}
    networks:
      - app-network
    depends_on:
      - mysql_db
    volumes:
      - ./wpress_data:/var/www/html/

  mysql_db:
    image: mysql:latest
    container_name: wpress_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql-volume:/var/lib/mysql
    networks:
      - app-network

  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/wpress.conf
      - ./server.crt:/etc/nginx/ssl/server.crt
      - ./server.key:/etc/nginx/ssl/server.key
      - ./wpress_data:/var/www/html/
      - ./.htpasswd:/etc/nginx/.htpasswd
    networks:
      - app-network

volumes:
  mysql-volume:
networks:
  app-network:
    driver: bridge
```
##### Things I want to mention.
+  NGINX and Wordpress services should have **shared** bind mounted directory or volume for nginx to actually have access to Wordpress files.
+  For Wordpress service we need to use an image with **php-fpm**.
+ All the configuration files that were created in your current directory need to be mounted into services as in [docker-compose.yaml](docker-compose.yaml) file.

### 5. Start the service
+ After finishing configuring your Wordpress, you can finally start the service.
```
docker-compose up -d
```
+ Then go to your browser and search your **domain**.


### That's it.
You have successfully configured Wordpress using Docker Compose!