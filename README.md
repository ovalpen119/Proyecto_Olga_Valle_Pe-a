# Proyecto Olga Valle Peña

## Implantación de una plataforma web con WordPress en AWS

# 1. Introducción

El objetivo de este proyecto es implantar una plataforma web basada en
WordPress sobre Amazon Web Services (AWS), automatizando la
configuración de los servidores mediante Ansible.

La arquitectura separa las funciones principales en varios servidores:
los servidores frontend ejecutan Apache, PHP y WordPress; el backend
aloja MySQL; el servidor NFS proporciona almacenamiento compartido; y un
servidor Nginx actúa como balanceador de carga y punto de entrada para
las peticiones web. Se contempla también la configuración de HTTPS
mediante Let's Encrypt.

La documentación describe los playbooks, las plantillas y los archivos
auxiliares del proyecto. Las capturas deben añadirse después de realizar
las comprobaciones reales.

## 2. Requisitos previos

-   Cuenta de AWS con acceso a EC2.
-   Instancias Ubuntu para los servidores de la arquitectura.
-   Clave privada SSH para acceder a las instancias.
-   Ansible instalado en la máquina de control.
-   Git instalado.
-   Direcciones IP públicas y privadas de las instancias.
-   Un dominio que apunte a la dirección pública del balanceador para
    las pruebas con HTTPS.
-   Grupos de seguridad de AWS configurados para permitir únicamente las
    comunicaciones necesarias entre los servidores.

## 3. Estructura del repositorio
```
Proyecto_Olga_Valle/
├── README.md
├── main.yml
├── .gitignore
├── exports
├── htaccess/
│   └── .htaccess
├── templates/
│   ├── 000-default.conf
│   └── load_balancer.conf
├── inventory/
│   └── inventory
├── vars/
│   └── variables.yml
└── playbooks/
    ├── install_lamp_frontend.yml
    ├── install_lamp_backend.yml
    ├── deploy_wordpress_backend.yml
    ├── deploy_wordpress_frontend.yml
    ├── setup_nfs_server.yml
    ├── setup_nfs_client.yml
    ├── setup_load_balancer.yml
    └── setup_letsencrypt_https.yml
```

## 4. Preparación de Ubuntu e instalación de Git

### 4.1. Actualizar los paquetes:

```bash
sudo apt update
sudo apt upgrade -y
```
Estos comandos actualizan la información de los repositorios e instalan
las actualizaciones disponibles.

### 4.2. Instalar Git

```bash
sudo apt install git -y
```

### 4.3. Configurar el usuario de Git


```bash
git config --global user.name "TU_NOMBRE"
git config --global user.email "TU_CORREO"
```

### 4.4. Clonar el repositorio

```bash
git clone URL_DEL_REPOSITORIO
```

## 5. Inventario de Ansible

El archivo `inventory/inventory` contiene los grupos de servidores y sus direcciones.

Ejemplo de estructura:


```ini
[frontend]
IP_PUBLICA_FRONTEND_1
IP_PUBLICA_FRONTEND_2

[frontend1]
IP_PUBLICA_FRONTEND_1

[backend]
IP_PUBLICA_BACKEND

[balanceador]
IP_PUBLICA_BALANCEADOR

[nfs]
IP_PUBLICA_NFS

[all:vars]

ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/ruta_clave_privada
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'       
```

Las direcciones IP son ejemplos y deben sustituirse por las direcciones reales.La ruta de `ansible_ssh_private_key_file` debe apuntar a la
clave privada disponible en la máquina de control.


-   `frontend`: incluye los dos servidores web.
-   `frontend1`: identifica individualmente el primer frontend, al que
    apunta el playbook de despliegue de WordPress.
-   `backend`: identifica el servidor de MySQL.
-   `balanceador`: identifica el servidor Nginx.
-   `nfs`: identifica el servidor que comparte los archivos.




## 6. Variables

El archivo `vars/variables.yml` centraliza los valores utilizados por
los playbooks, como las direcciones IP, las credenciales de la base de
datos, el dominio y el directorio de WordPress.

Se incluyen variables para:

- Direcciones IP públicas y privadas.
- Base de datos: nombre, usuario y contraseña.
- Servidor: dominio y correo electrónico.
- WordPress: título, administrador, contraseña, correo electrónico y URL de acceso.
- NFS: dirección IP del servidor y de los clientes.
- Balanceador: direcciones IP de los servidores web.

Las contraseñas y las claves privadas no deben publicarse en el repositorio.

Ejemplo basado en la estructura compartida:


``` yaml
ip:
  frontend_private_ip: ""
  backend_private_ip: ""

db:
  name: 
  user: 
  password: 

certbot:
  email: otrousuario@gmail.es
  domain: dominio

wordpress:
  directory: /var/www/html
  domain: "https://dominio"
  title: "Mi Sitio WordPress"
  admin_user: ""
  admin_password: ""
  admin_email: "admin@example.com"
  hide_login_url: ""
  locale: es_ES

nfsserver:
  server_ip: ""

nfscliente:
  client_ip: ""

balanceador:
  IP_FRONTEND_1: ""
  IP_FRONTEND_2: ""
```

Las IP vacías son marcadores de posición: deben completarse antes de
ejecutar los playbooks. Los nombres de las variables deben escribirse
exactamente igual en todos los archivos, ya que Ansible distingue entre
mayúsculas y minúsculas.

## 7. Archivo .gitignore

Este archivo evita incorporar accidentalmente archivos sensibles al repositorio.

```gitignore
*.pem
*.key
.aws/
.env
.env.*
*.log
```

Estas reglas evitan que Git añada archivos que coincidan con esos
patrones cuando todavía no están siendo rastreados. No excluyen
automáticamente `vars/variables.yml`.

Si un archivo ya se ha añadido al repositorio, incluirlo posteriormente
en `.gitignore` no elimina su historial ni las copias que ya se hayan
subido..

## 8.  Instalación y configuración del frontend

### Archivo `playbooks/install_lamp_frontend.yml`

Este playbook prepara los servidores frontend mediante la instalación de
Apache y PHP, y copia la configuración del sitio web.

### 8.1. Selección de los servidores

``` yaml
- name: Playbook para instalar la pila LAMP en el FrontEnd
  hosts: frontend
  become: yes
```

-   `name`: describe el propósito del playbook.
-   `hosts: frontend`: ejecuta las tareas en los servidores del grupo
    `frontend`.
-   `become: yes`: permite realizar las tareas con privilegios de
    administrador.


### 8.2. Instalación de Apache

``` yaml
- name: Instalar el servidor web Apache
  apt:
    name: apache2
    state: present
```

Instala Apache, que recibirá las peticiones web y servirá el contenido
de WordPress.
`state: present` indica que el paquete debe estar
instalado.

### 8.3. Copia de la configuración de Apache

``` yaml
- name: Copiar el archivo de configuración de Apache
  copy:
    src: ../templates/000-default.conf
    dest: /etc/apache2/sites-available/
    mode: '0644'
```

Copia `000-default.conf` desde `templates` al directorio de sitios
disponibles de Apache. El archivo define la configuración del sitio web.

### 8.4. Habilitación del módulo `rewrite`

``` yaml
- name: Habilitar el módulo rewrite de Apache
  apache2_module:
    name: rewrite
    state: present
```

El módulo `rewrite` permite aplicar reglas de reescritura de URL,
necesarias para determinados enlaces permanentes de WordPress. Si la
tarea aparece duplicada en el playbook, basta con mantener una sola
copia.

### 8.5. Instalación de PHP

``` yaml
- name: Instalar PHP y los módulos necesarios
  apt:
    name:
      - php
      - php-mysql
      - libapache2-mod-php
    state: present
```

-   `php`: instala el intérprete de PHP.
-   `php-mysql`: permite que PHP se conecte a MySQL.
-   `libapache2-mod-php`: integra PHP con Apache.

### 8.6. Reinicio de Apache

``` yaml
- name: Reiniciar el servidor web Apache
  service:
    name: apache2
    state: restarted
```

Reinicia Apache para aplicar los cambios que lo requieran.

### 8.7. Comprobaciones del frontend

Comprobar el estado de Apache:

``` bash
sudo systemctl status apache2
```

Validar la sintaxis de la configuración de Apache:

``` bash
sudo apache2ctl configtest
```

La respuesta `Syntax OK` indica que la sintaxis comprobada es válida.
También hay que confirmar que el sitio `000-default.conf` está
habilitado, porque copiarlo a `sites-available` no lo activa
automáticamente.

**Captura** ejecución del playbook y comprobación del
estado de Apache.

![Comprobación de Apache](images/apache-frontend.png)


## 9. Archivo de configuración de Apache: `templates/000-default.conf`

Este archivo define el sitio que Apache sirve en el frontend. Es una
configuración de Apache, no de Nginx.

Contenido compartido:

``` apache
ServerSignature Off
ServerTokens Prod

<VirtualHost *:80>
    DocumentRoot /var/www/html
    DirectoryIndex index.php index.html
    ServerName _dominio_

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory "/var/www/html">
        AllowOverride All
    </Directory>
</VirtualHost>
```

### 9.1. Ocultar información de la versión

``` apache
ServerSignature Off
ServerTokens Prod
```

Estas directivas reducen la información de identificación de Apache que
puede aparecer en determinadas respuestas y páginas de error.

### 9.2. Definir el sitio web

-   `<VirtualHost *:80>`: define el sitio que atiende las peticiones
    recibidas en el puerto 80.
-   `DocumentRoot /var/www/html`: establece el directorio de los
    archivos web.
-   `DirectoryIndex index.php index.html`: indica que Apache busque
    primero `index.php` y después `index.html`.
-   `ServerName`: especifica el nombre de dominio asociado al sitio.

### 9.3. Registros

-   `ErrorLog`: indica dónde se guardan los errores de Apache.
-   `CustomLog`: registra las peticiones de acceso.

### 9.4. Permitir el uso de `.htaccess`

``` apache
<Directory "/var/www/html">
    AllowOverride All
</Directory>
```

`AllowOverride All` permite que Apache aplique las directivas
compatibles incluidas en `.htaccess`. Esto puede ser necesario para las
reglas de enlaces permanentes de WordPress.

El dominio de `ServerName` debe coincidir con el dominio que realmente
se utiliza en el proyecto. El archivo también debe estar habilitado y
ser aceptado por Apache antes de considerarlo operativo.

## 10. Instalación y configuración del backend

### Archivo `playbooks/install_lamp_backend.yml`

Este playbook prepara el servidor backend para alojar MySQL, la base de
datos que utilizará WordPress.

### 10.1. Selección del servidor y carga de variables

``` yaml
- name: Playbook para instalar la pila LAMP en el Backend
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml
```

`vars_files`
carga los valores definidos en `variables.yml`.


### 10.2. Instalación de MySQL

``` yaml
- name: Instalar el sistema gestor de bases de datos MySQL
  apt:
    name: mysql-server
    state: present
```

Instala MySQL para almacenar las entradas, páginas, usuarios y opciones de WordPress.

### 10.3. Instalación de PyMySQL

``` yaml
- name: Instalar el módulo de pymysql
  apt:
    name: python3-pymysql
    state: present
```

Instala la biblioteca que permite a Python comunicarse con MySQL y que
puede ser necesaria para que los módulos de Ansible administren bases de datos y usuarios.

### 10.4. Configuración de la dirección de escucha de MySQL  

////


### 10.5. Comprobaciones del backend

``` bash
sudo systemctl status mysql
```

Se puede revisar la dirección de escucha y confirmar que el puerto 3306
está limitado a las interfaces previstas.

**Captura** estado del servicio MySQL y comprobación de su
configuración.

![Comprobación de MySQL](images/mysql-backend.png)


## 11. Despliegue de la base de datos de WordPress

### Archivo `playbooks/deploy_wordpress_backend.yml`

Este playbook crea la base de datos y el usuario de MySQL que utilizará
WordPress.

### 11.1. Creación de la base de datos

En el archivo recibido, la primera tarea elimina la base de datos
(`state: absent`) y después la vuelve a crear (`state: present`).


``` yaml
- name: Crear la base de datos de WordPress
  community.mysql.mysql_db:
    name: "{{ db.name }}"
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock
```

### 11.2. Creación del usuario de MySQL

El propósito de esta tarea es crear el usuario de WordPress y concederle
permisos sobre su base de datos.

``` yaml
- name: Crear el usuario de la base de datos
  community.mysql.mysql_user:
    name: "{{ db.user }}"
    password: "{{ db.password }}"
    priv: "{{ db.name }}.*:ALL"
    host: "{{ ip.frontend_private_ip }}"
    state: present
    login_unix_socket: /var/run/mysqld/mysqld.sock
```

-   `name`: nombre del usuario.
-   `password`: contraseña del usuario.
-   `priv`: permisos sobre las tablas de la base de datos especificada.
-   `host`: dirección desde la que el usuario puede conectarse.
-   `state: present`: garantiza que el usuario exista.
-   `login_unix_socket`: permite administrar MySQL mediante el socket
    local del backend.

La IP configurada en `host` debe coincidir con la dirección de origen
que MySQL ve cuando el frontend se conecta. Si hay dos frontend, ambos
deben estar autorizados. La colección `community.mysql` y el método de
autenticación deben ser compatibles con las versiones instaladas.

_Reinicio del servicio_


## 12. Configuración del servidor NFS

### Archivo `playbooks/setup_nfs_server.yml`

Este playbook prepara un servidor NFS (Network File System) para
compartir los archivos de WordPress con los servidores frontend. Así,
ambos frontend pueden acceder al mismo almacenamiento.

### 12.1. Selección del servidor

El grupo `nfs` identifica el servidor de almacenamiento compartido.

### 12.2. Actualización e instalación

El playbook actualiza los repositorios e instala el paquete
`nfs-kernel-server`, que permite exportar directorios a otros equipos.

### 12.3. Creación del directorio compartido

``` yaml
- name: Crear el directorio para WordPress
  file:
    path: /var/www/html
    state: directory
    owner: nobody
    group: nogroup
    mode: '0755'
```

Se crea `/var/www/html`, que será la ubicación compartida. Los permisos
y la propiedad deben ser compatibles con el usuario con el que Apache y
WordPress necesitan leer y escribir archivos.

### 12.4. Copia y configuración de `exports`

El playbook copia el archivo `exports` del proyecto a `/etc/exports` y
sustituye el marcador `nfscliente.client_ip` por la variable
correspondiente.

El archivo recibido contiene:

``` text
/var/www/html nfscliente.client_ip(rw,sync,no_root_squash,no_subtree_check)
```

-   `/var/www/html`: directorio exportado.
-   `nfscliente.client_ip`: cliente autorizado, sustituido por Ansible.
-   `rw`: permite lectura y escritura.
-   `sync`: confirma las escrituras una vez guardadas en el
    almacenamiento.
-   `no_root_squash`: conserva los privilegios de root del cliente sobre
    el recurso compartido.
-   `no_subtree_check`: desactiva determinadas comprobaciones de
    subdirectorios.

`no_root_squash` tiene implicaciones de seguridad y no debe utilizarse
sin valorar el riesgo. Además, si los dos frontend necesitan montar el
directorio, ambos deben estar autorizados en la configuración de
exportación.

### 12.5. Reinicio y comprobaciones

El playbook reinicia el servicio `nfs-kernel-server`. Después se puede
comprobar su estado:

``` bash
sudo systemctl status nfs-kernel-server
```

Y revisar los recursos exportados:

``` bash
sudo exportfs -v
```

**Captura** resultado de `exportfs -v`.

![Configuración del servidor NFS](images/nfs-server.png)



## 13. Configuración de los clientes NFS

### Archivo `playbooks/setup_nfs_client.yml`

Este playbook configura los servidores frontend para montar el
directorio compartido del servidor NFS en `/var/www/html`.

### 13.1. Selección de los servidores e instalación


Se ejecuta sobre el grupo `frontend`. El playbook actualiza los
repositorios e instala `nfs-common`, que proporciona las herramientas de
cliente NFS.

### 13.2. Montaje del directorio compartido

El playbook original utiliza un comando `mount` para montar el
directorio del servidor NFS en el directorio de WordPress. Después
ejecuta `df -h` y añade una entrada a `/etc/fstab`.

La idea de la entrada de montaje es:

``` text
IP_SERVIDOR_NFS:/var/www/html /var/www/html nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
```

`auto` permite intentar montar el recurso durante el arranque, `nofail`
permite continuar el arranque si el montaje falla, `tcp` solicita TCP y
`actimeo=1800` configura la caché de atributos.

### 13.3. Automatización recomendada del montaje

Para que el playbook pueda ejecutarse repetidamente sin fallar porque el
recurso ya está montado, es preferible utilizar el módulo
`ansible.posix.mount` en lugar de ejecutar `mount` directamente y
gestionar `/etc/fstab` por separado.


Después de instalar `nfs-common`, puedes sustituir las tareas originales
de montaje y edición de `/etc/fstab` por una tarea como esta:


``` yaml
- name: Asegurar que el directorio de montaje existe
  ansible.builtin.file:
    path: "{{ wordpress.directory }}"
    state: directory
    mode: '0755'

- name: Montar y registrar el recurso NFS
  ansible.posix.mount:
    src: "{{ nfsserver.server_ip }}:{{ wordpress.directory }}"
    path: "{{ wordpress.directory }}"
    fstype: nfs
    opts: "defaults,_netdev,nofail,noatime,tcp,actimeo=1800"
    state: mounted
```

`state: mounted` mantiene el recurso montado y asegura que la entrada
correspondiente esté presente en `/etc/fstab`. Es más adecuado para
repetir el playbook que ejecutar el comando `mount` en cada ocasión.

Antes de aplicar este cambio, verifica que `nfsserver.server_ip` tenga
la IP privada correcta y que el directorio remoto esté exportado. Si
`/var/www/html` ya contiene archivos locales de WordPress, ten presente
que montar NFS encima del directorio oculta temporalmente esos archivos
locales; realiza una copia de seguridad o migra los archivos al recurso
compartido antes de montarlo.

Si necesitas desmontar el recurso deliberadamente y eliminar su entrada
de `/etc/fstab`, puedes utilizar una tarea separada con `state: absent`,
pero no la incluyas en el despliegue normal:

``` yaml
- name: Desmontar y retirar la entrada del recurso NFS
  ansible.posix.mount:
    path: "{{ wordpress.directory }}"
    state: absent
```

No borres ni limpies `/var/www/html` mientras el recurso NFS esté
montado: podrías borrar archivos del almacenamiento compartido o recibir
errores por estar operando sobre un sistema de archivos montado.

### 13.4. Comprobaciones del cliente

``` bash
findmnt /var/www/html
df -h
```

El resultado de `findmnt` debe indicar que `/var/www/html` está montado
desde el servidor NFS. También hay que comprobar que los grupos de
seguridad de AWS permiten el tráfico NFS entre los clientes y el
servidor, y que el servidor exporta las direcciones de ambos frontend.

**Captura** resultado de `findmnt /var/www/html` en un
frontend.

![Montaje del directorio NFS](images/nfs-client.png)


## 14. Despliegue de WordPress en el frontend

### Archivo `playbooks/deploy_wordpress_frontend.yml`

Este playbook instala y configura WordPress en el grupo `frontend1`.
Utiliza WP-CLI para descargar WordPress, generar `wp-config.php`,
realizar la instalación inicial y configurar un tema, un plugin y los
enlaces permanentes.


### 14.1. Descarga de WP-CLI

``` yaml
- name: Descargar la utilidad WP-CLI
  get_url:
    url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
    dest: /usr/local/bin/wp
    mode: '0755'
```

Descarga WP-CLI y lo guarda en `/usr/local/bin/wp`. El permiso `0755`
permite ejecutar el archivo.

El playbook original también cambia el propietario del archivo a
`www-data`.

### 14.3. Descarga del código fuente

El playbook utiliza `wp core download` para descargar los archivos
principales de WordPress en `wordpress.directory`.

La tarea debe ejecutarse en un directorio preparado para recibir los
archivos. Si WordPress ya está instalado, hay que comprobar el estado
del directorio antes de repetirla.

### 14.4. Creación de `wp-config.php`

La tarea `wp config create` utiliza estas variables:

-   `db.name`: nombre de la base de datos.
-   `db.user`: usuario de MySQL.
-   `db.password`: contraseña del usuario.
-   La IP privada del backend: dirección del servidor MySQL.
-   `wordpress.directory`: directorio de instalación.

Este archivo contiene los parámetros de conexión entre WordPress y
MySQL. La dirección IP debe utilizar el mismo nombre de variable que se
haya definido en `variables.yml`.

### 14.5. Instalación inicial de WordPress

La tarea `wp core install` configura:

-   `certbot.domain`: URL utilizada por la instalación.
-   `wordpress.title`: título del sitio.
-   `wordpress.admin_user`: usuario administrador.
-   `wordpress.admin_password`: contraseña del administrador.
-   `wordpress.admin_email`: correo del administrador.
-   `wordpress.directory`: directorio de instalación.

La base de datos y el usuario deben existir previamente, y el frontend
debe poder conectarse al backend.

### 14.6. Permisos de los archivos

El playbook configura `www-data` como propietario y grupo de
`/var/www/html`, `wp-content` y `wp-content/upgrade`.


### 14.7. Instalación del tema Mindscape

Se utiliza WP-CLI para instalar y activar el tema `mindscape`, que
determina parte de la apariencia del sitio web.

Si se vuelve a ejecutar el playbook, debe comprobarse si el tema ya está
instalado para evitar errores innecesarios.

### 14.8. Instalación del plugin `wps-hide-login`

Se instala y activa el plugin `wps-hide-login`, que permite cambiar la
URL de acceso al panel de administración.

### 14.9. Configuración de la dirección de acceso

El playbook actualiza la opción `whl_page` con el valor de
`wordpress.hide_login_url`. De esta forma, se establece la ruta
personalizada que utiliza el plugin.

### 14.10. Configuración de los enlaces permanentes

La tarea `wp rewrite structure` configura la estructura `/%postname%/`,
que utiliza el nombre de la entrada en la URL.

Por ejemplo, una entrada puede utilizar una ruta similar a
`/mi-primera-entrada/`. Para que funcione, la configuración del servidor
web debe permitir las reglas de reescritura correspondientes.

### 14.11. Copia de `.htaccess`

El playbook copia `../htaccess/.htaccess` al directorio de WordPress.
Este archivo contiene reglas compatibles con Apache, por ejemplo, para
los enlaces permanentes.

La ruta de origen debe coincidir con la ubicación real del archivo en el
repositorio. Las reglas de `.htaccess` no se aplican automáticamente si
WordPress se sirve directamente mediante Nginx.

### 14.12. Configuración de HTTPS en `wp-config.php`

El playbook intenta añadir esta línea a `wp-config.php`:

``` php
$_SERVER['HTTPS'] = 'on';
```

La tarea utiliza un patrón `insertbefore` que busca una definición
concreta de `DB_NAME` con el valor `wordpress`. Si el nombre de la base
de datos o el formato de la línea son diferentes, puede no encontrar la
coincidencia.

Además, si HTTPS termina en el balanceador, WordPress debe reconocer
correctamente las peticiones HTTPS reenviadas por el proxy. Esta
configuración debe probarse con el dominio y el balanceador.


### 14.14. Comprobaciones de WordPress

Se puede comprobar la instalación con WP-CLI:

``` bash
sudo -u www-data wp core is-installed --path=/var/www/html
```

Si la instalación se ejecutó con `--allow-root`, también puede
comprobarse con el usuario adecuado para la instalación. Además, hay que
abrir el dominio y verificar que la página carga correctamente.

**Capturas** resultado de WP-CLI y página principal de
WordPress.

![WordPress instalado](images/wordpress-instalado.png)

## 15. Configuración de Nginx como balanceador de carga

### Archivo `playbooks/setup_load_balancer.yml`

Este playbook instala y configura Nginx en el servidor `balanceador`
para distribuir las peticiones web entre los dos servidores frontend.

### 15.1. Actualización e instalación

El playbook actualiza los repositorios y los paquetes del sistema, e
instala Nginx.

### 15.2. Copia de la plantilla

Copia `templates/load_balancer.conf` a
`/etc/nginx/sites-available/load_balancer.conf`.

A continuación, elimina el enlace del sitio predeterminado, sustituye
los marcadores de las dos IP frontend y del dominio, y crea un enlace
simbólico en `sites-enabled` para habilitar la configuración.

Al final solicita a systemd que recargue Nginx. 

### 15.3. Plantilla `templates/load_balancer.conf`

El archivo compartido es:

``` nginx
upstream frontend_servers {
    server balanceador.IP_FRONTEND_1;
    server balanceador.IP_FRONTEND_2;
}

server {
    listen 80;
    server_name certbot.domain;

    location / {
        proxy_pass http://frontend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

#### Definición de los frontend


Los textos `balanceador.IP_FRONTEND_1` y `balanceador.IP_FRONTEND_2` se
sustituyen por las direcciones configuradas en las variables del
proyecto.

#### Configuración del servidor

-   `listen 80`: hace que Nginx escuche en el puerto HTTP 80.
-   `server_name`: define el dominio que atiende el balanceador.
-   `proxy_pass`: reenvía las peticiones al grupo `frontend_servers`.
-   `proxy_set_header Host $host`: conserva el nombre de dominio
    solicitado.
-   `proxy_set_header X-Forwarded-For`: transmite la cadena de
    direcciones IP de origen.
-   `proxy_set_header X-Forwarded-Proto https`: informa al frontend de
    que el protocolo original es HTTPS.

La cabecera `X-Forwarded-Proto` está fijada en `https`. Esto solo es
correcto si el diseño garantiza que las peticiones reenviadas
corresponden a conexiones HTTPS externas

### 15.4. Comprobaciones del balanceador

``` bash
sudo nginx -t
sudo systemctl status nginx
```

`nginx -t` comprueba la sintaxis de la configuración. Después, hay que
verificar que el dominio apunta a la IP pública del balanceador y que
ambos frontend responden.

**Captura** resultado de `nginx -t` y acceso a WordPress a
través del dominio.

![Configuración del balanceador](images/balanceador-nginx.png)

## 16. Configuración de HTTPS mediante Let's Encrypt

### Archivo `playbooks/setup_letsencrypt_https.yml`

Este playbook configura HTTPS en el servidor `balanceador` mediante
Certbot y Let's Encrypt.

### 16.1. Selección del servidor y variables

El playbook se ejecuta sobre el grupo `balanceador`, con privilegios de
administrador, y carga `vars/variables.yml`.

Utiliza `certbot.email` para el correo electrónico y `certbot.domain`
para el dominio del certificado.

### 16.2. Desinstalación de Certbot instalado mediante APT

Se elimina el paquete `certbot` instalado por APT para preparar la
instalación mediante Snap.

### 16.3. Instalación de Certbot mediante Snap

Se instala Certbot con Snap y se habilita el modo clásico.

### 16.4. Solicitud del certificado

El comando utilizado es equivalente a:

``` bash
sudo certbot --nginx -m CORREO --agree-tos --no-eff-email -d DOMINIO --non-interactive
```

Los marcadores `CORREO` y `DOMINIO` se sustituyen por las variables del
proyecto. El parámetro `--nginx` permite a Certbot trabajar con la
configuración de Nginx.

Para que la solicitud funcione, el dominio debe apuntar al balanceador,
Nginx debe estar instalado y correctamente configurado, y los puertos 80
y 443 deben estar permitidos en los grupos de seguridad
correspondientes.

### 16.5. Orden y comprobaciones

El balanceador debe estar instalado y tener una configuración válida
antes de ejecutar Certbot. Después se puede probar el acceso mediante
`https://` y comprobar la renovación del certificado.

**Captura** salida de Certbot y acceso al dominio mediante
HTTPS.

![HTTPS con Let's Encrypt](images/https-lets-encrypt.png)


## 9. Archivo principal main.yml

El archivo `main.yml` organiza la ejecución de los playbooks y permite coordinar el despliegue de la infraestructura.

Cada playbook se comprobará individualmente antes de integrarlo en la ejecución general.

## 10. Configuración del dominio No-IP

Se ha configurado un dominio mediante el servicio No-IP para acceder al sitio web utilizando un nombre de dominio en lugar de una dirección IP pública.

Este dominio se utiliza para facilitar el acceso a WordPress y debe estar asociado a la dirección IP pública del punto de entrada de la infraestructura, normalmente la del balanceador de carga.

Además, el dominio debe coincidir con la configuración de WordPress y, si se habilita HTTPS, con la configuración del certificado SSL/TLS correspondiente.

La configuración del dominio permite acceder al sitio web mediante una dirección más sencilla de recordar y facilita la posterior configuración del acceso seguro mediante HTTPS.

## 11. Plantillas y archivos auxiliares

### Archivo `.htaccess`

Se ha creado el archivo `.htaccess` para definir reglas de configuración y acceso relacionadas con WordPress cuando se utiliza Apache. Permite gestionar determinadas reglas del sitio web, como las relacionadas con las URL amigables y otras directivas compatibles con este servidor.

Su uso dependerá de la configuración del servidor web. Si se utiliza Nginx, las reglas de Apache deberán adaptarse a la configuración de Nginx, ya que no se aplican directamente.

### Plantilla `000-default.conf`

Plantilla utilizada para configurar el sitio web en el servidor web. Contiene los parámetros necesarios para definir cómo se sirve WordPress, incluyendo la configuración del sitio y las rutas correspondientes.

Ansible utiliza esta plantilla para generar y desplegar la configuración del servidor durante la instalación y configuración de WordPress.

### Plantilla `login-balancer.conf`

Plantilla utilizada para configurar el *load balancer* (balanceador de carga). Permite definir los parámetros necesarios para recibir las peticiones de los usuarios y distribuirlas entre los servidores web configurados en la infraestructura.

Ansible utiliza esta plantilla para generar y desplegar la configuración del balanceador, facilitando el acceso al sitio web a través de un único punto de entrada.


## 12. Almacenamiento compartido NFS

El servidor NFS proporcionará almacenamiento compartido a los servidores autorizados.

Los clientes NFS se configurarán para acceder a los directorios compartidos y permitir que los frontends utilicen los archivos comunes de WordPress.

## 13. HTTPS mediante Let's Encrypt

Se utilizará Let's Encrypt para configurar certificados HTTPS cuando el dominio y los servicios web estén preparados.

## 14. Proceso de implantación

1. Crear las instancias EC2.
2. Configurar los grupos de seguridad.
3. Instalar Git y preparar Ansible.
4. Configurar el inventario y las variables.
5. Instalar y configurar los servicios web.
6. Preparar la base de datos.
7. Configurar el servidor y los clientes NFS.
8. Configurar el balanceador de carga.
9. Desplegar WordPress.
10. Configurar HTTPS.
11. Ejecutar las pruebas finales.

## 14. Seguridad

- Restringir el acceso SSH.
- Evitar la exposición innecesaria de puertos.
- Utilizar IP privadas para las comunicaciones internas cuando corresponda.
- Proteger las claves SSH y las credenciales de AWS.
- No publicar contraseñas reales en GitHub.
- Utilizar Ansible Vault para proteger las variables sensibles.
- Limitar el acceso al servidor NFS.
- Utilizar HTTPS para proteger las comunicaciones.