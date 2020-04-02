# Documentacao desafio 1

## Objetivo

O primeiro desafio tem como propósito a criação de 4 sites web utilizando tecnologias e plataformas distintas, sendo elas:
 * Um site estatico;
 * Um “servidor tomcat”
 * Um blog em wordpress
 * Uma loja virtual em magento
Possuindo as especificações técnicas abaixo:
 * O servidor deve ser um centos 7
 * versão do php 7
 * As aplicações deveriam estar em pastas separadas em `/var/www/html/[nome-do-site]`
 * O servidor web deve ser nginx
 * O tomcat deve ser instalado em `/opt`
 * Configuração de ssl “opcional”


## Instalação

Foi provisionado um servidor na aws em centos 7, para isso foi necessário localizar a Amazon Machine Image (AMI), com a versão de centos deseja. 
Após esse provisionamento manual  da instância na aws, foi realizado o acesso e com isso instalamos as dependências para que o desafio fosse concluído.

Processo de instalação das dependências:
1) yum update
2) yum install epel-release
3) Adicionado o repo do nginx
```
/etc/yum.repos.d/nginx.repo
    [nginx]
    name=nginx repo
    baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
    gpgcheck=0
    enabled=1
```
4) yum-config-manager --enable remi
5) yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
6) yum install nginx
7) curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup |  bash
8) yum -y install MariaDB-server MariaDB-client
9) Desabilitado selinux setenforce 0
10) Desabilitado firewalld
11) yum install php php-fpm php-mysql  php-soap php-intl php-mbstring php-xml php-gd unzip

## Configuração do php-fpm

Editado o arquivo `/etc/php-fpm.conf`, conteúdo:
```
[global]
error_log = /var/log/php-fpm.log
syslog.facility = daemon
syslog.ident = php-fpm
log_level = notice
emergency_restart_threshold = 0
emergency_restart_interval = 0
process_control_timeout = 0
process.max = 0
include=/etc/php-fpm.d/*.conf
```
Criados os arquivos de configuração para o magento e para o wordpress com os conteúdo abaixo:

### MAGENTO
`/etc/php-fpm.d/magento.conf`
```
[magento]

listen = 127.0.0.1:10000
listen.backlog = -1
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
user = www-data
group = www-data
pm = dynamic
pm.max_children = 2
pm.start_servers = 1
pm.min_spare_servers = 1
pm.max_spare_servers = 2
pm.process_idle_timeout = 10s
pm.max_requests = 200
pm.status_path = /php_status
ping.response = pong
access.log = /var/log/php-fpm/magento/access.log
access.format = %R – %u %t \”%m %r%Q%q\” %s %f %{mili}d %{kilo}M %C%%”
request_terminate_timeout = 60s
request_slowlog_timeout = 0
slowlog = /var/log/php-fpm/magento-slow.log
catch_workers_output = no
php_admin_value[error_log] = /var/log/php-fpm/magento/error.log
```

### WORDPRESS 
`/etc/php-fpm.d/wordpress.conf`
```
[wordpress]

listen = 127.0.0.1:11000
listen.backlog = -1
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
user = www-data
group = www-data
pm = dynamic
pm.max_children = 2
pm.start_servers = 1
pm.min_spare_servers = 1
pm.max_spare_servers = 2
pm.process_idle_timeout = 10s
pm.max_requests = 200
pm.status_path = /php_status
ping.response = pong
access.log = /var/log/php-fpm/wordpress/access.log
access.format = %R – %u %t \”%m %r%Q%q\” %s %f %{mili}d %{kilo}M %C%%”
request_terminate_timeout = 60s
request_slowlog_timeout = 0
slowlog = /var/log/php-fpm/wordpress-slow.log
catch_workers_output = no
php_admin_value[error_log] = /var/log/php-fpm/wordpress/error.log
```

## Instalação do MAGENTO

A instalação do magento foi realizada vai composer no diretório `/var/www/html/loja-blog-stevan` seguindo o processo abaixo:
* `php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"`
* `php composer-setup.php --install-dir=/usr/bin/ --filename=composer`
* `composer create-project --repository=https://repo.magento.com/ magento/project-community-edition magento`

Após a instalação do composer foram realizados os ajustes de permissionamento do diretório:
* `find var generated vendor pub/static pub/media app/etc -type f -exec chmod g+w {} +`
* find var generated vendor pub/static pub/media app/etc -type d -exec chmod g+w {} +`
* `chmod u+x bin/magento `

## Instalação do WORDPRESS

A instalação do wordpress foi realizada via tar.gz no diretorio  `/var/www/html/blog-stevan`
* `wget http://wordpress.org/latest.tar.gz`
* `tar -xzvf latest.tar.gz`

## Instalação TOMCAT

A instalação do wordpress foi realizada via zip  no diretorio  `/opt`
* `wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.33/bin/apache-tomcat-9.0.33.zip`
* `unzip apache-tomcat-9.0.33.zip`

## Instalaçao do certbot

Para a criação dos certificados ssl foi instalado o certbot para gerar os certificados do let’s encript
* `wget https://dl.eff.org/certbot-auto && chmod a+x certbot-auto`


## Certificado wildcard

Foi gerado um certificado wildcard para o dominio stevan.tk, assim é possível que todos os sites tenham ssl, essa configuração depende da criação de uma entrada de dns que o certbot solicita durante o processo:

* `./certbot-auto certonly --manual --preferred-challenges=dns --email=teste@okashi-scans.com.br --server https://acme-v02.api.letsencrypt.org/directory --agree-tos  -d *.stevan.tk `


## Configuração do nginx

Foi criada a configuração do nginx para cada um dos sites:

Nginx.conf

```
user www-data;
worker_processes auto;
worker_rlimit_nofile 1024;

error_log  /var/log/nginx/error.log warn;
#pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format main '[host: $host] - internal_ip:$remote_addr remote_ip:$http_x_forwarded_for logical_port:$remote_port - Upstream $upstream_addr [$time_local]  Request: [$request $status] ' - 'Nginx Upstream: [$upstream_response_time] ' - 'Request time: [$request_time] MSEC: [$msec]';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    server_tokens off;
    #tcp_nopush     on;

    keepalive_timeout  65;

    gzip  on;
    proxy_set_header        Host $host;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        Proxy "";
    proxy_headers_hash_bucket_size 64;
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

```
### site-stevan.stevan.tk
```
server {
    listen    80 default_server;
    server_name site-stevan.stevan.tk;

    location / {
        return 301 https://$server_name$request_uri;
   }
}


server {
    
    listen 443 ssl default_server;
    server_name site-stevan.stevan.tk;
    root /var/www/html/site-stevan;
    index index.php;
    access_log            /var/log/nginx/site/access.log;
    error_log             /var/log/nginx/site/error.log;
    ssl_certificate         /etc/letsencrypt/live/stevan.tk/fullchain.pem;
    ssl_certificate_key     /etc/letsencrypt/live/stevan.tk/privkey.pem;
    ssl_protocols TLSv1.2;
    location ~\.php$ {
        try_files $uri $uri/ =404;
        fastcgi_pass 127.0.0.1:11000;
        fastcgi_index index.php;
    }
}

```
### tomcat-stevan.stevan.tk

```
upstream tomcat_backend {
        server  127.0.0.1:8080;
}

server {
    listen    80;
    server_name tomcat-stevan.stevan.tk;

    location / {
        return 301 https://$server_name$request_uri;
   }
}

server {

        listen 443 ssl;
    access_log            /var/log/nginx/tomcat/access.log main;
      error_log             /var/log/nginx/tomcat/error.log;
        server_name tomcat-stevan.stevan.tk;
    ssl_certificate         /etc/letsencrypt/live/stevan.tk/fullchain.pem;
        ssl_certificate_key     /etc/letsencrypt/live/stevan.tk/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
    location ~ / {
        proxy_pass http://tomcat_backend;
    }
}
```
### blog-stevan.stevan.tk

```
server {
        listen       80;
        server_name  blog-stevan.stevan.tk;
        root         /var/www/html/blog-stevan/wordpress;
        index        index.php;
        # log files
        access_log /var/log/nginx/wordpress/access.log;
        error_log /var/log/nginx/wordpress/error.log;


  location ~ \.php$ {
    try_files $uri =404;
    fastcgi_pass 127.0.0.1:11000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
  }
}

```
### loja-blog-stevan.stevan.tk

```
upstream fastcgi_backend {
        server  127.0.0.1:10000;
}

server {

        listen 80;
    access_log            /var/log/nginx/magento/access.log main;
      error_log             /var/log/nginx/magento/error.log;
        server_name loja-blog-stevan.stevan.tk;
        set $MAGE_ROOT /var/www/html/loja-blog-stevan/magento;
        set $MAGE_MODE developer;
        include /var/www/html/loja-blog-stevan/magento/nginx.conf.sample;
}
```