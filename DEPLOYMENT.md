
# Docker-Compose Deployment

## Overview:

Desired Outcome of this Document is a Deployment of a Stack of Docker Containers providing a Production Backend that serves a WordPress Application via the FastCGI Application Server to the Web Server with built in FastCGI Cache & Lossless Compression. 

The Reverse Proxy Performs Secure Domain Name Validation, Auto TLS Management & Provides A Web Application Firewall along with CloudFlare as CDN.

Automation & Orchestration by Docker-Compose. Developer / Admin UI by Portainer.

This deployment is comprised of the following technologies: 
```env
I)    Alpine Linux OS : Container Operating System
II)   Debian Linux OS : Container Operating System
III)  MariaDB         : Relational Database Management System ( RDBMS )
IV)   RedisDB Cache   : In-Memory, Schema-Less Database Object Cache
V)    PHP 7.4         : Hypertext Preprocessor ( General-Purpose Scripting Language )
VI)   WordPress 5.6   : Content Management System ( CMS )
VII)  PHP-FPM         : Application Server ( FastCGI Process Manager )
VIII) Nginx           : Web Server ( FASTCGI_CACHE & Lossless Compression )
IX)   Caddy           : Reverse Proxy ( HTTP/3 | TLS Management | Application Firewall  )
X)    CloudFlare      : Content Distribution Network ( CDN )
XI)   Portainer       : Container Management UI
```

### Table of Content

* Architecture
* Requirements
* Preparation
* Clone Repository
* Environment Variables
* Create Directories
* Configure Caddyfile
* Deployment with Docker-Compose
* WordPress Installation & Configuration
* Update Wp-Config.Php
* Activate Plugins
* Portainer Admin UI
* Stoppage of Deployment
* Removal of Containers & Images

### Architecture

![tsuki_architecture_image](https://user-images.githubusercontent.com/75030055/100444533-db329c00-30ab-11eb-8cde-68061e918231.png)

### Requirements

* KVM, LXC or Similiar Cloud Hypervisor
* OS: Linux (Debian 10) with Root Privileges
* Docker and Docker-Compose installed on the host machine
* A Fully Qualified Domain Name (FQDN)
* Cloudflare "Orange-Cloud" DNS Record with your FQDN pointing to your server’s public IP address

### Preparation


* [Register a Fully Qualified Domain Name (FQDN)](https://www.namecheap.com/support/knowledgebase/article.aspx/10072/35/how-to-register-a-domain-name)

* [Create "Orange-Cloud" DNS Records](https://www.namecheap.com/support/knowledgebase/article.aspx/9607/2210/how-to-set-up-dns-records-for-your-domain-in-cloudflare-account/)


SSH into the host machine : 

    ssh root@PUBLIC_IP_ADDRESS


* [Create a secondary SUDO USER](https://linuxhint.com/create_new_sudo_user_debian10/)

* [Setup SSH access & enable NEW_USER public-key only access](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-debian-10)

* [Disable root SSH login](https://www.tecmint.com/disable-or-enable-ssh-root-login-and-limit-ssh-access-in-linux/)
    
Confirm Seconday User has SSH access from a new shell : 

    ssh NEW_USER@PUBLIC_IP_ADDRESS

* [Install Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10)

* [Install Docker-Compose](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-debian-10)

Enable Uncomplicated Firewall (UFW) :

    sudo apt install ufw
    sudo ufw allow ssh                        # Delete the ipv6 RULESET If you do not require it
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow http
    sudo ufw allow https
    sudo systemctl enable ufw
    sudo ufw enable
    sudo ufw status verbose

Install git :

    sudo apt install -y git

### STEP 1: Clone Repository

    git clone https://github.com/techwise-technologies/tsuki-webshop.git my-webshop
    cd my-webshop/

### STEP 2: Environment Variables

The .env file, stored as a hidden file in the main directory, requires your input. There is a .env_example that you can copy to start.

    cp .env_example .env

You now have a .env file. This file contains insecure default values for configuration options. 

During deployment this .env file is used to by YOUR WordPress Application to initialize the configuration files. 

The values you input are used for secure authentication between the Services running within the Docker Containers. It is important to keep a copy of the values you input as only you have them.

    nano .env

This opens the .env file with the nano editor and you are now required to define certain values.

Example `.env` file (default values):

```env
# Caddy v2
CADDY_CONF_DIR=./caddy/config
CADDY_DATA_DIR=./caddy/data
CADDYFILE=./caddy/Caddyfile
CADDY_VERSION=2-alpine

# Cloudflare
# CloudFlare 
CLOUDFLARE_EMAIL=email@yourdomain.com
CLOUDFLARE_AUTH_TOKEN=cloudflare_api_token

# Nginx
NGINX_CONF_DIR=./nginx/conf.d
NGINX_LOG_DIR=./logs/nginx
FASTCGI_CACHE_DIR=./cache
NGINX_VERSION=prod

# PHP Configs
TSUKI_PHP_CONF=./tsuki.ini

# WordPress
WORDPRESS_DB_PASSWORD=strong_password      <------------ EDIT THIS
WORDPRESS_VERSION=php7.4           
WORDPRESS_DB_NAME=yourdomain_com_wp        <------------ EDIT THIS   ## leave the trailing _wp 
WORDPRESS_DB_USER=yourdomain_com           <------------ EDIT THIS      
WORDPRESS_DATA_DIR=./wordpress
WORDPRESS_TABLE_PREFIX=wp_
WORDPRESS_DB_HOST=db

# Redis
REDIS_PASSWORD=very_very_strong_password   <------------ EDIT THIS
REDIS_CONF=./redis/redis.conf
REDIS_DATA_DIR=./redis/data/
REDIS_VERSION=alpine

# MariaDB
MYSQL_ROOT_PASSWORD=very_strong_password    <------------ EDIT THIS 
MYSQL_DATABASE=yourdomain_com_wp            <------------ EDIT THIS   ## leave the trailing _wp 
MYSQL_PASSWORD=strong_password              <------------ EDIT THIS
MYSQL_USER=yourdomain_com                   <------------ EDIT THIS
DB_DATA_DIR=./mariadb
MARIADB_VERSION=10.5

```
Replace all occurances of "yourdomain_com", "strong_password", "very_strong_password" & "very_very_strong_password" as desired. 

Exit the file by pressing and holding ctrl + x. 

This will initiate a prompt that will ask if you wish to save the changes, press Y and Enter. 

You now have a .env file ready to use for deployment.

### STEP 3: Create Directories

    mkdir -p cache/ mariadb/ wordpress/ logs/nginx/


### STEP 4: Configure Caddyfile
   
    nano Caddyfile

This opens the  Caddyfile with the nano editor and you are now required to input your domain name.

```env
{
    experimental_http3
}

yourdomain.com, www.yourdomain.com {
  reverse_proxy 127.0.0.1:8080
  tls email@yourdomain.com
  
  tls {
    dns cloudflare cloudflare_api_token
  }
}

devpanel.yourdomain.com {
  reverse_proxy 127.0.0.1:9000
  tls email@yourdomain.com
  
  tls {
    dns cloudflare cloudflare_api_token
  }
}
```

Replace all occurances of "yourdomain.com" with your Desired domain name and all occurances of "email@yourdomain.com" & "cloudflare_api_token" with your own token and email.
Exit the file by pressing and holding ctrl + x, this will initiate a prompt that will ask if you wish to save the changes, press Y and Enter. 

### STEP 5: Deployment with Docker-Compose

Time to Deploy, Run this Command and Watch the code Execute. It may take a few minutes, be patient.

    docker-compose up -d 


After a few moments your WordPress Application & Your Admin UI is ready to be configured.

WordPress Application: https://www.yourdomain.com 
Admin UI :             https://devpanel.yourdomain.com 

### STEP 6: WordPress Installation & Configuration

Go to your domain and Follow the Steps to install WordPress. Keep a Copy of the Username & Password, this shall be the Admin Credentials for the WordPress Application. 

Log into the WordPress Admin Panel.

Go To Plugins on the Left Menu & Delete the 2 default options (Hello Dolly & Akismet). 

Add 2 New Plugins By Till Krüss :

    I)  Redis Object Cache
    II) Nginx Cache

Before Activating the Plugins, we have one more Step. 

### STEP 7: Update Wp-Config.Php

Navigate to the wordpress folder and open the wp-config.php in nano

    cd wordpress/
    sudo nano wp-config.php

Add The Following Settings just above the MYSQL entries:

    // ** REDIS settings ** //
    define( 'WP_CACHE_KEY_SALT', 'wp-docker-redis');
    define( 'WP_REDIS_HOST', 'redis');
    define( 'WP_REDIS_PASSWORD', 'very_very_strong_password');      <------------ EDIT THIS

Replace very_very_strong_password with the SAME value you used previously for this key. 

Add This at the very Bottom of the file:

    /** Filesystem API Error Fix */
    define('FS_METHOD','direct');

Exit the file by pressing and holding ctrl + x. This will initiate a prompt that will ask if you wish to save the changes, press Y and Enter. 

### STEP 8: Activate Plugins

In The WordPress Admin UI, Navigate To Plugins and Activate Redis & Nginx Cache.

* Navigate to Redis Settings and "Enable Object Caching"

* Navigate to Nginx Settings and Set the Path to "/tmp/cache"

### STEP 9: Portainer Admin UI

Navigate to Your DevPanel URL and Folow the Instructions to Create an Admin User. Keep a Copy of the Credentials. Log into the Admin UI.

### Stoppage of Deployment

    docker-compose down

### Removal of Containers & Images

WARNING : THIS REMOVES ALL CONTAINERS & NETWORKS & IMAGES !!!

    docker stop $(docker ps -a -q)
    docker rm $(docker ps -a -q)
    docker image prune -a
    docker network prune


