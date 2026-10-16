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