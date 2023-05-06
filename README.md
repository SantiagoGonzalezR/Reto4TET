# Topicos de telelmatica
# Reto 
## Integrantes
Santiago Gonzalez Rodriguez
Mariana Vasquez Escobar

## Descripcion del proyecto
Plataforma de moodle Dockerizada montada en AWS, junto a un balanceador de carga, un grupo de autoscaling para escalabilidad, una base de datos para la recoleccion de la informacion del moodle y un EFS, todos los servicios anteriores son brindados por AWS.

# Como se realizo?
## Despliegue de moodle
El proyecto lo empezamos creando una base de datos RDS tipo mySQL en AWS, un sistema EFS que servira como NFS, y una instancia en EC2 donde podremos montar el Moodle.
Una vez entramos a la maquina del EC2 corremos los siguientes comandos:

``` bash
sudo apt update
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql php-gd php-xml php-mbstring php-curl php-zip php-xmlreader php-simplexml php-soap nfs-common cifs-utils
```
Bajada de los requerimientos para implementar el EFS, apache2 y sus librerias, las herramientas de mysql, php, y las librerias de php necesarias para moodle.

```bash
sudo mkdir efs
```
Para conectar el EFS, vamos al EFS que creamos y presionamos el boton conectar, copiando y pegando en la consola la segunda opcion que nos dan

```bash
sudo wget https://download.moodle.org/download.php/direct/stable39/moodle-latest-39.tgz
sudo tar -zxvf moodle-latest-39.tgz -C /var/www/html/
sudo chown -R www-data:www-data /var/www/html/moodle/
```
Bajamos y desempaquetamos moodle, despues de esto le modificamos los permisos para usarlo con apache sin problemas.

```php
$CFG->dbtype = 'mysqli';
$CFG->dbhost = 'endpoint de la BD';
$CFG->dbname = 'Nombre de la BD';
$CFG->dbuser = 'Usuario ppal de la BD';
$CFG->dbpass = 'Contraseña de la BD';
$CFG->prefix = 'mdl_';
$CFG->wwwroot = 'http://ip publica de la maquina  O  DNS del balanceador de carga';
$CFG->dataroot = '/var/www/html/moodledata';
```
Para wwwroot notamos que marcamos tanto la ip publica como el DNS del balanceador de cargas, la ip publica la ubicamos inicialmente con el proposito de verificar incialmente que todo corra bien, el DNS del balanceador lo ponemos cuando vamos a configurar el auto-scaling.
 
```bash
sudo mkdir /var/www/html/moodledata
sudo chown -R www-data:www-data /var/www/html/moodledata/
```
Creamos el directorio moodledata y le cambiamos los permisos para usar apache de manera mas libre.

```bash
sudo nano /etc/apache2/sites-available/moodle.conf
```
Creamos un archivo llamado moodle.conf dentro de la carpeta de apache para facilitar la conexion entre apache y moodle, dentro de este archivo escribimos lo siguiente:
```html
<VirtualHost *:80>
    ServerAdmin sgonzalez6@eafit.edu.co
    DocumentRoot /var/www/html/moodle
    ServerName demo-675289129.us-east-1.elb.amazonaws.com
    ServerAlias demo-675289129.us-east-1.elb.amazonaws.com

    <Directory /var/www/html/moodle>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Order allow,deny
        allow from all
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/moodle_error.log
    CustomLog ${APACHE_LOG_DIR}/moodle_access.log combined
</VirtualHost>

<Directory /var/www/html>
    AllowOverride All
</Directory>
```
Esto permite la comunicacion facil entre apache, moodle y todos los archivos dentro de la maquina.

```bash
sudo a2ensite moodle.conf
```
Habilitamos el moodle.

```bash
sudo systemctl restart apache2
```
Reiniciamos apache y abrimos la IP del servidor, donde nos deberia recibir la pagina de inicio de moodle.

Ya que tenemos el moodle creado, procedemos a configurar la pagina, es un proceso intuitivo entonces no lo mostraremos.

## Dockerizacion
Ahora procederemos a dockerizar el moodle:
```bash
sudo apt install docker.io
sudo systemctl start docker
sudo systemctl enable docker
```
Bajamos e iniciamos el docker para comenzar con todo el proceso de dockerizacion.

```bash
cd /var/www/html/moodledata
sudo nano Dockerfile
```
Accedemos a moodledata para crear el Dockerfile, donde pondremos lo siguiente:

```Dockerfile
FROM php:7.4-apache

RUN apt-get update && \
    apt-get install -y \
        libicu-dev \
        libxml2-dev \
        libpng-dev \
        libjpeg-dev \
        libfreetype6-dev \
        libzip-dev \
        zlib1g-dev \
        libldap2-dev \
        && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-configure ldap --with-libdir=lib/x86_64-linux-gnu/ \
    && docker-php-ext-install -j$(nproc) intl xml gd opcache pdo_mysql zip ldap bcmath sockets \
    && pecl install apcu \
    && docker-php-ext-enable apcu
    
COPY --chown=www-data:www-data . /var/www/html/

RUN chown -R www-data:www-data /var/www/html && \
    chmod -R 775 /var/www/html && \
    mkdir -p /var/www/html/moodledata && \
    chown -R www-data:www-data /var/www/html/moodledata && \
    chmod -R 777 /var/www/html/moodledata && \
    rm -rf /var/www/html/install.php
```

```bash
sudo docker build -t imagen-moodle .
sudo killall apache2
sudo docker run -d -p 80:80 -p 443:443 --name contenedor-moodle -v /var/moodledata:/var/www/html/moodledata imagen-moodle
```
Creamos la imagen del moodle, matamos las instancias del apache para liberar el puerto 8080 y corremos el contenedor


Si se generan errores a la hora de conectarse al moodle, se corren los siguientes comandos:
```bash
sudo chown -R www-data:www-data /var/www/html/moodledata
sudo chmod -R 775 /var/www/html/moodledata
sudo docker stop contenedor-moodle
sudo docker rm contenedor-moodle
sudo docker run -d --name contenedor-moodle \
    -p 80:8080 -p 443:443 \
    -e MOODLE_DATABASE_TYPE=mysqli \
    -e MOODLE_DATABASE_HOST=reto4.c8tzzolejqxd.us-east-1.rds.amazonaws.com \
    -e MOODLE_DATABASE_PORT_NUMBER=3306 \
    -e MOODLE_DATABASE_USER=admin \
    -e MOODLE_DATABASE_PASSWORD=moodleee \
    -e MOODLE_DATABASE_NAME=moodle \
    [ID de la imagen a usar]
```
Aca modificamos los permisos de la carpeta moodledata y recreamos el contenedor de moodle con settings especiales para la conexion con el internet.

Si despues de esto no se muestra la pagina reiniciaremos apache2 de la siguiente manera:
```bash
sudo docker exec reto4-moodle-container service apache2 restart
sudo service apache2 restart
```