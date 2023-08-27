# Despliegue de Wordpress usando Vagrant y Ansible

- Master ![Build Status on Master Branch](https://github.com/cppmx/wordpress_chef/actions/workflows/ci.yaml/badge.svg)

Este proyecto es para una tarea de la Maestría en Desarrollo y Operaciones de UNIR.

El Objetivo es desplegar Wondpress usando Vagrant y Ansible.

## Supuestos

- Se espera que la red de las VMs sea 192.168.56.0/24. Si VirtualBox tiene otro rango de red entonces hay que ajustar el archivio `.env` con los valores adecuados.

## Pre-requisitos

- Necesitas tener instalado Git
- Necesitas tener instalado Vagrant 2.3.7 o superior
- Necesitas tener instalado VirtualBox 7.0 o superior

Instala el plugin `vagrant-env` para poder cargar variables ed ambiente desde el archivo `.env`

```bash
 vagrant plugin install vagrant-env
```

## Arquitectura

El proyecto se compone de tres servicios, cada uno deployado en una VM individual:

- database: En esta VM se instala MySQL.
- wordpress: En esta VM se instala el servidor web Apache y la aplicación Wordpress es instalada para ser servida por el servidor web.
- proxy: Em esta VM se instala un proxy Nginx el cual será el punto de entrada a la aplicación.

En el siguiente diagrama se pueden ver cómo se relacionan las VMs y los puertos de comunicación que esa cada una de ellas:

```mermaid
graph LR;
    A("Usuario") --> |80| B("proxy
    192.168.56.2") --> |8080| C("wordpress
  192.168.56.10") ---> |3306| D[("database
  192.168.56.20")]
```

## Configuraciones

En el archivo `.env` se pueden definir algunos valores como las IPs de las máquinas virtuales y el usuario y el password de la BD que se usará para configurar Wordpress.

Antes de levantar Vagrant se puede definir la caja que se usará. Mira el siguiente diagrama:

```mermaid
graph TB;
    A[Inicio] --> B{BOX_NAME?}
    B -->|No| C["Deploy
    ubuntu/focal64"]
    B -->|Si| D["Deploy
    generic/centos8"]
    C --> E[Fin]
    D --> E[Fin]
```

Lee la sección [Uso](#uso) para ver ejemplos de esto.

## Uso

Para levantar las dos máquinas virtuales con Ubuntu 20.04 ejecuta el comando:

```bash
 vagrant up
```

Para levantar las dos máquinas virtuales con CentOS 8 ejecuta el comando:

```bash
 BOX_NAME="generic/centos8" vagrant up
```

Si quieres mezclar las versiones puedes hacerlo del siguiente modo.

### Wordpress con Ubuntu y MySQL con CentOS:

```bash
 vagrant up wordpress
 BOX_NAME="generic/centos8" vagrant up database
```

### Wordpress con CentOS y MySQL con Ubuntu:

```bash
 BOX_NAME="generic/centos8" vagrant up wordpress
 vagrant up database
```

## Wordpress

Una vez que se hayan levantado todas las VMs podrás acceder a Wordpress en la página: http://192.168.56.2/

## Despliegue

[![asciicast](https://asciinema.org/a/6W0M6dJut80fWpPEXVhNnOtaJ.svg?autoplay=1)](https://asciinema.org/a/6W0M6dJut80fWpPEXVhNnOtaJ)
