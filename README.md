# platzigram

Liberar un proyecto de Django a producción es una tarea bastante sencilla pero que puede confundir a muchos la primera vez que se intente (a mi me sucedió). El objetivo de esta lectura es tener una breve a introducción a la arquitectura de un proyecto de Django corriendo en un servidor de producción (un servidor de verdad) y que consecuentemente los siguientes tutoriales de configuración tengan más sentido al momento de que los leas.

Al principio del curso hablamos de un archivo llamado wsgi.py ubicado dentro del folder de las configuraciones del proyecto, conviviendo junto con el archivo urls.py y settings.py. WSGI significa Web Server Gateway Interface y es un protocolo sencillo de llamadas para que un web server (como NGINX o Apache) se comuniquen con una aplicación web o framework escritos en Python.

WSGI nos permite delegar el trabajo de aplicar reglas complejas de enrutamiento a un web server como NGINX y al mismo tiempo lograr que exista una comunicación del usuario final de nuestro proyecto de Python. Dicho esto, esta sería la ilustración de un servidor que expone múltiples servicios como e-mail a través de pop3, un app server usando SSL, otro app server redirigiendo las peticiones HTTP a HTTPS y una base de datos de PostgreSQL:

:110 --> email

:80 --> wsgi

:443 --> wsgi --> django

:5432 --> postgres

Para el caso particular del proyecto del curso, nosotros usaremos un servidor Linux corriendo Ubuntu 16.04 en el cual configuraremos una base de datos de PostgreSQL, un web server NGINX y correremos nuestro proyecto de Django usando Gunicorn. Los archivos estáticos y subidos por los usuarios serán también servidos usando NGINX ya que no es trabajo de Django realizar estas tareas. La base de datos no tiene que estar disponible para el público por lo que no hay necesidad de que NGINX la exponga.

-- nginx --

:80 / ---> wsgi --> django <--> postgres

:80 /media --> media_root

:80 /static --> static_root

Django obtiene la estructura, acceso y control de los datos de una aplicación a través de su ORM (Object Relational Mapper), esto significa que no importa qué motor de base de datos esté usando, el mismo código seguirá funcionando, configurar esto en un proyecto de Django es cuestión de segundos.

Todo se define dentro del archivo settings.py de nuestro proyecto dentro de la variable DATABASES:

DATABASE

Será el nodo padre que nos servirá para indicar que definiremos una base de datos.
Dentro, tendremos el nodo default este tendrá toda la configuración clave de la base de datos.

```python
DATABASES= {
    'default':{
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydatabase',
        'USER': 'mydatabaseuser',
        'PASSWORD': 'mypassword',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}
```

Además, Django puede trabajar con múltiples bases de datos usando una estrategia llamada routers por lo que el diccionario DATABASES puede contener múltiples llaves con diferentes bases de datos. Pero eso sí, necesita siempre existir una llave “default”.

Es un diccionario de python el cual requiere definir una base de datos por default, más de eso al final, usando la llave default que a su vez será otro diccionario con los datos de configuración:

La configuración recibirá el engine el cual puede ser:

PostgreSQL: 'django.db.backends.postgresql’
MySQL: 'django.db.backends.mysql’
SQLite: 'django.db.backends.sqlite3’
Oracle: 'Django.db.backends.oracle’
El nombre de la base de datos “NAME”.
El usuario “USER”.
La contraseña “PASSWORD”.
La ubicación o host del servidor de la base de datos “HOST”.
Y el puerto de conexión “PORT”.

Adicionalmente, se pueden configurar más detalles por base de datos, por ejemplo, configurar que todos los queries de una vista sean empaquetados en una sola transacción a la base de datos usando ATOMIC_REQUESTS=True

## Configuración inicial del servidor

Es una buena práctica que la primera acción que realicemos al conectarnos a nuestro servidor sea actualizarlo. Lo que podemos hacer con los siguientes comandos:

`sudo apt-get update`
`sudo apt-get upgrade`

Luego crearemos un nuevo usuario que tenga la capacidad de correr algunos comandos de súper usuario pero que no sea súper usuario:

`sudo useradd [username] -g sudo -m`

Le asignamos una contraseña segura al nuevo usuario:

`sudo passwd [username]`

Iniciamos sesión con el nuevo usuario:

`su - [username]`

Instalar las dependencias necesarias

Dependencias de python:

`sudo apt-get install python3-pip python3-dev`

Dependencias de PostgreSQL:

`sudo apt-get install postgresql postgresql-contrib libpq-dev`

Git:

`sudo apt-get install git`

Nginx:

`sudo apt-get install nginx`

## Configurar PostgreSQL

Iniciamos sesión con el usuario postgres:

`sudo su postgres`

Entramos al shell interactivo de postgres:

`psql`

Creamos una base de datos:

`CREATE DATABASE platzi;`

Creamos un usuario para la base de datos:

`CREATE USER freddier WITH PASSWORD ‘cvander<3’;`

Le damos permisos al usuario sobre la base de datos:

`GRANT ALL PRIVILEGES ON DATABASE platzi TO freddier;`

Salimos del shell:

`\q`

Salimos de la sesión de postgres:

`exit`

## Configurar el proyecto

Clonar el proyecto de Github:

`git clone https://github.com/pablotrinidad/platzigram.git platzi`

Instalar virtualenv:

`sudo pip3 install virtualenv`

Crear entorno virtual:

`virtualenv -p $(which python3) .venv`

Activar entorno virtual:

`source .venv/bin/activate

Instalar dependencias para Pillow:

`sudo apt-get install libjpeg-dev`

Ir al folder del proyecto:

`cd platzi`

Instalar dependencias de python del proyecto:

`pip install -r requirements/prod.txt`

Agregamos algunas variables de entorno a **~/.bashrc ** para probar localmente que todo esté funcionando:

`vi ~/.bashrc`

Las variables lucen algo similar a lo siguiente:

[shell] export PLATZI_SECRET_KEY="random_key:aasdafasf"export PLATZI_DB_NAME=“platzi”; export PLATZI_DB_USER=“freddier”; export PLATZI_DB_PASSWORD=“cvander<3” export PLATZI_DB_PORT=“5432” export PLATZI_DB_HOST="localhost"export DJANGO_SETTINGS_MODULE=“platzi.settings.prod”[/shell]

Leemos las variables:

`source ~/.bashrc`

Editamos la variable **ALLOWED_HOSTS ** del archivo settings de producción:

`vim platzi/settings/prod.py`

La variable tendrá algo como lo siguiente, donde gatos.io sea tu dominio o IP:

`ALLOWED_HOSTS = [’.gatos.io’]`

## Sanity Check

Hasta este punto el proyecto debe ser capaz de escribir a la base de datos y servirse usando el servidor de desarrollo y gunicorn. Probémoslo.

Reflejar el modelo de Django en PostgreSQL:

`./manage.py migrate`

Crear un super usuario:

`./manage.py createsuperuser`

Correr servidor de desarrollo:

`./manage.py runserver 0.0.0.0:8000`

Correr gunicorn:

`gunicorn platzi.wsgi -b 0.0.0.0:8000`

Si todo funcionó correctamente, los pasos 3 y 4 debieron mostrar tu sitio en la URL o IP en el puerto 8000.

## Configurar Nginx

Iniciar sesión como super usuario:

`sudo su -`

Ir al directorio de Nginx:

`cd /etc/nginx/`

Borrar los antiguos archivos de configuración:

`rm sites-\*/default`

Crear un nuevo archivo:

E [shell]vim sites-available/app[/shell] Con el siguiente contenido:

[shell] upstream django_app { server 127.0.0.1:8000; }server {listen 80; server_name <server_name></server_name>;access_log /var/log/nginx/app.log; error_log /var/log/nginx/app.error.log;location /static { autoindex on; alias /home/platzi/platzi/staticfiles/; } location /media { autoindex on; alias /home/platzi/platzi/media/; } location / { proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header Host $http_host; proxy_redirect off; proxy_pass http://django_app; } } [/shell]

Linkear los archivos:

[shell]ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/[/shell]

Reiniciar Nginx:

[shell]service nginx restart[/shell]

## Configurar Gunicorn

Regresamos a la sesión de Platzi:

[shell]exit[/shell]

Creamos folder para los scripts y los logs:

[shell]mkdir deploy logs[/shell]

Creamos un script dentro de deploy:

[shell]vim deploy/gunicorn_start[/shell] Con un contenido similar al siguiente:

[shell] #!/bin/bashNAME=“platzi” VIRTUALENV="/home/platzi/.venv/" DJANGODIR="/home/platzi/platzi/" USER=platzi GROUP=sudo NUM_WORKERS=3 DJANGO_WSGI_MODULE=platzi.wsgiecho "Starting $NAME as whoami"cd $VIRTUALENV source bin/activate cd $DJANGODIRexport PLATZI_SECRET_KEY=“random_key:aasdafasf” export PLATZI_DB_NAME=“platzi” export PLATZI_DB_USER=“freddier” export PLATZI_DB_PASSWORD=“cvander<3” export PLATZI_DB_PORT=“5432” export PLATZI_DB_HOST=“localhost” export DJANGO_SETTINGS_MODULE=“platzi.settings.prod” export PYTHONPATH=$DJANGODIR:$PYTHONPATH exec gunicorn ${DJANGO_WSGI_MODULE} \ --workers $NUM_WORKERS \ --user=$USER --group=$GROUP \ --log-level=debug \ --bind=127.0.0.1:8000 [/shell]

Hacer el script ejecutable:

[shell]chmod +x deploy/gunicorn_start[/shell]

Probar el script:

[shell]deploy/gunicorn_start[/shell]

Mientras el script esté corriendo, el proyecto estará viviendo en la IP en el puerto 80.

## Crear un servicio

Iniciar sesión como super usuario:

[shell]sudo su -[/shell]

Ir al directorio de los servicios:

[shell]cd /etc/init[/shell]

Crear el servicio:

[shell]vim platzi.conf[/shell] Con el siguiente contenido:

[shell] # platzi# description “Platzi Linux Service” # authon "Pablo Trinidad"start on startupscript exec /home/platzi/deploy/gunicorn_start end script[/shell]

Iniciar servicio:

[shell]service platzi start[/shell]

Por último tenemos que regresar al folder de la carpeta que contiene el proyecto y ejecutar:

[shell]./manage.py collectstatic[/shell]

## El servidor

Para la demostración de la clase se usa una máquina t2.nanoque Amazon Web Services provee con Ubuntu Server. Toda la configuración del proyecto vive en el mismo servidor. Es decir, tanto la base de datos como los archivos estáticos y el código fuente son manejados por una sola máquina. Es importante mencionar que en casos donde nuestro proyecto es más grande y requiere de una mejor arquitectura, es recomendable separar cada uno de estos de manera que la base de datos tenga su propio servidor, que exista un balanceo de carga hacia las instancias que manejan el código y que la media y los estáticos sean servidos desde una CDN. El caso de instalación que veremos en este post es un método que se puede usar en cualquier servidor Linux con Ubuntu Server; por lo que no es una configuración que únicamente se pueda llevar a cabo usando AWS. Cualquier proveedor que te dé acceso a una máquina Linux es útil.

## Crear el servidor

Una vez dentro de la consola de administración de Amazon Web Services sigue estos pasos:

- Accede a la sección de Amazon EC2
- Da clic en el botón Launch Instance
- Selecciona Ubuntu 16.04 como el Sistema Operativo deseado
- Elige el tipo de instancia que más se adecúe a tus necesidades (t2.micro es parte de la capa gratuita)
- En el paso 3, deja todas las configuraciones tal y como están
- Selecciona la cantidad de GB de almacenamiento que quieras tener en tu instancia
- Asigna un nombre descriptivo a la instancia
- Crea un nuevo grupo de seguridad con el puerto 22, 80 y 8000 abiertos desde cualquier IP por el protocolo TCP
- Selecciona Launch
- Crea nuevas llaves SSH y descargarlas al ordenador
- Conectarse al servidor
- Para conectarnos al servidor usaremos la llave que acabamos de descargar. Es muy importante nunca perder esta llave ya que si la perdemos no tendremos otra forma de acceder al servidor.

Poner la llave en modo lectura:

[shell]chmod 0400 Platzi.pem[/shell]

Conectarse al servidor usando la IP pública que AWS nos asigna:

[shell]sudo ssh -i Platzi.pem ubuntu@IP[/shell]
