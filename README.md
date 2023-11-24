# Antes de leer nada

No es un tutorial como tal, estoy centrado en terminar el Common core y por ahora no dedicare mucho tiempo a esta guia pero estos son apuntes que he ido haciendo y explicaciones. Si te sirven de ayuda para realizar tu proyecto genial. Todavia le falta mucho trabajo a este tutorial. 

# Inception-Tutorial
Tutorial to install Debian virtual machine with multiple virtualised Docker images

## 1- _Descargar imagen de la maquina virtual_ 💿

[Click aqui](https://www.debian.org/distrib/index.es.html) para redireccionarte a la URL donde puedes descargar la ISO de manera segura.

## 2- Instalacion de la maquina 🛠

Para realizar la instalación se requiere de un software de virtualización. En este tutorial haremos uso de [VirtualBox](https://www.virtualbox.org/). Si ya tienes VirtualBox instalado y dispones de la ISO Debian ya podemos empezar con el tutorial.

1 ◦ Debemos abrir VirtualBox y pinchar sobre ```New```



2 ◦ Escogemos el nombre de nuestra máquina y la carpeta donde estará ubicada. Importante introducir la maquina dentro de la carpeta sgoinfre ya que si no la ubicamos ahí nos quedaremos sin espacio y fallará la instalación (dependiendo del campus la ruta de sgoinfre puede cambiar).




## Archivo .yml

🧠 Que es un archivo yml❓

Es un archivo de configuración utilizado para definir y gestionar múltiples contenedores en un entorno Docker. Permite describir las relaciones, configuraciones y servicios que compondrán una aplicación o conjunto de servicios interconectados.


```
networks:
    gemartinnet:
        name: gemartinnet
        driver: bridge
```

En este apartado definimos las redes. En este caso definimos una red llamada ```gemartinnet``` con el controlador de red como ```bridge```. Esto lo que va a indicar esque sera una red de tipo bridge, lo que permitira a los contenedores comunicarse entre sí en el mismo host.

```
services:
    nginx:
        container_name: nginx
        build: ./requirements/nginx
        image: nginx
        ports:
        - 443:443
        volumes:
        - wordpress_data:/var/www/html
        restart: always
        networks:
        - gemartinnet
```

En este apartado se definen los servicios que ejecutaran los contenedores. El primer servicio es ```nginx```. Esto es lo que hacen las siguientes configuraciones:

container_name → Asigna un nombre especifico al contenedor que se crea a partir de este servicio.

build → Indica la ubicacion del Dockerfile y los archivos necesarios para construir la imagen del contenedor.

image → Indica a docker que imagen debe usar como base para el servicio que estas definiendo. Si la imagen no se encuentra a nivel local en el sistema docker la descargara automaticamente.

ports → Indicamos que queremos hacer un mapeo de puertos, esto es una redireccion de puertos de un sistema operativo a otro. Para explicarlo mejor, tenemos un servidor web (nginx) ejecutandose dentro del contenedor, por defecto no es accesible desde fuera del contenedor. Al hacer el mapeo de puertos especificaremos el puerto del servidor web para que este disponible desde el sistema anfitrion. Dicho esto ponemos 443:443 para indicar el PUERTO_DEL_HOST:PUERTO_DEL_CONTENEDOR. 

volumes → Creamos un volumen en el host al directorio que especifiquemos en el contenedor. El proceso de montar un volumen se asemeja a "conectar" una carpeta del sistema operativo anfitrión con una carpeta dentro del contenedor. Esto permite que los datos sean compartidos y persistan incluso después de que el contenedor se detenga o se reinicie. Es importante comentar que en esta parte solo diremos que el contenido del volumen llamado wordpress_data lo encontraremos en la ubicacion /var/www/html dentro del contenedor. La configuracion del volumen wordpress_data y mariadb_data es definida al final del fichero. Otra cosa importante a comentar es que WordPress y Nginx comparten volumen, esto lo utilizamos para servir archivos del sitio web desde el contenedor de NGINX, y si se actualizan o modifican esos archivos desde el contenedor WordPress los cambios se reflejarán inmediatamente en el sitio web sin necesidad de detener y volver a iniciar el contenedor de NGINX. Esto proporciona una forma eficiente de trabajar con aplicaciones web y mantener los datos consistentes.

restart → Indica como debe comportarse el contenedor en caso de que se detenga. Basicamente lo que decimos es que si el contenedor asociado al servicio Nginx se detiene por algun motivo Docker lo reiniciara automaticamente.

networks → Especifica a que red o redes debe estar conectado un contenedor. En nuestro caso le decimos que el contenedor asociado al servicio nginx estara conectada a la red gemartinnet.

```
mariadb:
    container_name: mariadb
    build:
       context: ./requirements/mariadb
    image: mariadb
    volumes:
     - mariadb_data:/var/lib/mysql
    restart: always
    networks:
     - gemartinnet
    env_file:
     - .env
```

Como todos los campos han sido explicados previamente menos env_file solo explicare este.

env_file → Permite especificar un archivo que contiene variables de entorno para ser utilizadas por el contenedor. Estas variables de entorno pueden ser utilizadas dentro del contenedor, por la base de datos MariaDB en este caso, para configurar el entorno de ejecución.

```
wordpress:
    container_name: wordpress
    build: ./requirements/wordpress
    image: wordpress
    depends_on:
     - mariadb
    volumes:
     - wordpress_data:/var/www/html
    restart: always
    networks:
     - gemartinnet
    env_file:
     - .env
```

Todos los campos han sido explicados previamente

```
volumes:
    mariadb_data:
        driver: local
        driver_opts:
            type: none
            device: /home/gemartin/data/mysql
            o: bind

    wordpress_data:
        driver: local
        driver_opts:
            type: none
            device: /home/gemartin/data/wordpress
            o: bind
```

Finalmente definimos los volumenes que hemos mencionado anteriormente. En estos campos especificaremos como seran manejados por Docker.

driver → Define el tipo de controlador que Docker utilizara para manejar este volumen. En este caso ponemos local, lo que significa que el volumen estara almacenado en el sistema local de archivos del host.

driver_opts → Es un conjunto de opciones especificas del controlador que permiten configurar como se va a manejar el volumen. En este caso establecemos 3 opciones: type, device y o. Vamos a explicar una por una.

- type → Especifica el tipo de sistema de archivos que utilizara para el volumen. El uso que doy (none) indica que no se debe aplicar un tipo de sistema de archivos especifico. 

- device → Especifica el directorio o dispositivo en el sistema de archivos del host que se utilizaran para almacenar los datos del volumen. 

- o (opciones de montaje) → Esta es una serie de opciones que se utilizan cuando se monta el volumen. En este caso, o: bind indica que se debe realizar un enlace directo (bind) entre el volumen y el directorio del host, lo que permite que los datos sean accesibles desde el sistema de archivos del host.


# Nginx

🧠 Que es un dockerfile❓ Es un archivo de configuración utilizado para construir una imagen de contenedor en Docker.

## Dockerfile 

```
FROM debian:10.11

RUN apt-get update
RUN apt-get install -y nginx openssl

EXPOSE 443

COPY ./conf/default /etc/nginx/sites-enabled/default
COPY ./tools/nginx_start.sh /var/www

RUN chmod +x /var/www/nginx_start.sh

ENTRYPOINT [ "var/www/nginx_start.sh" ]

CMD ["nginx", "-g", "daemon off;"]
```

FROM → Especifica la imagen base que se usara como punto de partida para construir el contenedor. En este caso utilizaremos Debian con la version 10.11. En un caso de uso real lo normal seria utilizar la imagen de nginx directamente pero el subject indica claramente que no se pueden utilizar imagenes preconfiguradas.

RUN → Este comando ejecuta el comando dentro del contenedor. En este caso el comando lo que hara es actualizar la lista de paquetes disponibles en los repositorios debian. El siguiente comando run lo que hara es instalar dos paquetes nginx y openssl, el flag -y indica que se debe aceptar automaticamente todas las confirmaciones.

EXPOSE → Especifica que puertos estan disponibles para ser expuestos cuando se ejecute el contenedor. En este caso sera el 443 aunque no tiene un efecto practico por si mismo, me explico, no abre el puerto ni realiza ningun enlace hacia fuera del contenedor. Sirve como informacion para el usuario sobre que puerto es utilizado por la aplicacion dentro del contenedor.

COPY → Copia un archivo de configuracion desde el directorio especificado a nivel local al directorio especificado dentro del contenedor. En este caso copiaremos la configuracion por defecto de nginx a la ruta ```/etc/nginx/sites-enabled/default``` dentro de nuestro contenedor. Acto seguido hacemos lo mismo, en este caso copiamos el script nginx_start.sh al directorio ```/var/www``` del contenedor.

RUN → Cambiaremos los permisos del script que acabamos de copiar al contenedor, de esta manera ya puede ser ejecutado.

ENTRYPOINT → Define el comando que se ejecutara cuando el contenedor se inicie. En este caso se ejecutara el script nginx_start.sh, que es el que acabamos de copiar y cambiar los permisos.

CMD → Establece el comando predeterminado que se ejecutara cuando el contenedor se inicie. En este caso lo que haremos sera iniciar el servidor nginx en modo demonio, esto lo que quiere decir es que el servidor se queda en ejecucion en segundo plano. La terminal esta libre para que el usuario, es decir, nosotros podamos utilizarla.

## Script Nginx

```
#!/bin/bash
if [ ! -f /etc/ssl/certs/nginx.crt ]; then
echo "Nginx: setting up ssl ...";
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout /etc/ssl/private/nginx.key -out /etc/ssl/certs/nginx.crt -subj "/C=ES/ST=Barcelona/L=Barcelona/O=wordpress/CN=gemartin.42.fr";
echo "Nginx: ssl is set up!";
fi
exec "$@"
```

Resumidamente , lo que hara este script es verificar si el certificado SSL (nginx.crt) existe en la ubicacion especificada. Si no existe lo generearemos y luego ejecutaremos los comandos que reciba como argumentos. Solo entrare en la explicacion de los campos escogidos para la generacion del certificado.

-x509 → Indica que se debe generar un certificado autofirmado.

-nodes → Significa que no se utilizara contraseña.

-days 365 → Indica que el certificado sera valido 365 dias.

-newkey rsa:4096 → Generara una nueva clase RSA de 4096 bits.

-keyout ruta → Indica donde se guardara la clave privada.

-out ruta → Indica donde se guardara el certificado.

-subj → Establece los detalles del sujeto del certificado.

## Configuracion servidor nginx

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name gemartin.42.fr;

    ssl_certificate /etc/ssl/certs/nginx.crt;
    ssl_certificate_key /etc/ssl/private/nginx.key;
    ssl_protocols TLSv1.3;

    index index.php;
    root /var/www/html;
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_read_timeout 300;
    }
}
```
server → Indica el inicio de la configuracion del servidor.

listen 443 ssl → Establece que el servidor escuchara por el puerto 443 y que se utilizara el protocolo SSL para las conexiones.

listen [::]:443 ssl → Lo mismo que lo anterior pero esto sera para las conexiones IPv6. 

server_name gemartin.42.fr → Especifica el nombre del servidor.

ssl_certificate → Indica la ubicacion del certificado SSL.

ssl_certificate_key → Indica la ubicacion de la clave privada correspondiente al certificado.

ssl_protocols TLSv1.3 → Establece que el protocolo TLS version 1.3 sera utilizado.

index index.php → Define el archivo que se utilizara como la pagina principal si el cliente no especifica una.

root /var/www/html → Establece el directorio raiz del sitio web.

location / → Define como manejar las peticiones a rutas que no coinciden con otras reglas.

location ~ → Define como manejar las peticiones terminadas en .php.

# MariaDB

## Dockerfile

```
FROM debian:10.11

RUN apt-get update && \
	apt-get install -y \
	mariadb-server

COPY conf/mariadb.conf /etc/mysql/mariadb.conf.d/50-server.cnf

RUN mkdir -p /var/run/mysqld \
	&& chown -R mysql:mysql /var/run/mysqld \
	&& chmod 777 /var/run/mysqld

EXPOSE 3306

COPY ./tools/mariadb.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/mariadb.sh

RUN mysql_install_db

ENTRYPOINT [ "/usr/local/bin/mariadb.sh" ]
```

## Conf file

<img width="531" alt="Screen Shot 2023-11-24 at 3 24 23 AM" src="https://github.com/gemartin99/Inception-Tutorial/assets/66915274/971d0f40-f276-4e1a-a14a-51904d09384e">

## Script MariaDB

<img width="805" alt="Screen Shot 2023-06-20 at 12 07 06 PM" src="https://github.com/gemartin99/inception/assets/66915274/0af6986c-f984-42c1-8a05-551b21ef3e25">

🔴 Inicia el servicio MySQL. El servidor estara lito para aceptar conexiones.

🟠 Condicion que si no existe el directorio de la base de datos entrara al if. Si no se cumple la condicion directamente pasara al paso 🟤.

🟡 Nos conectamos a MySQL mediante nuestro usuario y contraseña (utilizamos las variables de entorno) y ejecutamos la instruccion ```CREATE DATABASE $MYSQL_DATABASE;``` para crear una nueva base de datos con el nombre almacenado en la variable de entorno.

🟢 Crearemos un usuario dentro de la base de datos especificando el nombre y la contraseña (utilizamos las variables de entorno).

🌐 Le concedemos todos los permisos al usuario recien creado para que pueda acceder a la base de datos y realizar cualquier operación en las tablas.

🔵 Actualizamos los privilegios para que los cambio realizados se apliquen.

🟣 Cambiamos la contraseña por defecto del usuario root de MySQL a la que tenemos almacenada en nuestra variable de entorno.

🟤 Apagamos el servidor utilizando el usuario y contraseña de root.

⚪️ Iniciamos el servidor MySQL de nuevo. Hemos realizado este reinicio para que el servidor se inicie con la configuracion y cambios realizados en el script.

# WordPress

## Dockerfile

```
FROM debian:10.11

RUN apt-get update && apt-get -y install \
    curl \
    php7.3-fpm \
    php7.3-mysqli \
    mariadb-client

RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x wp-cli.phar \
    && mv wp-cli.phar /usr/local/bin/wp

COPY conf/www.conf /etc/php/7.3/fpm/pool.d/

RUN mkdir -p /run/php
RUN chmod 755 /run/php

COPY ./tools/wp.sh /usr/local/bin/wp.sh
RUN chmod +x /usr/local/bin/wp.sh

EXPOSE 9000

WORKDIR /var/www/html/

ENTRYPOINT [ "/usr/local/bin/wp.sh" ]
```

## Conf file

<img width="501" alt="Screen Shot 2023-11-24 at 3 25 50 AM" src="https://github.com/gemartin99/Inception-Tutorial/assets/66915274/f5ec2989-d929-4547-89d1-627d1863be7b">

## Script Wordpress

<img width="1008" alt="Screen Shot 2023-10-13 at 10 55 20 PM" src="https://github.com/gemartin99/Inception-Tutorial/assets/66915274/caa9880f-8b3f-44f2-b303-8a214349e9de">

🔴 Se verifica si el archivo no existe , de ser asi se entra al if.

🟠 Descarga el núcleo de WordPress utilizando la herramienta WP-CLI. La opción ```--allow-root``` permite ejecutar WP-CLI como el usuario root.

🟡 Creamos el archivo de configuracion ```wp-config.php``` con los detalles de la base de datos proporcionado a traves de variables de entorno.

🟢 Realizamos la instalacion de WordPress con los parametros proporcionados en las vaiables de entorno. La opcion --skip-email evita que se envie un correo electronico de notificacion.

🔵 Creamos un nuevo usuario de WordPress con el nombre, correo y contraseña proporcionados en las variables de entorno. En este caso al usuario nuevo se le asigna el rol de autor.

🟣 Descargamos, instalamos y activamos el tema twentysixteen.

🟤 Iniciamos el servicio PHP-FPM en segundo plano y lo dejamos en ejecucion constante con -F.

# Preguntas evaluacion 

## ◦ Resumen del proyecto

Cómo funcionan docker y docker compose

Diferencia entre una imagen Docker utilizada con docker compose y sin docker compose

Ventajas de Docker frente a las máquinas virtuales

Que la estructura de directorios sea la misma que el subject

## ◦ Configuración sencilla

Asegúrese de que sólo se puede acceder a NGINX por el puerto 443. Una vez hecho esto, abra la página

Asegúrese de que se utiliza un certificado SSL/TLS

Asegúrese de que el sitio web de Wordpress está correctamente instalado y configurado (no debería ver la página de instalación de WordPress). Para acceder a él, abra https:://gemartin.42.fr en su navegador, donde login es el nombre de usuario del estudiante de evaluación. No debería poder acceder al sitio a través de http://. Si algo no funciona como se esperaba el final de la evaluación ahora

## ◦ Fundamentos de docker

Empieza por comprobar los dockerfiles. Debe haber un Dockerfile por servicio. Asegúrese de que los dockerfiles no son archivos vacíos. Si no es el caso o si falta un dockerfile, la evaluación termina ahora.

Asegúrese de que el stduent evaluado ha escrito sus propios dockerfiles y construido sus propias imágenes docker. De hecho, está prohibido utilizar las ya hechas o utilizar servicios como dockerhub

Asegurate de que todos los contenedores tienen la penultima version estable debian y si el dockerfile no empieza con FROM debian:XXX la evaluacion termina

Las imágenes docker deben tener el mismo nombre que su servicio correspondiente. De lo contrario , el proceso de evaluación termina ahora.

Asegúrese de que Makefile ha configurado todos los servicios a través de docker compose. Esto significa que los contenedores deben haber sido construidos usando docker compose y que no ha ocurrido ningún crash. De lo contrario, el proceso de evaluación termina.

## ◦ Red Docker

Asegúrate de que se utiliza docker-network comprobando el archivo docker-compose.yml. A continuación, ejecute el comando 'docker network ls' para verificar que hay una red visible.

El alumno evaluado tiene que darle una explicación sencilla de docker-network. Si alguno de los puntos anteriores no es correcto, el proceso de evaluación termina ahora.

## ◦ Nginx con SSL/TLS

Asegúrese de que existe un archivo Dockerfile

Utilizando el comando 'docker compose ps', asegúrese de que el contenedor ha sido creado (utilizando la bandera '-p' está autorizado si es necesario)

Intente acceder a través de http (puerto 80) y compruebe que no puede conectarse.

Abra la URL en su navegador, donde login es el login del estudiante evaluado. La página mostrada debe ser el sitio web de wordpress configurado (no debería ver la página de instalación de WordPress)

El uso de un certificado TLSv1.3 es obligatorio y debe demostrarse. El certificado TLS no tiene que ser reconocido. Puede aparecer un aviso de certificado autofirmado. Si alguno de los puntos anteriores no está claramente explicado y es correcto,el proceso de evaluación termina ahora.

## ◦ Wordpress con php-fpm y su volumen

Asegúrese de que existe un archivo Dockerfile

Asegúrese de que no hay NGINX en el dockerfile

Utilizando el comando 'docker compose ps', asegúrese de que el contenedor ha sido creado (utilizando la bandera '-p' está autorizado si es necesario)

Asegúrese de que hay un Volumen.Para ello: Ejecute el comando 'docker volume ls' y luego 'docker volume inspect <volume name>'. Compruebe que el resultado es la salida estándar contiene la ruta '/home/login/data'

Asegúrese de que puede añadir un comentario utilizando el usuario de WordPress disponible.

Inicie sesión con la cuenta de administrador para acceder al panel de administración. El nombre de usuario de administrador no debe incluir 'admin' o 'Admin'.

Desde el panel de administración, edite una página. Compruebe en el sitio web que la página se ha actualizado. Si alguno de los puntos anteriores no es correcto, el proceso de evaluación termina ahora.

## ◦ MariadDB y su volumen

Asegúrese de que existe un archivo Dockerfile

Asegúrese de que no hay NGINX en el dockerfile

Utilizando el comando 'docker compose ps', asegúrese de que el contenedor ha sido creado (utilizando la bandera '-p' está autorizado si es necesario)

Asegúrese de que hay un Volumen.Para ello: Ejecute el comando 'docker volume ls' y luego 'docker volume inspect <volume name>'. Compruebe que el resultado es la salida estándar contiene la ruta '/home/login/data'

El estudiante evaluado debe ser capaz de explicarle cómo iniciar sesión en la base de datos. Compruebe que la base de datos no está vacía. Si alguno de los puntos anteriores no es correcto, el proceso de evaluación termina ahora.

## ◦ Persistencia!

Esta parte es bastante sencilla. Tienes que reiniciar la máquina virtual. Una vez que se haya reiniciado, ejecute docker compose de nuevo. A continuación, comprueba que todo funciona y que tanto WordPress como MariaDB están configurados. Los cambios que hiciste anteriormente en el sitio web de WordPress deberían seguir aquí. Si alguno de los puntos anteriores no es correcto, el proceso de evaluación termina ahora.
















