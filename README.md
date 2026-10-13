# Proyecto Olga Valle Peña

## Implantación de una plataforma web con WordPress en AWS

# 1. Introducción

El objetivo de este proyecto es implantar una plataforma web basada en WordPress sobre Amazon Web Services (AWS).

## 2. Requisitos previos

- Cuenta de AWS con acceso a EC2.
- Clave SSH para acceder a las instancias.
- Ansible instalado en la máquina de control.
- Git instalado.

## 3. Estructura del repositorio
```
.
├── README.md
├── templates
│   └── 000-default.conf
    └──load_balancer.conf

└── inventory
│   └── inventory
└── playbooks
│   ├── setup_load_balancer.yml
│   ├── install_lamp_frontend.yml
│   ├── install_lamp_backend.yml
│   ├── setup_nfs_server.yml
│   ├── setup_nfs_client.yml
│   ├── setup_letsencrypt_certificate.yml
│   ├── deploy_wordpress_backend.yml
│   └── deploy_wordpress_frontend.yml
└── vars
│   └── variables.yml
└── main.yml
└──.gitignore
└── exports

```

## 4. Preparación de Ubuntu e instalación de Git

Actualizar los paquetes:

```bash
sudo apt update
sudo apt upgrade -y
```

Instalar Git:

```bash
sudo apt install git -y
```

Comprobar la instalación:

```bash
git --version
```

Configurar el usuario:

```bash
git config --global user.name "TU_NOMBRE"
git config --global user.email "TU_CORREO"
```

Clonar el repositorio:

```bash
git clone URL_DEL_REPOSITORIO
```

## 5. Inventario de Ansible

El archivo `inventory/inventory` contiene los grupos de servidores y sus direcciones IP.

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

Las direcciones IP son ejemplos y deben sustituirse por las direcciones reales.

## 6. Variables

El archivo `vars/variables.yml` centraliza las variables de configuración de los servidores.

Se incluyen variables para:

- Direcciones IP públicas y privadas.
- Base de datos: nombre, usuario y contraseña.
- Servidor: dominio y correo electrónico.
- WordPress: título, administrador, contraseña, correo electrónico y URL de acceso.
- NFS: dirección IP del servidor y de los clientes.
- Balanceador: direcciones IP de los servidores web.

Las contraseñas y las claves privadas no deben publicarse en el repositorio.

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

Si un archivo ya está siendo rastreado por Git, incluirlo en `.gitignore` no lo elimina del repositorio.

## 8. Playbooks de Ansible

Los playbooks creados son:

- `deploy_wordpress_backend.yml`: tareas relacionadas con el despliegue de WordPress en el backend.
- `deploy_wordpress_frontend.yml`: despliegue de WordPress en los frontends.
- `install_nginx_backend.yml`: instalación y configuración de Nginx en el backend.
- `install_nginx_frontend.yml`: instalación y configuración de Nginx en los frontends.
- `setup_letsencrypt_certificate.yml`: configuración de certificados HTTPS.
- `setup_load_balancer.yml`: configuración del balanceador de carga.
- `setup_nfs_client.yml`: configuración de los clientes NFS.
- `setup_nfs_server.yml`: configuración del servidor NFS.

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