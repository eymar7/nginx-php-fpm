# Implementación de NGINX y PHP-FPM Compilados


## Universidad
TECNOLOGICO DE ESTUDIOS SUPERIORES DEL ORIENTE DEL ESTADO DE MÉXICO

## Materia
SISTEMAS OPERATIVOS

## Proyecto
Implementación de un Servidor NGINX y PHP-FPM compilados desde código fuente

## Docente
Gustavo Moises Romero Gonzalez

## Integrantes
- ZAMORA PEÑA JHOANA
- MARMOLEJO MUÑOZ EYMAR URIEL


# Objetivo General

Implementar un servidor web utilizando NGINX versión 1.31.x y PHP versión 8.4.x compilados desde código fuente, configurando la comunicación mediante FastCGI usando sockets UNIX y administrando ambos servicios mediante SystemD.

---

# Objetivos Específicos

- Descargar y compilar NGINX desde código fuente.
- Instalar NGINX en la ruta `/srv/nginx`.
- Crear el usuario y grupo del sistema `nginx`.
- Configurar el servicio NGINX utilizando SystemD.
- Descargar y compilar PHP 8.4.x desde código fuente.
- Configurar PHP-FPM usando sockets UNIX.
- Habilitar soporte para procesamiento de imágenes, fechas e internacionalización.
- Configurar la comunicación FastCGI entre NGINX y PHP-FPM.
- Verificar el funcionamiento mediante un archivo `phpinfo.php`.
- Subir la documentación y configuración a un repositorio Git.

---

# Desarrollo del Proyecto

# 1. Actualización del sistema

Antes de iniciar la instalación se actualizaron los paquetes del sistema operativo.

```bash
sudo apt update && sudo apt upgrade -y
```

---

# 2. Instalación de dependencias

Se instalaron las dependencias necesarias para compilar NGINX y PHP desde código fuente.

```bash
sudo apt install build-essential curl wget tar gcc g++ make \
libpcre3 libpcre3-dev zlib1g zlib1g-dev \
libssl-dev libxml2-dev libsqlite3-dev \
libcurl4-openssl-dev libjpeg-dev libpng-dev \
libfreetype6-dev libonig-dev libzip-dev \
libicu-dev pkg-config autoconf bison re2c -y
```

---

# 3. Creación del usuario y grupo NGINX

Se creó el usuario y grupo del sistema que administrará el servicio web.

```bash
sudo groupadd nginx
sudo useradd -r -g nginx -s /usr/sbin/nologin nginx
```

---

# 4. Descarga y compilación de NGINX

## Descarga del código fuente

```bash
cd /usr/local/src
sudo wget https://nginx.org/download/nginx-1.31.0.tar.gz
sudo tar -xzvf nginx-1.31.0.tar.gz
cd nginx-1.31.0
```

## Configuración de compilación

```bash
sudo ./configure \
--prefix=/srv/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_stub_status_module
```

## Compilación e instalación

```bash
sudo make
sudo make install
```

## Verificación

```bash
/srv/nginx/sbin/nginx -v
```

---

# 5. Configuración del servicio NGINX en SystemD

Se creó el archivo:

```bash
sudo nano /etc/systemd/system/nginx.service
```

Contenido:

```ini
[Unit]
Description=NGINX Web Server
After=network.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/srv/nginx/sbin/nginx -s reload
ExecStop=/srv/nginx/sbin/nginx -s quit
User=nginx
Group=nginx

[Install]
WantedBy=multi-user.target
```

## Habilitar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl start nginx
```

## Verificar estado

```bash
sudo systemctl status nginx
```

---

# 6. Descarga y compilación de PHP 8.4.x

## Descarga del código fuente

```bash
cd /usr/local/src
sudo wget https://www.php.net/distributions/php-8.4.0.tar.gz
sudo tar -xzvf php-8.4.0.tar.gz
cd php-8.4.0
```

## Configuración de compilación

```bash
sudo ./configure \
--prefix=/srv/nginx/php \
--enable-fpm \
--with-fpm-user=php \
--with-fpm-group=nginx \
--with-zlib \
--with-curl \
--with-openssl \
--enable-mbstring \
--with-pdo-mysql \
--with-mysqli \
--with-jpeg \
--with-freetype \
--with-png \
--enable-gd \
--enable-intl
```

## Compilación e instalación

```bash
sudo make
sudo make install
```

---

# 7. Creación del usuario PHP

```bash
sudo groupadd php
sudo useradd -r -g php -s /usr/sbin/nologin php
```

---

# 8. Configuración de PHP-FPM

## Copiar archivos de configuración

```bash
cd /usr/local/src/php-8.4.0
sudo cp php.ini-development /srv/nginx/php/lib/php.ini
sudo cp sapi/fpm/php-fpm.conf /srv/nginx/php/etc/php-fpm.conf
sudo cp /srv/nginx/php/etc/php-fpm.d/www.conf.default /srv/nginx/php/etc/php-fpm.d/www.conf
```

## Editar archivo www.conf

```bash
sudo nano /srv/nginx/php/etc/php-fpm.d/www.conf
```

Modificar:

```ini
user = php
group = nginx
listen = /tmp/php84.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

---

# 9. Configuración del servicio PHP-FPM en SystemD

Crear archivo:

```bash
sudo nano /etc/systemd/system/php-fpm8.4.service
```

Contenido:

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
ExecStart=/srv/nginx/php/sbin/php-fpm --nodaemonize
ExecReload=/bin/kill -USR2 $MAINPID
User=php
Group=nginx

[Install]
WantedBy=multi-user.target
```

## Habilitar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable php-fpm8.4
sudo systemctl start php-fpm8.4
```

## Verificar estado

```bash
sudo systemctl status php-fpm8.4
```

---

# 10. Configuración FastCGI entre NGINX y PHP-FPM

Editar el archivo:

```bash
sudo nano /srv/nginx/conf/nginx.conf
```

Agregar:

```nginx
server {
    listen 80;
    server_name localhost;

    root /srv/nginx/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/tmp/php84.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

## Reiniciar servicios

```bash
sudo systemctl restart nginx
sudo systemctl restart php-fpm8.4
```

---

# 11. Creación de archivo de prueba PHP

Crear el archivo:

```bash
sudo nano /srv/nginx/html/phpinfo.php
```

Contenido:

```php
<?php
phpinfo();
?>
```

---

# 12. Verificación del funcionamiento

Se abrió un navegador web y se ingresó la dirección:

```text
http://localhost/phpinfo.php
```

Como resultado se mostró correctamente la página de información de PHP, comprobando:

- Funcionamiento de NGINX.
- Funcionamiento de PHP-FPM.
- Comunicación FastCGI.
- Uso del socket UNIX.
- Soporte de módulos PHP.

---


# Conclusiones

La implementación de NGINX y PHP-FPM desde código fuente permitió comprender el proceso completo de compilación, instalación y configuración manual de servicios web en Linux. También se logró configurar la comunicación mediante FastCGI utilizando sockets UNIX, mejorando el rendimiento y seguridad de la comunicación entre servicios.

Además, se aprendió el manejo de servicios SystemD, automatización de arranque y administración de usuarios y grupos dedicados para servicios web.

---

# Bibliografía

NGINX. (2026). *NGINX documentation*. Recuperado de https://nginx.org/en/docs/

PHP Documentation Group. (2026). *PHP manual*. Recuperado de https://www.php.net/docs.php

The Linux Foundation. (2026). *SystemD documentation*. Recuperado de https://systemd.io/

GitHub. (2026). *GitHub documentation*. Recuperado de https://docs.github.com/

Free Software Foundation. (2026). *GNU Make manual*. Recuperado de https://www.gnu.org/software/make/manual/

