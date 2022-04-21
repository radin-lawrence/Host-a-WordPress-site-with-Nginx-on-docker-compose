
# Host a WordPress site with Nginx on docker-compose

In this tutorial, I will guide you step-by-step to use docker-compose. We will deploy 'Wordpress' with Nginx, MySQL, and PHP-FPM. Each service has its own container, and we will use images from the docker hub registry. I will show you how to create containers from docker images and manage all containers with docker-compose.

![image](https://user-images.githubusercontent.com/100775027/163682867-a9b33d47-2a4d-4ac7-8689-f2e4da02c2dd.png)


## Features

- Faster and easier configuration
- Multiple isolated environments on a single host
- reserve volume data when containers are created

## Install docker

 Use the following command to install Dokcer

 ```bash
 sudo yum install docker -y
 ```
 When the installation is done, start docker and add it to start automatically at boot time:
 ```bash
 sudo systemctl start docker.service
 sudo systemctl enable docker.service
 ```
 If you want to avoid typing sudo whenever you run the docker command, add your username to the docker group:

 ```bash
sudo usermod -a -G docker <user-name>
 ```

To check the docker installation enter the command below.

```bash
docker version
```

##  Install Docker-Compose

Run this command to download the current stable release of Docker Compose:
```bash
 sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
 ```
> Note: To install a different version of Compose, substitute 1.29.2 with the version of Compose you want to use.

Apply executable permissions to the binary:
```bash
 sudo chmod +x /usr/local/bin/docker-compose
 ```
 Test the installation.
```bash
 docker-compose --version
 ```
**Create project Directory**

First, create a project directory for your project setup called project and navigate to it:
```bash
mkdir myproject && cd myproject
```
# Creating docker-compose.yml file and Defining Services

Create a docker-compose.yml file and add the below configuration to the file. 

**Add MySQL configuration**

The configuration for mysql database is listed under the service name database.

```bash
version: "3"

services: 
  database:
    image: mysql:latest
    container_name: database
    networks:
      - mynet
    volumes:
      - mydb:/var/lib/mysql/
    environment:
      - MYSQL_ROOT_PASSWORD=<your_root_password>
      - MYSQL_DATABASE=<your_wordpress_database_name>
      - MYSQL_USER=<your_wordpress_database_user>
      - MYSQL_PASSWORD=<your_wordpress_database_password>
    networks:
      mynet:
    volumes:
      mydb:
    restart: always


```

> mydb:/var/lib/mysql/ mounts the volume on the host machine mydb in the container path /var/lib/mysql.


**Add WordPress Configuration**

Add WordPress configuration to the docker-compose.yml file as a service named wordpress.

```bash
wordpress:
    image: wordpress:5.9-php7.4-fpm-alpine
    container_name: wordpress
    networks:
      - mynet
    volumes:
      - mywp:/var/www/html/
    environment:
      - WORDPRESS_DB_HOST=database
      - WORDPRESS_DB_USER=<your_wordpress_database_user>
      - WORDPRESS_DB_PASSWORD=<your_wordpress_database_password>
      - WORDPRESS_DB_NAME=<your_wordpress_database_name>
    networks:
      mynet:
    volumes:
      mywp:
    restart: always

```
Make sure the WordPress environment variables WORDPRESS_DB_NAME, WORDPRESS_DB_USER, and WORDPRESS_DB_PASSWORD match exactly with the MariaDB environment variables MYSQL_ROOT_PASSWORD, MYSQL_DATABASE, MYSQL_USER, and MYSQL_PASSWORD respectively. In case of any mismatch, the WordPress application would not be able to connect to the database.

>  mywp:/var/www/html/: we are mounting a named volume called mywp to the /var/www/html mountpoint


**Add NGINX Configuration**

Before running any containers, our first step will be to define the configuration for our Nginx web server.

```bash
mkdir nginx && cd nginx
vim nginx.conf
```

Add the below lines to the server section in nginx.conf to make sure all PHP requests are sent to the PHP-FPM service.

```bash
server {
    listen 80;
    server_name www.example.com example.com;


    root /var/www/html;
    index index.php;


    # log files
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files $uri =404;
	    fastcgi_pass wordpress:9000;
        fastcgi_index   index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }

}
```
> Note: Replace example.com with your domain name

Then we are going to add the NGINX Configuration to the docker-compose.yml file as a service named nginx.

```bash
nginx:
    image: nginx:latest
    ports: 
      - "80:80"
    container_name: nginx
    networks:
      - mynet
    volumes:
      - ./nginx:/etc/nginx/conf.d/
      - mywp:/var/www/html/
      - certbot-etc:/etc/letsencrypt
    networks:
      mynet:
    volumes:
      mywp:
      certbot-etc:
    restart: always

```

Make sure that the volume mentioned in the nginx configuration — mywp — matches with the volume mentioned in the WordPress configuration.

> ./nginx:/etc/nginx/conf.d: This will bind mount the Nginx configuration directory on the host to the relevant directory on the container, ensuring that any changes we make to files on the host will be reflected in the container.

> certbot-etc:/etc/letsencrypt: This will mount the relevant Let’s Encrypt certificates and keys for our domain to the appropriate directory on the container.

**Add SSL**

Finally, below your nginx definition, add your last service definition for the certbot service. Be sure to replace the email address and domain names listed here with your own information:

```bash
certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - mywp:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --staging -d example.com -d www.example.com
```

We’ve used depends_on to specify that the certbot container should be started once the nginx service is running.It also uses named volumes to share resources with the Nginx container, including the domain certificates and key in certbot-etc and the application code in wordpress.



We have finished docker-compose.yml file  and the final code will look like this:

```bash
version: "3"

services: 
  database:
    image: mysql:latest
    container_name: database
    networks:
      - mynet
    volumes:
      - mydb:/var/lib/mysql/
    environment:
      - MYSQL_ROOT_PASSWORD=<your_root_password>
      - MYSQL_DATABASE=<your_wordpress_database_name>
      - MYSQL_USER=<your_wordpress_database_user>
      - MYSQL_PASSWORD=<your_wordpress_database_password>

  wordpress:
    image: wordpress:5.9-php7.4-fpm-alpine
    container_name: wordpress
    networks:
      - mynet
    volumes:
      - mywp:/var/www/html/
    environment:
      - WORDPRESS_DB_HOST=database
      - WORDPRESS_DB_USER=<your_wordpress_database_user>
      - WORDPRESS_DB_PASSWORD=<your_wordpress_database_password>
      - WORDPRESS_DB_NAME=<your_wordpress_database_name>

  nginx:
    image: nginx:latest
    ports: 
      - "80:80"
    container_name: nginx
    networks:
      - mynet
    volumes:
      - ./nginx:/etc/nginx/conf.d/
      - mywp:/var/www/html/
      - certbot-etc:/etc/letsencrypt

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - mywp:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --staging -d example.com -d www.example.com

networks:
  mynet:

volumes:
  mywp:
  mydb:
  certbot-etc:

```

Save and close the file when you are finished editing.

With your service definitions in place, you are ready to start the containers and test your certificate requests.

##  Obtaining SSL Certificates and Credentials

Create the containers with docker-compose up and the -d flag, which will run the database, wordpress, and nginx containers in the background:

```bash
docker-compose up -d
```

![image](https://user-images.githubusercontent.com/100775027/163681961-40480483-a9d1-405c-ba9e-62c0da182906.png)

Using docker-compose ps, check the status of your services:

```bash
docker-compose ps
```

![image](https://user-images.githubusercontent.com/100775027/163682074-210cae71-c314-4daa-8ae4-46b8d0179422.png)


If everything was successful, your database, wordpress, and nginx services will be Up and the certbot container will have exited with a 0 status message:



> Note: If you see anything other than Up in the State column for the database, wordpress, or nginx services, or an exit status other than 0 for the certbot container, be sure to check the service logs with the command: "docker-compose logs service_name"

Find the section of the file with the certbot service definition, and replace the --staging flag in the command option with the --force-renewal flag, which will tell Certbot that you want to request a new certificate with the same domains as an existing certificate. The certbot service definition will now look like this:

```bash
certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - mywp:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --force-renewal -d example.com -d www.example.com

```

You can now run docker-compose up to recreate the certbot container. We will also include the --no-deps option to tell Compose that it can skip starting the nginx service, since it is already running:

```bash
docker-compose up --force-recreate --no-deps certbot
```
## Modifying the Web Server Configuration

Enabling SSL in our Nginx configuration will involve adding an HTTP redirect to HTTPS, specifying our SSL certificate and key locations. Since you are going to recreate the nginx service to include these additions, you can stop it now:

```bash
docker-compose stop nginx
```

Before we modify the configuration file itself, let’s first get the recommended Nginx security parameters from Certbot using curl:\

```bash
curl -sSLo /home/ec2-user/ngnx/nginx/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
```

Next, remove the Nginx configuration file you created earlier and create a new one:

```bash
rm -rf nginx/nginx.conf
vim nginx/nginx.conf
```

Add the following code to the file to redirect HTTP to HTTPS and to add SSL credentials. Remember to replace example.com with your own domain:

```bash
# Redirect HTTP -> HTTPS
server {
    listen 80;
    server_name www.example.com. example.com;

   
    return 301 https://example.com$request_uri;
}

# Redirect WWW -> NON WWW
server {
    listen 443 ssl http2;
    server_name www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    include /etc/nginx/conf.d/options-ssl-nginx.conf;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    root /var/www/html;
    index index.php;

    # SSL parameters
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    include /etc/nginx/conf.d/options-ssl-nginx.conf;

    # log files
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files $uri =404;       
        fastcgi_pass wordpress:9000;
        fastcgi_index   index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires max;
        log_not_found off;
    }

}
```

Before recreating the nginx service, you will need to add a 443 port mapping to your nginx service definition.

Open your docker-compose.yml file, add the following port mapping:

```bash
nginx:
    image: nginx:latest
    ports: 
      - "80:80"
      - "443:443"
    container_name: nginx
    networks:
      - mynet
    volumes:
      - ./nginx:/etc/nginx/conf.d/
      - mywp:/var/www/html/
      - certbot-etc:/etc/letsencrypt
```

he docker-compose.yml file will look like this when finished:

```bash
version: "3"

services: 
  database:
    image: mysql:latest
    container_name: database
    networks:
      - mynet
    volumes:
      - mydb:/var/lib/mysql/
    environment:
     - MYSQL_ROOT_PASSWORD=<your_root_password>
      - MYSQL_DATABASE=<your_wordpress_database_name>
      - MYSQL_USER=<your_wordpress_database_user>
      - MYSQL_PASSWORD=<your_wordpress_database_password>

  wordpress:
    image: wordpress:5.9-php7.4-fpm-alpine
    container_name: wordpress
    networks:
      - mynet
    volumes:
      - mywp:/var/www/html/
    environment:
      - WORDPRESS_DB_HOST=database
      - WORDPRESS_DB_USER=<your_wordpress_database_user>
      - WORDPRESS_DB_PASSWORD=<your_wordpress_database_password>
      - WORDPRESS_DB_NAME=<your_wordpress_database_name>

  nginx:
    image: nginx:latest
    ports: 
      - "80:80"
      - "443:443"
    container_name: nginx
    networks:
      - mynet
    volumes:
      - ./nginx:/etc/nginx/conf.d/
      - mywp:/var/www/html/
      - certbot-etc:/etc/letsencrypt

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot-etc:/etc/letsencrypt
      - mywp:/var/www/html/
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --force-renewal -d example.com -d www.example.com

networks:
  mynet:

volumes:
  mywp:
  mydb:
  certbot-etc:
  ```

Recreate the nginx service:

```bash
docker-compose up -d --force-recreate --no-deps nginx
```


With your containers running, you can now complete your WordPress installation through the web interface.

![image](https://user-images.githubusercontent.com/100775027/163681990-a5f22a45-d02f-4a1b-a4ab-b8dea151c6ab.png)



## Conclusion
This is how we create wordpress site using docker-compose tool with Nginx and PHP. Please contact me when you encounter any difficulty error while using this terrform code. Thank you and have a great day!
  
 ### ⚙️ Connect with Me
<p align="center">
<a href="https://www.linkedin.com/in/radin-lawrence-8b3270102/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>
<a href="mailto:radin.lawrence@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
