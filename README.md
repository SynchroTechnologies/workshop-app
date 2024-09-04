# Workshop demo APP

Esta es una aplicación simple, de 3 capas, que utilizaremos para recorrer el ciclo de vida hasta la puesta en producción.

Abarcaremos:

- Deploy manual de la aplicación, de forma monolítica, en una VM de prueba.
- Proceso de convertir cada capa en un container.
- Proceso de Integración contínua para el building de los contenedores.
- Deploy manual de los contenedores usando un runtime simple (Podman o Docker).
- Creación de manifiestos para deploy de la aplicación en OpenShift.
- Deploy manual de los contenedores con sus manifiestos en un ambiente OpenShift de Test
- Proceso de Deploy contínuo, usando Helms y ArgoCD.
- Deploy automático de la aplicación en un ambiente de Producción.

## Arquitectura de la aplicación

![Arquitectura Workshop APP](./workshop.app.drawio.png)

La aplicación cuenta con un Single Page Application (SPA) contenida en un único archivo .html, que encapsula el contenido, los estilos y los scripts Javascript para la ejecución.
La aplicación se descarga, y corre íntegramente en el browser.

Esta SPA se comunica con una API en el backend para enviarle información que la API deberá persistir en una base PostgreSQL.

La SPA tambien puede pedirle a la API los últimos 5 registros grabados, y los mostrará en una ventana Modal.

## Instalación de la Aplicación de forma monolítica

Se pueden desplegar todas las piezas de la aplicación en una misma máquina, ya sea física o virtual. Para este ejemplo, cada componente usa tecnología que no entra en conflicto con los otros componentes:

- La base de datos PostgreSQL es una instalación de binarios que se encuentra disponibles en cualquier distribución moderna de Linux. Ya sea Debian o sus derivados, como Ubuntu. O RedHat y sus derivados como Oracle Linux, Almalinux, etc.

- La API de Backend está desarrollada en Python 3, con el framework Flask, también disponible en cualquier distribución moderna.

- El frontend se sirve desde cualquier web server que tengamos a mano: Apache, Nginx, NodeJS, etc. En nuestro caso, por simpleza, vamos a usar Nginx como base para servir los archivos estáticos.

---

### Base de datos PostgreSQL

Se descarga del repositorio que ofrece cada distribución, junto al manejador de paquetes propio, ```apt``` en el caso de Debian / Ubuntu y sus derivados, ```yum``` o ```dnf``` en el caso de RedHat / Almalinux / Oracle Linux y sus derivados.

Para Ubuntu 22.04 y superior:

```bash
> sudo apt install postgresql
```

Para RedHat 9:

```bash
> sudo dnf install postgresql-server
```

Cada distribución guarda los archivos de configuración en locaciones diferentes, pero los archivos importantes en cualquier caso son:

```bash
postgresql.conf

pg_hba.conf
```

El primero administra los parámetros del motor, el segundo administra la seguridad de acceso de las conexiones al motor.

En principio no es necesario modificar nada, la configuración por default va a ser suficiente para nuestra aplicación.

En el caso de las distribuciones basadas en RedHat, es necesario inicializar el motor, con el comando ```postgresql-setup --initdb```.

En el caso de las distribuciones basadas en Debian, el motor ya viene inicializado, y los archivos mencionados están en ```/etc/postgresql/<version>/main```

Para iniciar el motor, desbloqueamos el servicio administrado por ```systemd``` y le damos arranque:

```bash
> sudo systemctl enable postgresql.service

> sudo systemctl start postgresql.service

> sudo systemctl status postgresql.service
```

Una vez iniciado el motor, debemos crear un usuario y una base de datos para nuestra aplicación. Nos conectamos localmente con el usuario ```postgres``` que no requiere password cuando la conexión se realiza contra ```localhost (127.0.0.1)```.

```bash
psql -U postgres
```

```sql
postgres=# CREATE ROLE synchro WITH SUPERUSER LOGIN PASSWORD 'synchro-2024';

postgres=# CREATE DATABASE app WITH OWNER synchro ENCODING 'UTF8';
```

Podemos verificar si el usuario y la base se crearon correctamente:

```bash
> psql -h localhost -U synchro -d app -W
```

Una vez verificado el nuevo usuario y la nueva base, nuestro motor está listo para ser utilizado. La API Backend se encargará de crear la tabla que necesita si es que no existe.

---

### API Backend

El código Python para la aplicación de backend lo encontramos en un repositorio GitHub público:

https://github.com/synchrotechnologies/workshop-app-backend.git

Debemos clonar este repo a nuestro servidor para poder usar la aplicación.

```bash
> git clone https://github.com/synchrotechnologies/workshop-app-backend.git
```

En el repositorio hay un archivo de dependencias para instalar las librearías que son necesarias. Sin embargo, hacen falta dos módulos utilitarios de Python que a veces no están instalados en la distribución: ```pip``` y ```venv```.

Con el manejador de paquetes de nuestra distro buscamos e instalamos los siguientes paquetes:  ```python3-pip``` y ```python3-venv```.

Antes de inicar la aplicación, vamos a verificar en el archivo ```app.py``` que esté bien el string de conexión a la base de datos.

```python
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://synchro:synchro-2024@localhost/app'
```

Luego, parados en el directorio que se creó en la clonación del repo, podemos iniciar nuestra aplicación:

```bash
> python3 -m venv venv

> source venv/bin/activate

> pip3 install -r requirements.txt

> nohup python3 app.py &
```

Si todo marcha correctamente, la aplicación quedará corriendo en backgrund y se habrá generado un archivo ```nohup.out``` que tiene los logs de la aplicación.

Podemos salir del entorno virtual de Python con el comando ```deactivate```.

---

### APP Frontend

La página HTML única y estática, junto a las imágenes necesarias, están en un repositorio GitHub público:

https://github.com/synchrotechnologies/workshop-app-frontend.git

Debemos lonar este repo a nuestro servidor para poder iniciar nuestro servidor web.

```bash
> git clone https://github.com/synchrotechnologies/workshop-app-frontend.git
```

En esta oportunidad usaremos ```Nginx``` como servidor web. Los instalamos del repositorio de nuestra distribución, como hicimos con la base Postgresql.

Para Ubuntu 22.04 y superior:

```bash
> sudo apt install nginx
```

Para RedHat 9:

```bash
> sudo dnf install nginx
```

Cada distribución organiza los archivos de configuración de Nginx de forma diferente dentro del directorio ```/etc/nginx```.

Debemos encontrar la definición del default site, y modificar la definción del directorio ```root```.

Vamos a crear un directorio donde Nginx pueda encontrar los archivos que debe servir.

```bash
> sudo mkdir -p /opt/www/html
```

Y copiamos los archivos ```index.html``` y ```nube2.jpeg``` en ese directorio. Nos aseguramos que todos los usuarios y grupos tengan permisos de lectura.

Luego localizamos y modificamos la definición del directorio ```root``` en el archivo de configuración de Nginx.

```conf
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /opt/www/html;
```

Finalmente habilitamos e iniciamos el servicio administrado por ```systemd```:

```bash
> sudo systemctl enable nginx.service

> sudo systemctl start nginx.service

> sudo systemctl satus nginx.service
```

Podemos verificar si Nginx está funcionando correctamente:

```bash
> curl http://localhost
```

Si todo marcha bien, ya tenemos los tres componentes de nuestra aplicación activos.

Desde un navegador podemos llamar a la aplicación:

```
http://<url del servidor>/
```

---
