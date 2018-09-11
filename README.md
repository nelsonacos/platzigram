# platzigram

Proyecto desarrollado en el curso de django de [platzi](https://platzi.com/)

**Autor:** Pablo Trinidad

**Url del curso:** https://platzi.com/cursos/django/

## Desplegar en AWS

Conectarse al servidor usando la IP pública que AWS nos asigna:

`sudo ssh -i [clave].pem ubuntu@IP`

## Configuración inicial del servidor

Es una buena práctica que la primera acción que realicemos al conectarnos a nuestro servidor sea actualizarlo. Lo que podemos hacer con los siguientes comandos:

`sudo apt-get update`

`sudo apt-get upgrade`

Para mayor seguridad crearemos un nuevo usuario que tenga la capacidad de correr algunos comandos de súper usuario pero que no sea súper usuario:

`sudo useradd [nelson] -g sudo -m`

En mi caso le di el nombre **nelson** tu puedes llamarlo como desees, le asignamos una contraseña segura al nuevo usuario:

`sudo passwd [mipass]`

Iniciamos sesión con el nuevo usuario:

`su - [nelson]`

Ahora instalaremos las dependencias que deben vivir de manera global en nuestro servidor de produccion:

Dependencias de python:

`sudo apt-get install python3-pip python3-dev`

Dependencias de PostgreSQL:

`sudo apt-get install postgresql postgresql-contrib libpq-dev`

Git:

`sudo apt-get install git`

Nginx:

`sudo apt-get install nginx`

Supervisor:

`sudo apt-get install supervisor`

Vim para modificar archivos:

`sudo apt-get install vim`

## El entorno virtual

Ya tenemos lo básico instalado para iniciar nuestro despliegue en el servidor, adicional a esto necesitamos instalar un par de cosas como es un entorno virtual y gunicorn. Gunicorn es un servidor de aplicaciones que nos permite conectar por medio de sockets nuestra aplicación Django con Nginx para que pueda ser utilizada. Para nuestro entrono virtual vamos a instalar virtualenv:

`sudo pip3 install virtualenv`

Aqui voy a hacer una pausa para solucionar un problema que me encontrando en varias oportunidades al instalar virtualenv si te produce este error:

Para completar los valores perdidos, edite ~ / .bashrc:

`$ vim ~/.bashrc`

Agregue las siguientes líneas después del comando anterior (suponga que desea que en_US.UTF-8 sea su idioma):

```bash
export LANGUAGE="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
```

Después de guardar el archivo, haga lo siguiente:

`$ source ~/.bashrc`

Ahora ya no estarás enfrentando el mismo problema y podemos Crear un entorno virtual usando Python 3 como versión de entorno virtual:

```bash
virtualenv -p python3 .venv
```

le producira una salida como esta:

```bash
New python executable in .venv/bin/python
Installing distribute..............done.
Installing pip.....................done.
```

Aquí .venv es el nombre del entorno virtual. Cámbielo si para usted es necesario. Para activar este entorno, ejecute:

```bash
source .venv/bin/activate
```

notara el siguiente cambio en la terminal:

```bash
(.venv) $
```

vamos traer el codigo nuestro proyecto a nuestro servidor Usando Git:

`git clone [https://github.com/nelsonacos/platzigram.git]`

Instalamos las dependencias que viviran en nuestro entorno virtual:

`pip install -r requeriments.txt`

deberia tener todas las dependencias instaladas con el comando anterior, sin embargo vale la pena tomar en cuenta que en algunas ocasiones puedes haber usado una base de datos en desarrollo como sqlite y en produccion probrablemente quiera usar postgres, debe asegurarse de instalar psycopg2 en su entorno virtual:

`pip install psycopg2`

Otra cosa que debe considerar para produccion que probablemente no uso en desarrollo es gunicorn:

`pip install gunicorn`

## Configurar PostgreSQL

Crear un usuario sin permisos de superusuario sin capacidad de crear bases de datos. Sólo debe asignarle una base de datos y otorgarle todos los permisos necesarios sobre esa base de datos, para ello debe hacerlo con el usuario que postgres crea por defecto:

`sudo su - postgres`

luego de iniciar session en la terminal con el usuario postgres ejecute el siguiente comando para crear un usuario de postgres de forma interactiva.

```bash
createuser --interactive -P
```

le producira una salida como esta:

```bash
Enter name of role to add: db_user
Enter password for new role:
Enter it again:
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) n
Shall the new role be allowed to create more new roles? (y/n) n
postgres@nelson:~$
```

como puede ver le pide primero el nombre de usuario, luego la contraseña, confirmar la contraseña, luego pregunta si desea que este usuario sea superusuario, debe contestar **n** por medidas de seguridad asi como tambien en todas las siguientes, la otra pregunta es si queremos que este usuario pueda crear bases de datos, y la ultima pregunta es si queremos que este usuario pueda crear nuevos usuarios. Recuerde por medidas de seguridad debe contestar **n**

A continuación, modifique algunos de los parámetros de conexión para el usuario que acaba de crear. Esto acelerará las operaciones de base de datos de modo que los valores correctos no tengan que ser consultados y configurados cada vez que se establezca una conexión.

Debe establecer la codificación por defecto a UTF-8, que es la que Django espera. También debe que establecer el régimen de aislamiento de las transacciones de “read committed”, el cual bloquea la lectura de transacciones no confirmadas. Por último,debera establecer la zona horaria.

Ejecute postgres con el siguiente comando:

`psql`

De forma predeterminada, se establecerán los proyectos de Django para usar UTC. Éstas son todas las recomendaciones del propio proyecto de Django. Ejecute los siguientes comandos:

`ALTER ROLE [db_user] SET client_encoding TO ‘utf8’;`

`ALTER ROLE [db_user] SET default_transaction_isolation TO ‘read committed’;`

`ALTER ROLE [db_user] SET timezone TO ‘UTC’;`

Para salir de postgres ejecute:

`\q`

Debe otorgar a este usuario todos los permisos sobre la base de datos que usara en la aplicacion:

`createdb --owner [db_user] [platzigram_db]`

En el ejemplo anterior, el nombre de usuario de la base de datos es db_user y la base de datos platzigram_db. Recuerde proporcionar un nombre apropiado a la base de datos, según su aplicación Django.

Cierre la sesión del usuario postgres.

`logout`

## Configurar la base de datos en django

`vim platzigram/platzigram/settings.py`

Dentro, tendra el nodo default este tendrá toda la configuración clave de la base de datos.

```python
DATABASES= {
    'default':{
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'platzigram_db',
        'USER': 'db_user',
        'PASSWORD': 'mypassword',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}
```

Django puede trabajar con múltiples bases de datos usando una estrategia llamada routers por lo que el diccionario DATABASES puede contener múltiples llaves con diferentes bases de datos. Pero necesita siempre existir una llave “default”.

Para esta guia se usa postgres pero usted puede asignar otras opciones. La configuración que recibirá **ENGINE**s puede ser:

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

otra variable que debemos editar del archivo settings de producción es **ALLOWED_HOSTS**. La variable tendrá algo como lo siguiente, donde www.nelsonacosta.cl sea tu dominio o IP:

`ALLOWED_HOSTS = [’www.nelsonacosta.cl’]`

cambiamos la configuración de DEPURACIÓN a False :

`DEBUG = False`

## Probar nuestro servidor

Hasta este punto el proyecto debe ser capaz de escribir a la base de datos y servirse usando el servidor de desarrollo y gunicorn. Probémoslo.

Activar entorno virtual:

`source .venv/bin/activate`

nos movemos hasta la carpeta del proyecto django:

`cd platzigram`

Ahora para asegurar que nuestros archivos estaticos se muestren correctamente ejecutamos el siguiente comando:

`./manage.py collectstatic`

Refleje el modelo de Django en PostgreSQL:

`./manage.py migrate`

Cree un super usuario para la app:

`./manage.py createsuperuser`

Correr servidor de desarrollo:

`./manage.py runserver 0.0.0.0:8000`

Salga del servidor con CONTROL-C.

Correr gunicorn:

`(.venv) $ gunicorn platzigram.wsgi:application --bind 0.0.0.0:8001`

Ahora, puede acceder a gunicorn desde la IP pública de su servidor con el puerto 8001. Para hacer que Gunicorn sea más útil para la aplicación django, debe configurarlo.

Salga del servidor con CONTROL-C.

## Configurar Gunicorn

Creamos un script bash llamado gunicorn_start.bash. Puede cambiar el nombre del archivo según su elección.

Nos movemos a la ruta del usuario:

`cd ..`

Ahora es el momento de configurar gunicorn para que pueda trabajar en conjunto con supervisor. Si el sistema se reinicia o la aplicación se cierra inesperadamente, supervisor se encargará de su reinicio. creamos el script:

`vim gunicorn_start.bash`

ahora agregue las siguientes configuraciones en el archivo

```bash
#!/bin/bash

NAME="platzigram"                                   # Name of the application
DJANGODIR=/home/nelson/platzigram                   # Django project directory
SOCKFILE=/home/nelson/.venv/run/gunicorn.sock       # we will communicte using this unix socket
USER=nelson                                         # the user to run as
GROUP=sudo                                          # the group to run as
NUM_WORKERS=3                                       # how many worker processes should Gunicorn spawn
DJANGO_SETTINGS_MODULE=platzigram.settings      # which settings file should Django use
DJANGO_WSGI_MODULE=platzigram.wsgi              # WSGI module name
echo "Starting $NAME as `whoami`"

# Activate the virtual environment

cd $DJANGODIR
source /home/nelson/.venv/bin/activate
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH

# Create the run directory if it doesn't exist

RUNDIR=$(dirname $SOCKFILE)
test -d $RUNDIR || mkdir -p $RUNDIR

# Start your Django Unicorn
# Programs meant to be run under supervisor should not daemonize themselves (do not use --daemon)

exec gunicorn ${DJANGO_WSGI_MODULE}:application \
  --name $NAME \
  --workers $NUM_WORKERS \
  --user=$USER --group=$GROUP \
  --bind=unix:$SOCKFILE \
  --log-level=debug \
  --log-file=-
```

Todas las variables están bien explicadas (comentadas). Cambie el valor de las variables de acuerdo con su configuración. La variable NAME , definirá cómo se identificará su aplicación en programas como ps , top , etc. La forma recomendada de definir el numero de workers es igual a 2 \* CPUs + 1. Por ejemplo, para una sola máquina de CPU debe configurarse con 3 workers.

Este script nos permite levantar nuestra aplicación django sin usar `./manage runserver`

Asigne permisos de ejecucion al script:

`sudo chmod u+x gunicorn_start.bash`

Pruebe este script ejecutándo:

```bash
./gunicorn_start.bash
```

Le producira un salida como esta:

```bash
Starting hello_app as hello
2013-06-09 14:21:45 [10724] [INFO] Starting gunicorn 18.0
2013-06-09 14:21:45 [10724] [DEBUG] Arbiter booted
2013-06-09 14:21:45 [10724] [INFO] Listening at: unix:/webapps/hello_django/run/gunicorn.sock (10724)
2013-06-09 14:21:45 [10724] [INFO] Using worker: sync
2013-06-09 14:21:45 [10735] [INFO] Booting worker with pid: 10735
2013-06-09 14:21:45 [10736] [INFO] Booting worker with pid: 10736
2013-06-09 14:21:45 [10737] [INFO] Booting worker with pid: 10737
```

Ahora es el momento de configurar supervisor para que pueda supervisar nuestra aplicación. Si el sistema se reinicia o la aplicación se cierra inesperadamente, el supervisor se encargará de su reinicio.

salga del proceso con **CONTROL-C**

## Configurando Supervisor

Para supervisar cualquier programa a través de supervisor, se debe crear un archivo de configuración para ese programa dentro del directorio /etc/supervisor/conf.d/ Para nuestra aplicación Django que es platzigram, crearemos platzigram.conf

`sudo vim /etc/supervisor/conf.d/platzigram.conf`

Ahora, escriba el siguiente contenido en el archivo abierto.

```bash
[program:platzigram]
command = /home/nelson/gunicorn_start.bash                   ; Command to start app
user = nelson                                                ; User to run as
stdout_logfile = /home/nelson/logs/gunicorn_supervisor.log   ; Where to write log messages
redirect_stderr = true                                       ; Save stderr in the same log
environment=LANG=en_US.UTF-8,LC_ALL=en_US.UTF-8              ; Set UTF-8 as default encoding
```

Cambie el valor de configuración anterior según su configuración. Como mencionamos en el archivo anterior que los registros se almacenarán en /home/nelson/logs/gunicorn_supervisor.log , necesitamos crear este directorio y el archivo.

`mkdir -p /home/nelson/logs/`

`touch /home/nelson/logs/gunicorn_supervisor.log`

Una vez hecho esto, le pediremos al supervisor que vuelva a leer los archivos de configuración y los actualice para que nuestro nuevo archivo de configuración obtenga add.

### Para Ubuntu 14.04:

`sudo supervisorctl reread`

Le producira una salida como esta:

`platzigram: available`

Ejecute:

`sudo supervisorctl update`

Le producira una salida como esta:

`platzigram: added process group`

Como puede ver, el archivo de configuración de platzigram se agrega al grupo de procesos de supervisor. Ahora, inicia nuestra aplicación a través de él. Para esto:

`sudo supervisorctl start platzigram`

Le producira una salida como esta:

`platzigram: started`

### Para Ubuntu 16.04:

`sudo systemctl restart supervisor`

`sudo systemctl enable supervisor`

Para verificar el estado:

```bash
$ sudo supervisorctl status platzigram
platzigram                       RUNNING   pid 23267, uptime 0:00:26
```

Para detener:

```bash
$ sudo supervisorctl stop platzigram
platzigram: stopped
```

Para reiniciar:

```bash
$ sudo supervisorctl restart platzigram
platzigram: stopped
platzigram: started
```

Ahora, la aplicación se reiniciará automáticamente después de que el sistema se inicie o la aplicación se cuelgue.

Por último debe configurar Nginx. Actuará como servidor para la aplicación.

## Configurar Nginx

Configurar el servidor web para que se conecte con el socket que tenemos corriendo gracias a gunicorn y muestre nuestro sitio web.

Nginx es realmente sencillo de configurar, nginx cuenta con dos carpetas una donde se almacenan las configuraciones de los sitios disponibles y otra donde se almacenan los sitios activos, vamos a crear nuestra configuración en la carpeta /etc/nginx/sites-available creamos nuestro archivo igual a como creamos el archivo de configuración para supervisor

`$ sudo vim /etc/nginx/sites-available/platzigram.conf`

Ahora, coloca el siguiente contenido en el archivo abierto.

```bash
upstream platzigram_server {
  # fail_timeout=0 means we always retry an upstream even if it failed
  # to return a good HTTP response (in case the Unicorn master nukes a
  # single worker for timing out).
  server unix:/home/nelson/.venv/run/gunicorn.sock fail_timeout=0;
}

server {

    listen   80;
    server_name <your domain name>;

    client_max_body_size 4G;
    access_log /home/nelson/logs/nginx-access.log;
    error_log /home/nelson/logs/nginx-error.log;

    location /static/ {
        alias   /home/nelson/platzigram/static/;
    }

    location /media/ {
        alias   /home/nelson/platzigram/media/;
    }

    location / {

        # an HTTP header important enough to have its own Wikipedia entry:
        #   http://en.wikipedia.org/wiki/X-Forwarded-For
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;


        # enable this if and only if you use HTTPS, this helps Rack
        # set the proper protocol for doing redirects:
        # proxy_set_header X-Forwarded-Proto https;

        # pass the Host: header from the client right along so redirects
        # can be set properly within the Rack application
        proxy_set_header Host $http_host;

        # we don't want nginx trying to do something clever with
        # redirects, we set the Host: header above already.
        proxy_redirect off;

        # set "proxy_buffering off" *only* for Rainbows! when doing
        # Comet/long-poll stuff.  It's also safe to set if you're
        # using only serving fast clients with Unicorn + nginx.
        # Otherwise you _want_ nginx to buffer responses to slow
        # clients, really.
        # proxy_buffering off;

        # Try to serve static files from nginx, no point in making an
        # *application* server like Unicorn/Rainbows! serve static files.
        if (!-f $request_filename) {
            proxy_pass http://platzigram_server;
            break;
        }
    }

    # Error pages
    error_page 500 502 503 504 /500.html;
    location = /500.html {
        root /home/nelson/platzigram/static/;
    }
}
```

Una vez hecho esto debe dar de alta la configuración del sitio, para esto ejecute:

`sudo ln -s /etc/nginx/sites-available/platzigram.conf /etc/nginx/sites-enabled/platzigram.conf`

Prueba tu configuración de Nginx para buscar errores de sintaxis con el comando:

`sudo nginx -t`

Si todo va bien solo falta reiniciar nginx pero antes debe eliminar la configuracion anterior. Para ello nos movemos hasta:

`cd /etc/nginx/sites-available/`

veamos que hay:

`ls`

Eliminamos la configuracion por defecto:

`sudo rm default`

y finalente vamos a reiniciar nginx:

`sudo service nginx restart`

Enhorabuena, tu aplicación django lista para producción está configurada. visita desde tu navegador la ip de tu servidor o el wwww.nelsonacosta.cl

## Seguridad en nuestra app

¡Bien hecho! ¡Ya deberías tener una aplicación Django desplegada! Ahora es el momento de asegurar la aplicación para asegurarse de que es muy difícil hackearla. Para hacer eso, utilizaremos el ufwfirewall de Linux incorporado.

ufw funciona configurando reglas. Las reglas le dicen al firewall qué tipo de tráfico debe aceptar o rechazar. En este punto, hay dos tipos de tráfico que queremos aceptar, o en otras palabras, dos puertos que queremos abrir, vale recordar que este paso no es necesario si estas utilizando AWS ya que permite crear estas reglas de seguridad desde la interface grafica pero como esta guia sirve para cualquier maquina con linux le muestro como hacerlo desde la linea de comandos:

puerto 80 para escuchar el tráfico entrante a través de navegadores

puerto 22 para poder conectarse al servidor a través de SSH.

Abra el puerto escribiendo:

`$ ufw allow 80`

`$ ufw allow 22`

luego habilite ufw escribiendo:

`$ ufw enable`

**Consejo:** antes de cerrar la terminal, asegúrese de que puede conectarse a través de SSH desde otra terminal para que no esté bloqueado fuera de su droplet debido a las malas configuraciones del firewall.

## ¿Qué hacer después?

Esta publicación es la mejor guía para implementar una aplicación de Django en un único servidor. En caso de que esté desarrollando una aplicación que debería servir para grandes cantidades de tráfico, le sugiero que busque en una arquitectura de servidor altamente escalable.
