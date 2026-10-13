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