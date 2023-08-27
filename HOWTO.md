# Proceso de creación del proyecto

## Creación del espacio de trabajo

El primer paso es crear un directorio de trabajo vacío:

```bash
home> mkdir -p /home/user/development/vagrant/unir/ansible

home> cd /home/user/development/vagrant/unir/ansible
```

Ahora vamos a crear los directorios donde se crearán los playbooks y sus tareas:

```bash
ansible> mkdir -p provisioning/centos/tasks
ansible> mkdir -p provisioning/ubuntu/tasks
ansible> mkdir -p provisioning/common/tasks
ansible> mkdir -p provisioning/templates
```

Nos quedará una estructura de directorios como la siguiente:

```bash
ansible> tree
.
└── provisioning
    ├── centos
    │   └── tasks
    ├── common
    │   └── tasks
    ├── templates
    └── ubuntu
        └── tasks

8 directories
```

## Definición de las máquinas virtuales

Antes de iniciar con la creación de los playbooks vamos a crear las máquinas virtuales. Los playbooks serán cargados en una máquina virtual y serán ejecutadas, así que necesitaremos tres máquinas virtuales, una para cada uno de los playbooks. Vamos a crear el archivo *Vagrantfile* en el directorio inicial de trabajo será `/home/user/development/vagrant/unir/ansible/`:

```bash
ansible> touch Vagrantfile
```

Iniciamos el archivo Vagrantfile con el siguiente código:

```ruby
Vagrant.configure("2") do |config|
    
end
```

Necesitamos que haya comunicación entre las máquinas virtuales, es decir, que la base de datos debe saber de dónde va a recibir peticiones, la VM wordpress necesita saber dónde está la VM de la base de datos, y el proxy necesita saber dónde está la VM wordpress. Esto sólo se puede saber conociendo las IPs de las otras VMs, podemos copiar las IPs de todas las VMs en todos lados, o podemos ser un poco más estrictos y simular un ambiente más real donde cada VM sólo puede saber la ubicación de las VMs con las que va a interactuar. Seguiremos el segundo caso, sólo pondremos la información mínima necesaria para funcionar en cada VM.

Ahora bien, las IPs de cada VM pueden cambiar, no son fijas. Bueno, en teoría son fijas, pero pueden cambiar porque cada instalación de vagrant y VirtualBox es diferente. Para este ejemplo consideramos que las IPs están en el rango de 192.168.56.0/16, pero puede ser que otra persona que quiera usar este proyecto tenga un rango de IPs diferente. El problema con esto es que vamos a usar estas IPs en varias partes de los playbooks, sería muy difícil actualizar las IPs en caso de que el rango de IPs disponibles sea otro. Por eso usaremos variables de ambiente para definir las IPs de cada una de las VMs.

Para poder incluir esta configuración de las variables de ambiente en el repositorio usaremos un archivo llamado `.env` el cual debe de estar al mismo nivel que el archivo *Vagrantfile*. El contenido de este archivo será el siguiente:

```bash
wordpress> cat .env
DB_IP=192.168.56.20
WP_IP=192.168.56.10
PROXY_IP=192.168.56.2
```

Para que vagrant pueda leer el contenido del archivo `.env` instalaremos el plugin `vagrant-env`:

```bash
ansible> vagrant plugin install vagrant-env
```

También vamos a crear lo que seŕa nuestro archivo de invetario, así que al mismo nivel que el archivo `Vagrantfile` y `.env` crearemos un archivo llamado `inventory`:

```bash
ansible> touch inventory
```

Y el contenido de este archivo será el siguiente:

```ruby
[all:vars]
db_name="wordpress"
db_user="wordpress"
db_pswd="testP@55word"
wp_ip="192.168.56.10"
db_ip="192.168.56.20"

[database]
192.168.56.20

[wordpress]
192.168.56.10

[proxy]
192.168.56.2
```

Hay que tener en cuenta que si cambiamos el rango de IPs en el archivo `.env` entonces también debemos cambiar las IPs del archivo de invetario `inventory`.

Ahora iniciaremos con la creación del archivo `Vagrantfile`:

```ruby
Vagrant.configure("2") do |config|
    config.env.enable
end
```

Ahora procederemos a definir cada una de las VMs:

```ruby
Vagrant.configure("2") do |config|
    config.env.enable

    config.vm.define "database" do |db|
        
    end

    config.vm.define "wordpress" do |sitio|
        
    end

    config.vm.define "proxy" do |proxy|
        
    end
end
```

En cada una de las VMs vamos a definir la caja que usaremos, recordemos que el sistema operativo de las VMs que usa Vagrant viene empaquetado y se llaman cajas. El proyecto nos pide que las recetas se puedan ejecutar en Ubuntu y CentOS. Para poder usar un sistema operativo o el otro vamos a usar otra variable de ambiente la cual llamaremos `BOX_NAME`.

```ruby
Vagrant.configure("2") do |config|
    config.env.enable

    config.vm.define "database" do |db|
        db.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
    end

    config.vm.define "wordpress" do |sitio|
        sitio.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
    end

    config.vm.define "proxy" do |proxy|
        proxy.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
    end
end
```

Como se puede ver en el bloque anterior, cuando la variable de entorno `BOX_NAME` no esté definida se usará Ubuntu, de lo contrario se usará la caja que definamos en esta variable de entorno. Esto nos va a permitir decidir que sistema operativo queremos usar al momento de levantar las VMs. Si queremos usar CentOS sólo haremos lo siguiente:

```bash
actividad> BOX_NAME="generic/centos8" vagrant up
```

Y si queremos usar Ubuntu entonces basta con ejecutar lo siguiente:

```bash
actividad> vagrant up
```

Recordemos que en la estructura de directorios que definimos al inicio, creamos dos directorios dentro de provisioning, uno llamado centos y otro llamado ubuntu. La idea es que cuando usames una caja de Ubuntu pues se ejecuten las tareas que están dentro de `provisioning/ubuntu` y cuando la caja sea de CentOS pues se ejecuten las tareas de `provisioning/centos`. Para esto necesitamos definir una variable a la cula llamaremos `os_type`, la cual le pasaremos más adelante a Ansible:

```ruby
Vagrant.configure("2") do |config|
    config.env.enable

    # Determinar el valor de os_type basado en el valor de BOX_NAME
    os_type = case ENV["BOX_NAME"]
        when "generic/centos8"
            "centos"
        else
            "ubuntu"
    end

    config.vm.define "database" do |db|
        db.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
    end

    config.vm.define "wordpress" do |sitio|
        sitio.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
    end

    config.vm.define "proxy" do |proxy|
        proxy.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
    end
end
```

El siguiente paso es definir la IP y el hostname de cada VM, recordemos que esta información se va a cargar desde el archivo `.env`, así que nuestro *Vagrantfile* se verá así:

```ruby
Vagrant.configure("2") do |config|
    config.env.enable

    # Determinar el valor de os_type basado en el valor de BOX_NAME
    os_type = case ENV["BOX_NAME"]
        when "generic/centos8"
            "centos"
        else
            "ubuntu"
    end

    config.vm.define "database" do |db|
        db.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
        db.vm.network "private_network", ip: ENV["DB_IP"]
        db.vm.hostname = "db.unir.mx"
    end

    config.vm.define "wordpress" do |sitio|
        sitio.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
        sitio.vm.network "private_network", ip: ENV["WP_IP"]
        sitio.vm.hostname = "wordpress.unir.mx"
    end

    config.vm.define "proxy" do |proxy|
        proxy.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
        proxy.vm.network "private_network", ip: ENV["PROXY_IP"]
        proxy.vm.hostname = "wordpress.unir.mx"
    end
end
```

Ahora vamos a agregar la configuración de Ansible para cada VM:

```ruby
Vagrant.configure("2") do |config|
    config.env.enable

    # Determinar el valor de os_type basado en el valor de BOX_NAME
    os_type = case ENV["BOX_NAME"]
        when "generic/centos8"
            "centos"
        else
            "ubuntu"
    end

    config.vm.define "database" do |db|
        db.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
        db.vm.hostname = "db.unir.mx"
        db.vm.network "private_network", ip: ENV["DB_IP"]

        db.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.compatibility_mode = "2.0"
            ansible.playbook = "provisioning/database.yml"
            ansible.inventory_path = "./inventory"
            ansible.extra_vars = {
                os_type: os_type
            }
        end
    end

    config.vm.define "wordpress" do |sitio|
        sitio.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
        sitio.vm.hostname = "wordpress.unir.mx"
        sitio.vm.network "private_network", ip: ENV["WP_IP"]

        sitio.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.compatibility_mode = "2.0"
            ansible.playbook = "provisioning/wordpress.yml"
            ansible.inventory_path = "./inventory"
            ansible.extra_vars = {
                os_type: os_type
            }
        end
    end

    config.vm.define "proxy" do |proxy|
        proxy.vm.box = ENV["BOX_NAME"] || "ubuntu/focal64"
        proxy.vm.hostname = "wordpress.unir.mx"
        proxy.vm.network "private_network", ip: ENV["PROXY_IP"]

        proxy.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.compatibility_mode = "2.0"
            ansible.playbook = "provisioning/proxy.yml"
            ansible.inventory_path = "./inventory"
            ansible.extra_vars = {
                os_type: os_type
            }
        end
    end
end
```

La configuración de Ansible de cada VM continie lo siguiente:

- `ansible.verbose`: Habilitamos las líneas de depuración, esto nos va a ayudar a hacer un análisis simple en caso de que algo no se ejecute correctamente.
- `ansible.compatibility_mode = "2.0"`: Establecemos la versión de Ansible. Vagrant soporta las versiones 1.8 y 2.0 de Ansible.
- `ansible.playbook = "playbook.yml"`: Vamos a indicarle a Vagrant cuál plyabook debe ejecutar al momento de levantar la VM.
- `ansible.inventory_path = "./inventory"`: Le especificamos la ruta donde se encuentra nuestro archivo de inventario. Aquí usaremos el archivo de inventario que creamos hace unos instantes.
- `ansible.extra_vars = {}`: Aquí le podemos pasar variables extras a Ansible.

Listo, ya tenemos todo para empezar a trabajar con la creación de los playbooks.

## Estructura general de los playbooks

Voy a usar la siguiente estructura para definir cada uno de los tres playbooks:

```yaml
---
- name: Nombre del Plabook
  hosts: Nombre_del_host
  become: yes    # En todos los casos usaremos permisos elevados para ejecutar las tareas de instalación
  vars:
    load_database_tasks: true    # Variable que determina si debemos cargar o no las tareas de instalación y configuración de base de datos
    load_wordpress_tasks: false  # Variable que determina si debemos cargar o no las tareas de instalación y configuración de Wordpress
    load_proxy_tasks: false      # Variable que determina si debemos cargar o no las tareas de instalación y configuración del proxy
  roles:
    - common                     # En todos los casos ejecutaremos una serie de tareas comunes
    - "{{ os_type }}"            # Con esta variable que se definió en el archivo Vagrantfile sabremos si debemos cargar tareas para Ubuntu o para CentOS
  handlers:                      # En esta sección se definiran tareas globales o repetitivas
    - name: Nombre de la tarea global
```

La idea es crear las tres playbooks siguiendo este molde el cual nos permitirá cargar las tareas adecuadas en base al valor de las variables definidas.

## Playbook para la base de datos

El primer playbook que vamos a crear será el de la base de datos:

```bash
ansible> touch provisioning/database.yml
```

Siguiendo [la estructura general de los plyabooks](#estructura-general-de-los-playbooks), el contenido de este archivo será el siguiente:

```yaml
---
- name: Install and configure database
  hosts: database
  become: yes
  vars:
    load_database_tasks: true
    load_wordpress_tasks: false
    load_proxy_tasks: false
  roles:
    - common
    - "{{ os_type }}"
  handlers:
    - name: Restart MySQL Service
      service:
        name: mysql
        state: restarted
```

## Playbook para Wordpress

Ahora vamos a crear el playbook para la instalación y configuración de Wordpress:

```bash
ansible> touch provisioning/wordpress.yml
```

Siguiendo [la estructura general de los plyabooks](#estructura-general-de-los-playbooks), el contenido de este archivo será el siguiente:

```yaml
---
- name: Install and configure WordPress
  hosts: wordpress
  become: yes
  vars:
    load_database_tasks: false
    load_wordpress_tasks: true
    load_proxy_tasks: false
    wordpress_related_tasks:
      - apache.yml
      - wordpress.yml
      - post_install.yml
  roles:
    - common
    - "{{ os_type }}"
  handlers:
    - name: Restart Apache Service
      systemd:
        name: apache2
        state: restarted
    - name: Restart httpd service
      systemd:
        name: httpd
        state: restarted
```

Se agregó una variable extra llamada `wordpress_related_tasks`, más adelante usaremos esta variable para cargar todas las tareas que aquí se han definido.

## Playbook para el proxy

Ahora vamos a crear el playbook para la instalación y configuración del proxy:

```bash
ansible> touch provisioning/proxy.yml
```

Siguiendo [la estructura general de los plyabooks](#estructura-general-de-los-playbooks), el contenido de este archivo será el siguiente:

```yaml
---
- name: Install and configure nginx
  hosts: proxy
  become: yes
  vars:
    load_database_tasks: false
    load_wordpress_tasks: false
    load_proxy_tasks: true
  roles:
    - common
    - "{{ os_type }}"
  handlers:
    - name: Restart nginx service
      service:
        name: nginx
        state: restarted
```

## Definición de las tareas comunes

En todos los playbooks la primer tarea o rol que se invoca es `common`, así que vamos a crearlo del siguiente modo:

```bash
ansible> touch provisioning/common/tasks/main.yml
```

El contenido de este archivo será el siguiente:

```yaml
---
- name: Update and upgrade apt packages
  apt:
    upgrade: yes
    update_cache: yes
    cache_valid_time: 86400 
  when: ansible_pkg_mgr == 'apt'

- name: Update and upgrade apt packages
  yum:
    upgrade: yes
    update_cache: yes
    cache_valid_time: 86400 
  when: ansible_pkg_mgr == 'yum'

- name: Install common packages
  package:
    name: "{{ item }}"
    state: present
  loop:
    - epel-release
    - git
  when: ansible_pkg_mgr == 'yum'

- name: Install common packages
  package:
    name: "{{ item }}"
    state: present
  loop:
    - git
  when: ansible_pkg_mgr == 'apt'
```

Entonces cada que se levante una VM esta será actualizada, y también se instalarán algunos paquetes comunes en cada SO.

## Definición de los roles por SO

Aunque la instalación y configuración de MySQL es genérica, existen algunas pequeñas diferencias entre hacerlo en un sistema operativo basado en Debian y uno basado en Red Hat. Estas pequeñas diferencias son las que nos lllevan a tener roles basados en el sistema operativo. Lo mismo pasa con Apache, Wordpres y Nginx.

En el archivo `Vagrantfile` definimos una variable llamada `os_type` la cual puede tener el valor de `ubuntu` o `centos`, según el sistema operativo de la VM. El valor de esta variable lo estamos usando para definir el rol que debe ser ejecutado. Es decir, que cuando el valor de esta variable sea `ubuntu` entonces se llamará al archivo `provisioning/ubuntu/tasks/main.yml`, y cuando esta tenga el valor de `centos` entonces se llamará al archivo `provisioning/centos/tasks/main.yml`.

Entonces vamos a crear estos dos archivos:

```bash
ansible> touch provisioning/centos/tasks/main.yml
ansible> touch provisioning/ubuntu/tasks/main.yml
```

El contenido para ambos archivos será el siguiente:

```yaml
---
- include_tasks: database.yml
  when: load_database_tasks

- name: Include WordPress related tasks
  include_tasks: "{{ item }}"
  loop: "{{ wordpress_related_tasks }}"
  when: load_wordpress_tasks

- include_tasks: proxy.yml
  when: load_proxy_tasks
```

Como se puede ver el el bloque anterior, estamos usando el valor de las variables `load_database_tasks`,`load_wordpress_tasks` y `load_proxy_tasks` para cargar las tareas adecuadas. Estas variables fueron definidas en los playbooks `provisioning/database.yml`, `provisioning/wordpress.yml` y `provisioning/proxy.yml`.

Así, si se levanta la VM para la base de datos, y el valor de `os_type` es ubuntu, entoces se ejecutará el playbook `provisioning/database.yml`, este a su vez mandará llamar a `provisioning/ubuntu/tasks/main.yml` con los siguientes valores:

- `load_database_tasks: true`
- `load_wordpress_tasks: false`
- `load_proxy_tasks: false`

Y esto hará que se ejecuten las tareas del archivo `provisioning/ubuntu/tasks/database.yml`.

## Tareas para la instalación y configuración de la base de datos

Vamos a crear los archivos de tareas para la base de datos en cada uno de los roles:

```bash
ansible> touch provisioning/centos/tasks/database.yml
ansible> touch provisioning/ubuntu/tasks/database.yml
```

Contenido del archivo provisioning/centos/tasks/database.yml:

```yaml
---
- name: Install MySQL server
  package:
    name: mysql-server
    state: present

- name: Start and enable MySQL service
  systemd:
    name: mysqld
    state: started
    enabled: yes

- name: Create MySQL database
  command: mysql -e "CREATE DATABASE {{ db_name }};"
  register: creation
  changed_when: creation.rc == 0

- name: Create MySQL user and grant privileges
  shell: >
    mysql -e "CREATE USER '{{ db_user }}'@'{{ wp_ip }}' IDENTIFIED BY '{{ db_pswd }}';
    GRANT ALL PRIVILEGES ON wordpress.* TO '{{ db_user }}'@'{{ wp_ip }}';
    FLUSH PRIVILEGES;"
  register: privileges
  changed_when: privileges.rc != 0

- name: Open MySQL port in firewall
  firewalld:
    port: 3306/tcp
    permanent: yes
    state: enabled

- name: Reload firewall rules
  command: firewall-cmd --reload
```

Contenido del archivo provisioning/ubuntu/tasks/database.yml:

```yaml
---
- name: Install MySQL server
  apt:
    name: mysql-server
    state: present

- name: Start and enable MySQL service
  systemd:
    name: mysql
    state: started
    enabled: yes

- name: Create MySQL database
  command: mysql -e "CREATE DATABASE {{ db_name }};"
  register: creation
  changed_when: creation.rc == 0

- name: Create MySQL user and grant privileges
  shell: >
    mysql -e "CREATE USER '{{ db_user }}'@'{{ wp_ip }}' IDENTIFIED BY '{{ db_pswd }}';
    GRANT ALL PRIVILEGES ON wordpress.* TO '{{ db_user }}'@'{{ wp_ip }}';
    FLUSH PRIVILEGES;"
  register: privileges
  changed_when: privileges.rc != 0

- name: Update MySQL bind address
  replace:
    path: /etc/mysql/mysql.conf.d/mysqld.cnf
    regexp: 'bind-address\s*=\s*127.0.0.1'
    replace: 'bind-address = {{ db_ip }}'
  when: ansible_pkg_mgr == 'apt' and ansible_distribution == 'Ubuntu'
  notify: Restart MySQL Service
```

La diferencia principal entre ambos scripts es que en los sistemas basados en Red Hat, como es el caso de CentOS, todos los puertos están cerrados de forma predeterminada, así que hayq ue habilitar el puerto `3306` en el firewall para que se puedan recibir conexiones externas. En el caso de los sistemas Debian los puertos mpas comunes siempre están abiertos, pero la instalación base de MySQL considera que las conexiones simpre serán locales, así que hay que hacer el binding en la IP externa.

## Tareas para la instalación y configuración de apache

Vamos a crear los archivos de tareas para el servdior apache en cada uno de los roles:

```bash
ansible> touch provisioning/centos/tasks/apache.yml
ansible> touch provisioning/ubuntu/tasks/apache.yml
```

Contenido del archivo provisioning/centos/tasks/apache.yml:

```yaml
---
- name: Install required packages
  package:
    name:
      - httpd
      - php
      - php-mysqlnd
      - php-json
      - unzip
      - curl
    state: present

- name: Create info.php file
  copy:
    content: "<?php\nphpinfo();\n?>"
    dest: /var/www/html/info.php

- name: Enable httpd_can_network_connect on SELinux
  command: setsebool -P httpd_can_network_connect on

- name: Enable httpd_can_network_connect_db on SELinux
  command: setsebool -P httpd_can_network_connect_db on

- name: Add port 8080 to firewall rules
  command: firewall-cmd --zone=public --add-port=8080/tcp --permanent

- name: Add port 80 to firewall rules
  command: firewall-cmd --zone=public --add-port=80/tcp --permanent

- name: Reload firewall rules
  command: firewall-cmd --reload

- name: Start and enable httpd service
  systemd:
    name: httpd
    enabled: yes
    state: started
```

Contenido del archivo provisioning/ubuntu/tasks/apache.yml:

```yaml
---
- name: Install packages for Apache, PHP, and MySQL
  apt:
    name:
      - apache2
      - php
      - php-mysql
      - php-mysqlnd
      - php-mysqli
      - php-json
      - unzip
      - curl
    state: present

- name: Start and enable Apache service
  systemd:
    name: apache2
    state: started
    enabled: yes

- name: Create info.php file
  copy:
    content: "<?php\nphpinfo();\n?>"
    dest: /var/www/html/info.php
  notify: Restart Apache Service
```

La diferencia más grande entre ambos scripts es que el paquete de Apache se llama diferente, en los sistemas basados en Red Hat se llama `httpd`, mientras que en los sistemas Debian se llama `apache2`. Además la lista de dependencias para hacer funcionar correctamente Wodpress es diferente, se requieren una serie de paquetes extras en los sistemas Debian. También hay difencias en el tema de seguridad, en los sistemas basados en Bedian basta con la instalación de Apache para que las aplicaciones web se puedan conctar con otros sistemas, pero en el caso de los sistemas basados en Red Hat necesitamos habilitar los permisos de SELinux, de lo contrario no se podrá comunicar con la base de datos. Además hay que abrir el firewall para que puermita poder acceder desde a fuera.

## Tareas para la instalación y configuración de Wordpress

Vamos a crear los archivos de tareas para el sitio wordpress en cada uno de los roles:

```bash
ansible> touch provisioning/centos/tasks/wordpress.yml
ansible> touch provisioning/ubuntu/tasks/wordpress.yml
```

Contenido del archivo provisioning/centos/tasks/wordpress.yml:

```yaml
---
- name: Create /opt directory
  file:
    path: /opt/
    state: directory
    owner: root
    group: root

- name: Download Wordpress
  command: curl -o /tmp/wordpress.zip https://wordpress.org/latest.zip
  args:
    creates: /tmp/wordpress.zip

- name: Extract WordPress
  command: "unzip -q /tmp/wordpress.zip -d /opt/"
  args:
    creates: /opt/wordpress
  register: extraction
  changed_when: extraction.rc == 0

- name: Set Wordpress permissions
  file:
    path: /opt/wordpress/
    recurse: yes
    mode: "0755"

- name: Create wp-config.php
  template:
    src: wp-config.php.j2
    dest: /opt/wordpress/wp-config.php
    mode: "0644"
  when: not (ansible_run_tags | default([]) | bool and '/opt/wordpress/wp-config.php' in ansible_run_tags)

- name: Create wordpress.conf
  template:
    src: wordpress.conf.j2
    dest: /etc/httpd/conf.d/wordpress.conf
  notify: Restart httpd service
```

Contenido del archivo provisioning/ubuntu/tasks/wordpress.yml:

```yaml
---
- name: Download WordPress
  command: "curl -o /tmp/wordpress.zip https://wordpress.org/latest.zip"
  args:
    creates: /tmp/wordpress.zip

- name: Extract WordPress
  command: "unzip -q /tmp/wordpress.zip -d /opt/"
  args:
    creates: /opt/wordpress
  register: extraction
  changed_when: extraction.rc == 0

- name: Set WordPress Permissions
  command: "chown -R www-data:www-data /opt/wordpress"
  changed_when: false

- name: Create wp-config.php
  template:
    src: wp-config.php.j2
    dest: /opt/wordpress/wp-config.php
    owner: www-data
    group: www-data
    mode: '0644'
  when: not (ansible_run_tags | default([]) | bool and '/opt/wordpress/wp-config.php' in ansible_run_tags)

- name: Create wordpress.conf
  template:
    src: wordpress.conf.j2
    dest: /etc/apache2/sites-enabled/wordpress.conf
  notify: Restart Apache Service
```

Aquí las diferencias entre ambos sistemas no es tan grande, sólo hayq ue prestar atención a los permisos de los directorios.El servidor Apache necesita poder acceder a los archivos d Wordpress, en el caso de los sistemas basados en Red Hat basta con darle permisos `755` a todas las carpetas de Wordpress para que el servidor Apache pueda acceder a ellos. Mientras que en el caso de los sistemas basados en Debian hay que asignarle la propiedad de esas carpetas al usuario `www-data-www`, de lo contrario Apache no poderá acceder a esos archivos.

Otra diferencia entre ambos sistemas es la ubicación de los archivos de configuración de Apache, en los sistemas basados en Red Hat estos archivos se encuentran en `/etc/httpd/conf.d/`, mientras que en los sistemas basados en Debian estos archivos se encuentran en `/etc/apache2/sites-enabled/`. También es importante destacar que en algún momento vamos a necesitar reiniciar el servicio Apache, hay que recordar que el demonio de Apache se llama diferente en ambos sistemas.

### Tareas post-install de Wordpres

Una vez instalado Wordpress vamos a necesitar hacer unas configuraciones extras para que se muestre el primer blog en lugar del formulario de configuración al momento de abrir el sitio final. Para esto vamos a crear las tareas post-instalación en ambos roles:

```bash
ansible> touch provisioning/centos/tasks/post_install.yml
ansible> touch provisioning/ubuntu/tasks/post_install.yml
```

El contenido para ambos archivos será el siguiente:

```yaml
---
- name: Download and install WP CLI
  get_url:
    url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
    dest: /tmp/wp
    mode: '0755'

- name: Move WP CLI to /bin
  command: mv /tmp/wp /bin/wp

- name: Make WP CLI executable
  file:
    path: /bin/wp
    mode: '0755'

- name: Finish Wordpress installation
  command: >
    sudo -u vagrant -i -- wp core install --path=/opt/wordpress/
    --url=localhost --title="Laboratotio - Wordpress con Ansible - Carlos Colon"
    --admin_user=admin --admin_password="admin@UN1R" --admin_email=carlos@unir.mx
  environment:
    PATH: /bin:/usr/bin:/usr/local/bin
```

Aquí no hay diferencia alguna, el CLI de Wordpress es un ejecutable externo que funciona igual en ambos sistemas.

## Tareas para la instalación y configuración del proxy

Vamos a crear los archivos de tareas para el proxy en cada uno de los roles:

```bash
ansible> touch provisioning/centos/tasks/proxy.yml
ansible> touch provisioning/ubuntu/tasks/proxy.yml
```

Contenido del archivo provisioning/centos/tasks/proxy.yml:

```yaml
---
- name: Install nginx package
  package:
    name: nginx
    state: present

- name: Start and enable nginx service
  systemd:
    name: nginx
    enabled: yes
    state: started

- name: Create nginx configuration
  template:
    src: centos.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx service

- name: Enable httpd_can_network_connect on SELinux
  command: setsebool -P httpd_can_network_connect on

- name: Add port 80 to firewall rules
  command: firewall-cmd --zone=public --add-port=80/tcp --permanent

- name: Reload firewall rules
  command: firewall-cmd --reload
```

Contenido del archivo provisioning/ubuntu/tasks/proxy.yml:

```yaml
---
- name: Install nginx package
  package:
    name: nginx
    state: present

- name: Start and enable nginx service
  systemd:
    name: nginx
    enabled: yes
    state: started

- name: Create nginx configuration
  template:
    src: ubuntu.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx service
```

Nuevamente, la diferencia es el tema de seguridad. Los sistemas basados en Red Hat son sistemas que vienen cerrados por defualt para evitar que por accidente se abran las comunicaciones. Tenemos que ser explícitos en el tema de seguridad y decirle al sistema que permiat conexiones entrantes y slaientes en ciertos puertos.

## Plantillas

Vamos las siguientes plantillas para la configuración del sitio de Wordpress:

- `provisioning/templates/centos.conf.j2`: Configuración del proxy en sistemas CentOS.
- `provisioning/templates/ubuntu.conf.j2`: Configuración del proxy en sistemas Ubuntu.
- `provisioning/templates/wordpress.conf.j2`: Configuración del VirtualHost para Wordpress. Este es un archivo de configuración para Apache.
- `provisioning/templates/wp-config.php.j2`: Configuración inicial de Wordpress.

Veamos ahora cuál será el contenido de estos archivos, empecemos por el contenido de provisioning/templates/centos.conf.j2:

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    upstream backend {
        server {{ wp_ip }}:8080;
    }

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
            # don't cache it 
            proxy_no_cache 1;
            # even if cached, don't try to use it 
            proxy_cache_bypass 1;

            proxy_set_header   Host              $http_host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;

            proxy_pass http://backend/;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
}
```

Contenido del archivo provisioning/templates/ubuntu.conf.j2:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;

    gzip on;

    log_format  custom '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '"$http_x_forwarded_for" $request_id ';

    upstream backend {
        server {{ wp_ip }}:8080;
    }

    server {
        server_name actividad1.unir.mx;
        listen 80;

        error_log   /var/log/proxy_error.log warn;
        access_log  /var/log/proxy_access.log custom;

        server_tokens off;                                           # Don't display Nginx version
        add_header X-XSS-Protection "1; mode=block";                 # Prevent cross-site scripting exploits
        add_header Content-Security-Policy "frame-ancestors 'self'"; # Don't allow be embeded externally
        add_header X-Frame-Options "SAMEORIGIN";                     # Prevents clickjacking attacks by allowing/disallowing the browser to render iframes.

        gzip on;
        gzip_disable "msie6";
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        location / {
            # don't cache it 
            proxy_no_cache 1;
            # even if cached, don't try to use it 
            proxy_cache_bypass 1;

            proxy_set_header   Host              $http_host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;

            proxy_pass http://backend/;
        }
    }
}
```

Contenido del archivo provisioning/templates/wordpress.conf.j2:

```bashs
Listen 8080 http

<VirtualHost *:8080>
    ServerName localhost
    DocumentRoot /opt/wordpress

    ErrorLog /var/log/wordpress_error.log
    CustomLog /var/log/wordpress_access.log combined

    LogLevel debug

    <Directory /opt/wordpress>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Contenido del archivo provisioning/templates/wp-config.php.j2:

```php
<?php

define('DB_NAME', '{{ db_name }}');
define('DB_USER', '{{ db_user }}');
define('DB_PASSWORD', '{{ db_pswd }}');
define('DB_HOST', '{{ db_ip }}');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', 'utf8_general_ci');


/**
 * Authentication unique keys and salts.
 *
 * Llaves aleatorias generadas con:
 * https://api.wordpress.org/secret-key/1.1/salt/
 *
 */
define('AUTH_KEY',         '&P1N0sjb8X[hL6M-k@v@Yob^<yizwyNnr^+rx*%/}7| ~#>$=O5p,B9[v4h/&mr{');
define('SECURE_AUTH_KEY',  'X0$8lRfJ}0,XUX*iLz;,#<Nd_,U)o;zdzNx$ED5p*~!+3`q4Hi1a@v+Bn-T-!_3r');
define('LOGGED_IN_KEY',    'jz;2Qgu)vOaA/?})X`Q}|@fP`ip.?]6@R+d)-{HR=7yF4,|F#2_`e;lQ-lY+Kl?Z');
define('NONCE_KEY',        '[9@q<IZ+B(B|MT)BhMMhosu4yT*C[j4+@AAH?#--v]!(BmSd)=*gHv .sk}SSyy)');
define('AUTH_SALT',        'yW9GqvnYV-s$q=])Da3@XgYV[chJIA:)UL3:U4Pq+Rs)62I/LX6.Rt?~/v.Z/!}z');
define('SECURE_AUTH_SALT', '3{lBp`L{o,vLTHSEC|[3:cxsjAMfZ0EAm^s@BCX!wY^W4KIMsu26j},P+-zS7jP!');
define('LOGGED_IN_SALT',   'GWAsqP-27-MNWf&kNx/]J|.7,y3u+E%__sr;zx@#-d(jDEF-5ZLEq~VhTqB:]H1l');
define('NONCE_SALT',       ';DCi*&xsMf5og74iCqnQg(+Oo|-Z2>IEssn?--y(r0`,=y4 /A-`}55CPhp[Oi~x');

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/documentation/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
        define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';
```
