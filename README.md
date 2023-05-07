# Topicos de telelmatica
# Reto 
## Integrantes
Santiago Gonzalez Rodriguez

Mariana Vasquez Escobar

# Requerimientos del proyecto (previos)

Contar con una cuenta educativa de AWS o los recursos económicos para costear la elaboración del proyecto. 

# Análisis de la solución
Los requisitos del proyecto estaban planteados de una manera muy específica en el enunciado del reto 4, siendo la única decisión creativa que tomamos el utilizar o no una imagen de docker desde DockerHub. Decidimos instalar apache y en este a su vez instalar Moodle, haciendo todas las conexiones al EFS y al RDS y posteriormente dockerizarlo, principalmente para aprender a hacerlo correctamente pues ya conocíamos cómo utilizar una imagen desde DockerHub.

# Implementación
Plataforma de moodle Dockerizada montada en AWS, junto a un balanceador de carga, un grupo de autoscaling para escalabilidad, una base de datos para la recoleccion de la informacion del moodle y un EFS, todos los servicios anteriores son brindados por AWS.

# Diseño

![Untitled Diagram drawio (1)](https://user-images.githubusercontent.com/68928428/236653252-5b55f8c3-08a0-4638-a8e1-891e62e4b578.png)

Se determinó un Auto-scaling group cuyo estado deseado fuera de dos instancias, su máximo fuera de 3 y su mínimo de 1. El auto-scaling group sería conectado a un Application Load Balancer que es Internet-facing a través de un target group que apunta al ASG. Cada una de las instancias está conectada a una misma base de datos y un mismo NFS. Puede ingresarse al sistema a través del DNS del Load Balancer.

# Guía de uso
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
# Auto-escalado

Para el presente reto se utilizó la herramienta de Auto-scaling grpup, proporcionada por AWS. Para poder conformar un auto-scaling group es necesario seguir los siguientes pasos:

## A. Crear una AMI

A partir de una instancia creada previamente, ya dockerizada y conectada al NFS y a la base de datos, se creará una Amazon Machine Image (AMI) con el fin de tener una base para crear nuevas instancias sin tener que "poblar" cada una cada que se cree una nueva máquina.

### Pasos para crear una AMI
1. Entre a la consola EC2 de AWS donde se listan todas las instancias creadas, elija la instancia a partir de la cual desea crear la AMI y presione clic derecho sobre ella. 

2. Vaya al apartado de "Images and Templates" y elija "Create Image".

![image](https://user-images.githubusercontent.com/68928428/236651779-f3e39db6-98ec-478f-b3d0-1c970185e189.png)

3. Será redirigido a la pantalla para crear AMI's. Dele un nombre y presione "Create image".

![image](https://user-images.githubusercontent.com/68928428/236651821-16aed132-cc55-4da1-952d-a63b8ed48b7b.png)

4. Remítase al menú de la izquierda de la pantalla e ingrese a la página de AMIs, donde podrá visualizar las Imágens previamente creadas, entre ellas, la que acaba de crear.

![image](https://user-images.githubusercontent.com/68928428/236651851-8ba777cb-29b2-4bab-8e9e-7966f768c9cc.png)

## B. Crear una Launch Template a partir de la AMI

Los Auto-scaling groups generan nuevas instancias a medida que son demandadas a partir de una Launch Template.

### Pasos para crear una Launch Template

1. Remítase a la página de Launch Template a partir del menú de la izquierda de la consola de AWS y de clic en "Create Launch Template".

![image](https://user-images.githubusercontent.com/68928428/236651941-60fe991c-83c5-4a5b-825b-8d7bf05427df.png)

2. Dele un nombre.

3. En el apartado de "Application and OS Images (Amazon Machine Image)" de clic en "My AMI's", luego en "Owned by me" y seleccione la AMI que acaba de crear.

![image](https://user-images.githubusercontent.com/68928428/236651983-6d51d045-84f3-4a1e-b568-c07ba4c1a05d.png)

4. Elija el mismo grupo de seguridad de la base de datos RDS.

![image](https://user-images.githubusercontent.com/68928428/236652048-676ae448-4f65-4457-a972-d71013e0b939.png)

5. Agregue un tag con un key value que recuerde, estos serán usados nuevamente para conectar con el Target Group y el balanceador de carga.

![image](https://user-images.githubusercontent.com/68928428/236652081-b7f61721-39b7-48be-8b74-20a5577498c4.png)

6. Clickee sobre "Create Launch Instance" luego de elegir las demás esoecificaciones de la instancia a crear.

## C. Cree un grupo de Auto-scaling

A partir de la plantilla creada, es posible crear un nuevo grupo de Auto-scaling. 

### Pasos

1. A través del menú de la izquierda, ingrese al apartado de "Auto-scaling groups" y dé clic en "Create auto-scaling group".

![image](https://user-images.githubusercontent.com/68928428/236652250-4c72dc7e-dcd3-4d7e-9407-79290980e81f.png)

2. Dele un nombre y seleccione la plantilla que quiere usar.

![image](https://user-images.githubusercontent.com/68928428/236652262-3e55acf1-e8a4-48bc-8f0f-31d777906257.png)

3. Elija las zonas en que estará disponible su AG.

4. Vincúlelo a un Load Balancer nuevo, de tipo Application Load Balancer e Internet-facing.

![image](https://user-images.githubusercontent.com/68928428/236652316-1902bc86-2688-480b-bcd6-5ab989bac522.png)

5. Vincule el balanceador de carga a un nuevo Target Group y dele un nombre; el TG será configurado más adelante.

![image](https://user-images.githubusercontent.com/68928428/236652346-27eef95a-a543-43bb-bca0-4cc7708ff20f.png)

6. Asígnele al Load Balancer el mismo Tag (key y value) que se le dió a la plantilla anteriormente.

![image](https://user-images.githubusercontent.com/68928428/236652375-60b2a7c5-216f-432a-a176-4c214ec96a56.png)

7. Delimite la cantidad de máquinas del ASG si así lo desea.

![image](https://user-images.githubusercontent.com/68928428/236652397-c0de782f-e23f-43be-a770-522187f12319.png)

8. Nuevamente, asígnele el tag usado previamente para que las nuevas instancias sean creadas con dicho tag.

![image](https://user-images.githubusercontent.com/68928428/236652411-b7fdfa65-f28c-4d2d-a39c-9be57a20de8c.png)

9. En el apartado de review dé clic a "Create new auto-scaling group".

![image](https://user-images.githubusercontent.com/68928428/236652428-a0228690-91be-4c7b-8959-a22e2c3357ea.png)

10. Será remitido a la consola de ASG, dé clic en el que acaba de crear y e ingrese a la pestaña "Instance management", allí podrá observar las instancias que se crean automáticamente. En este caso se estipuló que la cantidad deseada de instancias sería de dos.

![image](https://user-images.githubusercontent.com/68928428/236652546-92de65e7-1067-480f-8a13-ad8a331e2bbb.png)


## D. Verifique un Target Group

Si los tags fueron colocados correctamente, el Target group sabrá que debe apuntar a cada instancia creada automáticamente. A continuación, un vistazo al TG del reto 4, en el cual por fine de prueba se incluyó la instancia a partir de la cual se creó la AMI:

![image](https://user-images.githubusercontent.com/68928428/236652669-a25c05d6-2fbe-4c02-9bf5-b3a95cfa9aa3.png)

Si desea verificar los tags, puede ir a la respeciva pestaña:

![image](https://user-images.githubusercontent.com/68928428/236652687-67b842d3-6cd3-425a-9021-2ec714b71c83.png)

## E. Verificar el Load Balancer

Ahora solo queda entrar al pánel de LB desde el menú de la izquierda y verificar que esté apuntando al target group correspondiente.

![image](https://user-images.githubusercontent.com/68928428/236652735-2776a680-6601-4a6d-8dec-5971330b165c.png)


# Problematicas encontradas

## Superadas
1. El auto-scaling group intentaba generar las instancias pero estas se caian, caimos en cuenta que esto ocurria porque ingresabamos informacion dentro del tab de User Data.

2. Inicialmente no sabiamos como hacer que se conectaran a las diferentes instancias del moodle, eventualmente caimos en cuenta de que el balanceador de cargas tenia el DNS propio disponible para esta conexion.

3. El EFS no conectaba, esto ocurrio porque el security group generado no tenia admitido el puerto por el cual se hacia la conexion.

## No superadas
1. No logramos encontrar un uso adecuado para el NFS, aunque si lo hallamos conectado.

2. No contamos con el tiempo para encontrar y adquirir un dominio, pues el sistema de AWS no nos funcionaba y los dominios que teniamos anteriormente volvieron a no funcionar correctamente.
