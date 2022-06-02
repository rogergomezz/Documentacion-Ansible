# ANSIBLE

## Comandos Importantes
* ``` -i ``` : Seleccionamos Inventario
* ``` -m ``` : Seleccionamos Módulo
* ``` -a ``` : Seleccionamos Acción "Instalar con name y state, copy, yum,apt ...
* ``` -u ``` : Seleccionamos Usuario
* ``` -c ``` : Seleccionamos el tipo de conexión que queremos realizar, se suele usar para Windows con "WinRM"
* ``` -e ``` : Seleccionamos variables de entorno, viene bien para ignorar el certificado ssl de windows


## Notas
Para usar Windows necesitaremos poner usuario con parametro -u y con -k introducir la clave para SSH
Para solucionar error CERTIFICATE_VERIFY_FAILED tenemos que introducir en el inventario la siguiente variable:  ansible_winrm_server_cert_validation = ignore

## Combinar Inventarios
Importante sobretodo utilizar bien los invetarios para especificar grupos y varibles.

## PlayBooks
Un playbook tiene una lista de tareas a realizar en una lista de servidores especificos. Además, incluye configuraciones y variables entre otros elementos.
Necesitamos poner el comando ansible-playbook

### Esenciales
hosts: Lista de servidores a administrar. Es posible especificar grupos o servidores individuales. El separador es ":" y es posible utilizar los siguientes carácteres típicos de programación como pueden ser el & ! 
* ``` : ``` : Seleccionamos Todo
* ``` & ``` : Seleccionamos que se encuentre en el primer playbook y en el segundo
* ``` -! ``` : Seleccionamos que este en el primer playbook pero NO en el segundo
* ``` remote_user ``` : Seleccionamos Usuario
* ``` become ``` : True o false / 1 o 0 para ser ROOT
* ``` become_user ``` : Seleccionamos usuario para cambiar de ROOT a otro usuario
* ``` become_method``` : Seleccionamos sudo/su/ksum... 
* ``` check_mode ``` : Seleccionamos si podemos realizar los cambios, es decir, si esta en true hace una "simulación" de lo que queremos hacer, para asegurarnos de no destrozar todo.

### Ansible_Playbook
```ansible-playbook opciones fichero.yml```
* ``` -i ``` : Seleccionamos Inventario, fichero,script...
* ``` --syntax-check ``` : Comprobar sintaxis
* ``` --list-task ``` : Listamos las tareas de nuestro playbook
* ``` --step``` : Pregunta paso a paso si hay q continuar
* ``` -start-at-task = tarea ``` : Empieza la tarea especificada
* ``` --forks=n / -f=n ``` : Limita las tareas en paralelo. Por defecto = 5.
* ``` -v vvvv``` : Verbose

### Variables
Una variable contiene un valor modificable para dinamizar las tareas. 

Tipos de variables:
* Facts (Obtenidas del servidor)
* PlayBook 
* Lineas de comando ```-e```

Usadas en:
* Tareas
* Plantillas
* Otros (Condiciones,bucles,etc)

{{variable}} asi se inicializa.

### Sintaxis
La sintaxis para YAML simplifica la definición de playbooks. Ademas ofrece opciones para mejorar la legibilidad.
Cuidado con el TAB mejor ESPACIOS EN BLANCO
* | Texto largo
* ">" Linea larga
* Si queremos hacer una lista de cosas tenemos q ir con cuidado con los tabs y espacios, pero es mejor para leer.
* Modulo DEBUG, permite mostrar un texto o valor de una variable, bastante útil. EJEMPLO: - debug: "un texto" o - debug var=VALOR

### Handlers
Un handler es una tarea que solo se ejecutara en caso de que otra tarea le llame, el uso mas comun es reiniciar un servicio cuando se cambie la configuracion.
EJEMPLO:
``` yml   
tasks:
- name: configurar sshd_config
  copy: src=sshd_config dest=/etc/sshd_config
  notify: reiniciar_sshd
```
Esto hace que si el fichero sshd_config se modifica, llamamos a la funcion reiniciar_sshd que esta previamente enlazada en el notify, para que esto funcione el notify y el name del handler deben ser el mismo.
``` yml   
handlers:
- name: reiniciar_sshd
  service: name=sshd state=restarted
```

OJO!! Los handlers se ejecutan al final del playbook.

### Include y Roles
Es posible dividir un playbook en distintas partes para facilitar la edicion y tratado con INCLUDE especificamos otro fichero que contiene playbook o una tarea.
``` yml   
- name: Primer playbook
  hosts: servweb
  tasks:
   - name: tarea en este fichero
     copy: src=fichero dest=fichero
   - include: otra_tarea.yml
```
DONDE OTRA_TAREA.YML HARÍA LO SIGUIENTE:
``` yml   
- name: Primer playbook
     service: name= nombre state= started 
```

HAY QUE IR CON CUIDADO CON INCLUDE, OBSOLETO HAY QUE USAR IMPORT_PLAYBOOK!!

### Roles
Con los roles podemos crear una estructura de ficheros y directorios para separar los elementos, permite ser reusados facilmente y es posible descargar los roles predefinidos.

IMPORTANTE:
> Orden de los roles:
>>Roles/
>>>Nombre/
>>>>Files/
>>>>>Templates/
>>>>>>Tasks/main.yml

DENTRO DEL PLAYBOOK:
```yml
roles: 
    -rol1
    -rol2
#PASAR VARIABLES
roles:
    - {role: rol1, clave: valor}
```

### Templates
Dentro de una plantilla "TEMPLATE" se pueden especificar las siguientes instrucciones:
```yml
* Expresiones: {{ variable }} -> {{ansible_fqdn}}
* Control: {{%...%}} -> 
    Condicion: 
        {% if ansible_distribution== "Debian" %}
        {%endif%}
    Bucles: {%...%} ->
        {% for usuario in lista_usuarios %}
        usuario
        {%endfor%}
* Comentarios: {#Comentario#}
```

### Prioridad de las Variables
El orden de menor a mayor prioridad de las variables es el siguiente:
* "Defaults" definidas en un rol
* Variables de grupo (Inventario -> group_vars/all -> group_vars/grupo)
* Variables de servidor(Inventario -> host_vars/servidor)
* "Facts" del servidor -> {{ansible_hostname}}
* Variables dentro del play
* Variables dentro del rol (Definidas en /roles/rol/vars/main.yml)
* Variables del bloque -> Variables de tareas
* Parametros de rol:
    ```yml
        role: apache2, lista_usuarios: ["usuario1","root1"]
    ```
* Variables set_facts / registered_vars
* extra vars (CLI SIEMPRE GANAN AL RESTO)

### Condiciones
Es posible condicionar la ejecución de una tarea, la inclusión de un fichero o el uso de un rol usando la expresión when.

Ejemplo:

  ```yml
        name: instalar apache2
        apt: name: apache2 state: latest
        when: ansible_distribution == "Windows"
  ```
Esto hará que si no cumple la distribución Windows, no instalará apache2, simplemente "skippeará".

También a partir del parámetro "include" para añadir otro fichero de configuración.
  ```yml
        name: instalar apache2
        include: instalar_apache2.yml
        when: ansible_distribution == "Windows"
  ```

Por último también podemos incluir roles a partir de una condicion.

  ```yml
        roles:
          - {role: apache2,
             when: ansible_distribution == "Debian"}
  ```

### Bucles
Hemos visto como reccorer una lista dentro de una plantilla, ahora vamos a recorrerlas dentro de una tarea. Para listas y diccionarios, utilizaremos la expresión with_items.

Ejemplo:
  ```yml
      - name: Instalar software necesario
        apt: name= {{item}} state= latest
        with_items:
         - mariadb
         - php5
         - phpmyadmin
  ```

También podemos crear usuarios:
  ```yml
      - name: Instalar software necesario
        user: name= {{item nombre}} state=present groups={{item grupo}}
        with_items:
         - {nombre: usuario1, grupo: www-data}
         - {nombre: adamuz, grupo: www-data}
  ```

También, es posible especificar una variable:
  ```yml
    "{{varible}}"
  ```

### Register
La expresión register nos permite guardar en una variable el resultado de la acción realizada por un módulo de la tarea.

Ejemplo:
  ```yml
      - name: Ejecutar comando
        command: uptime #Para ver la salida necesitamos utilizar -vv
        register: salida #Salida es el nombre de la variable

      - name: Mostrar salida
        debug: var=salida
  ```
Valores que puede devolver:
* Changed
* Failed
* Skipped
* rc
* stdout / stderr / stdout_lines / stderr_lines

Es posible utilizar:
  ```yml
  - when: salida.changed 
  ```

### Ignore Errors
La expresión ignore_errors permite ignorar una tarea marcada como error y continuará con el resto de tareas.

Ejemplo:
```yml
- name: Comporobar si el fichero existe
  command: ls /noexiste.txt
  register: existe
  ignore_errors: true
```
En un playbook hay que tener cuidado porque ignorara todas las salidas de error dentro del playbook.

Es posible utilizar la condición: not existed.failed para realizar una tarea dependiente.

### Failed When
Las expresiones como failed_when y changed_when permiten especificar las condiciones para marcar una tarea como fallida o cambiada.

En el caso de un comando será marcado como error si el "return code(rc)" es distinto a 0.

Ejemplo:
```yml 
#Este ejemplo es para aprender a como funciona failed_when, esto hara que si no tenemos eth2 de error.
- name: Ejecutar comando
  command: ip a
  register: salida
  failed_when: "eth2" not in salida.stdout
```

```yml 
#Este ejemplo es para aprender a como funciona failed_when, esto hara que si no tenemos eth2 de error.
- name: No marcar nunca como cambiado
  command: uptime
  changed_when: false
```

## Modulos

### Modulos
Ansible está compuesto de una múltitud de modulos. Éstos pueden administrar sistemas, dispositivos, usuarios y mucho más.

Cada tarea en un playbook está asociada a un módulo. Los argumentos pueden ser obligatorios u opcionales, además suelen tener un valor por defecto. Se separan en las siguientes categoriás:
* Cloud
* Clustering
* Commands
* Crypto
* Database
* Files
* Identity
* Inventory
* Messaging
* Monitoring
* Network
* Notification
* Packaging
* Remote management
* Source control
* Storage
* System
* Utilities
* Web infrastructures
* Windows

Podemos utilizar ansible-doc para listar (-l) u obtener información y ejemplos.

### Modulo ficheros Open SSL
Los modulos de tratado de fichero permite trabajar con ficheros, plantillas y/o directorios

Listado:

* acl: Establece y obtiene informacion de listas de control de acceso (ACLs)
* archive: Crea un fichero comprimido a partir de una lista de ficheros o estructura de directorios
* assemble: Asambla un fichero de configuración desde fragmentos.
* blockinfile: Inserta/actualiza/elimina un bloque de texto de un fichero
  * Obligatorios:
    * block="texto"
    * dest=/directorio/fichero
  * Opcionales:
    * owner=usuario
    * group=grupo
    * mode=modo
    * backup=yes/no
    * insertafter=expresion
    * insertbefore=expresion
    * marker=expresion
    * state=present/absent
  ```yml
  #Asegurar que bloque de texto este en fichero
  - name: Configurar sshd_config
    blockinfile: 
      path: /etc/ssh/sshd_config
      block: |
        match user monitor
          Password Authentication no
  ```
* copy: Copia ficheros a ubicaciones remotas, si usamos copy guardaremos el fichero debemos guardarlo en el directorio "files/"
  * Obligatorio:
    * dest=/directorio/fichero
  * Opcionales:
    * backup=yes/no
    * content="contenido"
    * force=yes/no
    * owner=usuario
    * group=grupo
    * mode=modo
    * src=/directorio/fichero
  ```yml
  tasks:
    - name: Crear fichero con contenido especifico
      copy: content="Esto es un ejemplo" dest=/home/vagrant/prueba backup=yes
  ```
* fetch: Obtiene ficheros a partir de un nodo remoto
  * Obligatorio:
    * src=/directorio/fichero <!--- De nodo a ansible-->
    * dest=/directorio/fichero 
  * Opcionales:
    * fail_on_missing=yes/no
    * flat=yes/no
  ```yml
  - name: Copiar configuracion red
    fetch:
      src: /etc/network/interfaces
      dest: /backup/configs
      flat: yes #Copiara directamente el fichero sin copiar toda la ruta 
  ```
* file: Establece atributos a ficheros
  * Obligatorio:
    * path=/directorio/fichero
  * Opcionales:
    * backup=yes/no
    * force=yes/no
    * owner=usuario
    * group=grupo
    * mode=modo
    * state:
      * file
      * link
      * directory
      * hard
      * touch
      * absent
  ```yml
  - name: Directorio exista
    file:
      path: "/var/log/ssh"
      state: directory
      owner: root
      group: ssh
      mode: 2755
    notify: reiniciar_ssh
  handlers:
  - name: reiniciar_ssh
    service: name=systemctl-ssh state=restart
  ```
* find: Devuelve una lista de ficheros a partir de un patron dado (busquedas)
* ini_file: Gestiona valores de un fichero tipo .INI
* iso_extract: Extrae ficheros de una imagen ISO
* lineinfile: Asegura que una linea esta en un fichero o reemplaza una linea usando expresiones regulares
  * Obligatorios:
    * line="texto"
    * dest=/directorio/fichero
  * Opcionales:
    * owner=usuario
    * group=grupo
    * mode=modo
    * backup=yes/no
    * insertafter=expresion
    * insertbefore=expresion
    * regexp=expresion
    * state=present/absent
  ```yml
  #Deshabilitar SELinux
  - linefile: path="etc/selinux/config" regexp='^SELINUX=' line='SELINUX=disabled' #Con regexp buscamos la línea a modificar.
  #Borrar línea en fichero
  - linefile: path=/etc/sudoers state=absent regexp='^%wheel'
  #Añadir linea en un fichero
  - linefile: path=/etc/apache2/ports.conf regexp='^listen' insertafter="Listen 80" line="Listen 8080"
  ```
* patch: Aplica parches utilizando GNU PATCH
* replace: Reemplaza las coincidencias de un texto especificado por otro indicado
* stat: Obtiene informacion de fichero o sistema de ficheros`
  * Obligatorio:
    * path=/directorio/fichero
  * Opcionales:
    * get_attributes=true/false
    * get_checksum=true/false
    * get_md5=true/false
    * get_mime=true/false
  ```yml
  - stat: path=/etc/services
    register: datos
    debug: datos #Para mostrar la informacion que contiene el registro datos.
    debug: msg="Es directorio?" #Otro ejemplo para saber si el contenido es un directorio.
    when: datos.stat.isdir
  ```
* synchronize: Modulo para sincronizar utilizando rsync
* tempfile: Crear ficheros o directorios temporales
* template: Copia y procesa una plantilla a un nodo remoto "Si utilizamos template, debemos guardarlo en el directorio "templates/"
  * Obligatorios:
    * dest=/directorio/fichero
    * src=/directorio/fichero
  * Opcionales:
    * backup=yes/no
    * force=yes/no
    * owner=usuario
    * group=grupo
    * mode=modo
  ```yml
    - name: Copiar fichero y establecer propietario
      template: src=apache2.conf.j2 dest=/etc/apache2/apache2.conf owner=www-data group=www-data
  ```
* unarchive: Extrae un fichero (Opcionalmente despues de copiarlo)
  * Obligatorios:
    * src=/directorio/fichero
    * dest=/directorio/fichero
  * Opcionales:
    * owner=usuario
    * group=grupo
    * mode=modo
    * remote_src=yes/no
    * list_files=yes/no
  ```yml
    - name: Copiar fichero y establecer propietario
      unarchive: src=oracle.tgz dest=/opt/oracle
      #Extraer un fichero que ya existe en el nodo remoto
      unarchive: src=oracle.tgz dest=/opt/oracle remote_src=true
  ```
* xatr: Establece/Obtiene atributos extendidos

Los modulos para OpenSSL son los siguientes:
* openssl_privatekey: Generar claves privadas de OpenSSLç
  * Obligatorios:
    * path=/directorio/fichero
  * Opcionales:
    * force=true/false
    * size=4096
    * state=present/absent
    * type= RSA/DSA
  ```yml
    - name: Instala módulo de Python requerido
      name: Instalar python-pyOpenSSL
      apt:
        name: python-openssl
        state: latest
    - name: Generar clave privada
      openssl_privatekey:
        path: /etc/ssl/public/oforte.net.pem #Ejemplo de ruta donde estaria la clave pública.
        privatekey_path: /etc/ssl/private/oforte.net.pem #Ejemplo de ruta donde estaria la clave privada.
  ```
* openssl_publickey: Generar claves públicas de OpenSSL
  * Obligatorios:
    * path=/directorio/fichero
  * Opcionales:
    * force=true/false
    * size=4096
    * state=present/absent
    * type= RSA/DSA
  ```yml
    - name: Instala módulo de Python requerido
      name: Instalar python-pyOpenSSL
      apt:
        name: python-openssl
        state: latest
    - name: Generar clave privada
      openssl_privatekey:
        path: /etc/ssl/public/oforte.net.pem #Ejemplo de ruta donde estaria la clave pública.
        privatekey_path: /etc/ssl/private/oforte.net.pem #Ejemplo de ruta donde estaria la clave privada.
  ```

### Modulos de gestor de paquetes
Gestor de paquetes para lenguajes de programación
* bower: Administra paquetes para desarrollo web
* bundler: Administra dependencias de Ruby Gem
* composer: Gestiona librerías PHP
* cpanm: Gestiona módulos perl (cpan)
  * Opciones:
    * from_path=ruta
    * name=nombre
    * locallib=ruta
    * mirror=mirror
    * mirror_only=yes/no
    * notest=yes/no
    * version=version
    * system_lib=directorio
  ```yml
    - name: Instalar primero cpanm
      apt: name=perl_app_cpanminus state=latest

    - name: Instalar modulo DBI
      cpanm: name=DBI

    - name: Instalar version especifica
      cpanm:
        name: DBI
        version: "1.360"
  ```
* easy_install: Gestiona módulos / librerías Python
  * Opciones:
    * Requeridas:
      * name=nombre
    * Opcionales:
      * state=present/latest
      * virtualenv=yes/no
      * virtualenv_command=comando
      * virtualenv_site_packages=yes/no
      * executable=ruta_ejecucion
  ```yml
  - name: Instalar PIP
    easy_install:
      name: pip
      state: latest
    #Utilizaremos esto para instalar pip, debido a que es mucho mas potente que easy_install
  ```
* gem: Gestiona Ruby Gems 
* maren_artifact: Descarga Artifacts desde un repositorio Maven 
* npm: Gestiona paquetes node.js con npm 
* pear: Gestiona paquetes pear / pcl 
* pip: Gestiona librerías / módulos Python
  * Opciones:
    * name=nombre
    * state=present/latest/absent/forcereinstall
    * virtualenv=yes/no
    * virtualenv_command=comando
    * virtualenv_site_packages=yes/no
    * executable=ruta_ejecucion
    * requirements=fichero.txt
    * version=version
    * chdir=ruta
  ```yml
  - name: Instalar requests
    pip:
      name: request
      executable: pip3
      state: latest

  - name: Instalar requisitos
    pip: requirements=/miapp/requirements.txt
  ```
Gestor de paquetes para sistemas operativos:
* apk: Gestiona paquetes Android
* apt: Gestiona paquetes APT
  * Opciones:
    * name=nombre[=version]
    * state=present/latest/absent/build-dep
    * upgrade=no/yes/safe/full/dist
    * force=yes/no
    * update_cache=yes/no
    * purge=yes/no
    * deb=/ruta/fichero.deb
    * autoremove=yes/no
    * default_release=release
  ```yml
  - name: Instalar nginx
    apt:
      name: nginx
      state: latest

  - name: Actualizat lista de paquetes
    apt: update_cache=yes

  - name: Actualiza todos los paquetes
    apt: upgrade=dist
  ```
* apt_key:
  * Opciones:
    * data=clave
    * file=fichero
    * id=identificador
    * keyring=/ruta/trusted.d/
    * keyserver=servidor
    * state=present/absent
    * url=direccion
    * validate_certs=yes/no
  ```yml
  - name: Añadir clave usando servidor
    apt_key:
      keyserver: keyserver.ubuntu.com
      id: 36A1D7869245C8950F...

  - name: Añadir utilizando un fichero adjunto
    apt_key: update_cache=yes

  - url: "https://ftp.master.debian.org/keys/archive-key-6.0.asc"
    state: present
  ```
* apt_repository: 
  * Requerido:
    * repo=origen
  * Opcionales:
    * state=present/absent
    * filename=nombre
    * update_cache=yes/no
    * validate_certs=yes/no
    * mode=modo
  ```yml
  - name: Añadir repositorio google-chrome
    apt_repository:
      repo: deb http://dl.google.com/linux/chrome/deb/ stable main
      state: present
      filename: "google-chrome"

  - name: Añadir repositorio Ubuntu
    apt_repository:
      repo: "ppa:nginx/stable"
  ```
  
* dnf: Gestiona paquetes Fedora
* macports: Gestiona paquetes MacPorts "OSX"
* Openbsd_pky: Gestiona paquetes OpenBSP
* Opkg: Gestiona paquetes OpenWRT
* Package: Gestor de paquetes genéricos (wrapper)
  * Requerido:
    * name=origen
    * state=present/absent/latest
  * Opcionales:
    * use=auto/yum/apt
  ```yml
  - name: Instala ultima version de ntpdate
    package:
      name: ntpdate
      state: latest
  ```
* Pacman: Gestor de paquetes Arch Linux
* Pkg 5: Gestiona paquetes Solaris 11
* pkgin: Gestiona paquetes SmartOS, NetBSD y otros
* Pkgng: Gestiona paquetes FreeBSD >= 9.0
* Portage: Gestiona paquetes Gentoo
* Redhat_subscription: Administra registro y subscripciones usando el comando subscription-manager
  * Opciones:
    * state=present/absent
    * activationkey=clave
    * autosubscribe=yes/no
    * org_id=origen/accion
    * pool=nombre
    * username=usuario
    * password=clave
    * server_hostname=servidor
    * force_register=yes/no
  ```yml
  - name: Registrar sistema
    redhat_subscription:
      state: present
      username: usuario@mail
      password: clave
      autosubscribe: yes
  ```
* Slackpkg: Gestiona paquetes Slackware >= 12.2
* Swdepot: Gestiona paquetes HP-UX 
* yum: Gestiona paquetes YUM
  * Requerido:
    * name=nombre
  * Opcionales:
    * state=present/absent/latest
    * conf_file=ruta
    * disable_gpg_check=true/fale
    * disablerepo=nombre
    * enablerepo=nombre
    * update_cache=yes/no
  ```yml
  - name: Instala ultima version de apache
    yum:
      name: httpd
      state: latest
  
  - name: Actualizar todos los paquetes
    yum:
      name: "*"
      state: latest

  - name: Instalar grupo
    yum:
      name: "@Development Tools"
      state: present
  ```

* yum_repository:
  * Requerido:
    * name=nombre
  * Opcionales:
    * state=present/absent
    * description=description
    * baseurl=direccion
    * file=nombre_fichero
    * microlist=direccion
    * enabled=yes/no
    * gpycheck=yes/no
  ```yml
  - name: Añadir EPEL
    yum_repository:
      name: httpd
      state: latest
      description: EPEL YUM Repo
      baseurl: http://download.fedoraproject.org/pub/epel...
  ```
* Zypper / Zypper_repository: Gestiona paquetes / repositorios SUSE / OpenSUSE

### Modulo Comando Utilidades

La siguiente lista permite ejecutar comandos en el nodo remoto.

* command
  * Opciones:
    * chdir=/directorio
    * creates=/fichero/comprobar
    * executable=/ruta/binario
    * removes=/fichero/comprobar
  ```yml
  - name: Obtener uname
    command: uname -a
    register: salida_uname
  
  - name: Crear base de datos si no existe
    command: /sbin/createdb.sh
    args:
      chdir: /var/lib/mysql
      creates: /basededatos/existe
  ```
* expect: Ejecuta un comando y responde a la introduccion de datos
  * Opciones:
    * Requeridas:
      * command=comando
      * responses=respuestas
    * Opcionales:
        * chdir=/directorio
        * creates=/fichero/comprobar
        * removes=/fichero/comprobar
        * echo=true/false
        * timeout=30
  ```yml
  - name: Instalar pexpect
    yum: name=pexpect state=latest
  
  - name: Cambiar contraseña usuario
    expect: 
      command: passwd usuario
      responses:
        (?i)password: "SuperSecreta$"
  ```
* raw: Envia comandos sin filtrar por SSH
  * Opciones:
    * executable=/ruta/ejecutable
  ```yml
  - name: Copia y ejecuta el script
    script: /mi/fichero/local.sh argumentos/parametros
  ```
* script: Transfiere y ejecuta un script indicado
  * creates=/fichero/comprobar
  * removes=/fichero/comprobar  
  * decrypt=true/false
* shell: Muy usado, utilizaremos /bin/bash
  * chdir=/directorio
  * creates=/fichero/comprobar
  * executable=/ruta/binario
  * removes=/fichero/comprobar

  ```yml
  - name: Obtener uname
    shell: uname -a | tee fichero.log
    register: salida_uname

  - name: Obtener uname
    shell: uname -a | tee fichero.log
    register: salida_uname
    args:
      executable: /bin/bash
      chdir: /tmp
  ```

* assert: Asegura que se cumplan ciertas condiciones.
  * Opciones:
    * Requerido
      * that=Condicion
    * Opcional:
      * msg="Mensaje a mostrar"
  ```yml
  - assert: {that:"ansible_os_family != "RedHat"}
  #Tambien se puede realizar de la siguiente manera:
  - assert:
      that:
        - "numero <=100"
        - "numero >=0"
      msg: "Numero debe estar entre 0 y 100"
  ```
### Modulos Utilidades

* debug: Muestra un texto personalizado o el valor de una variable.
  * Opciones:
    * msg="Mensaje a mostrar"
    * var=variable
    * verbosity=1
  ```yml
  - debug:
      msg: "Nombre {{ansible_hostname}};{{ansible_fqdn}}"

  - shell: uptime
    register: resultado

  - debug: var=resultado
  ```
* fail: Falla con un mensaje especifico
  * Opciones:
    * msg="Mensaje de texto"
  ```yml
  - fail: msg="Dato Incorrecto"
    when: valor not in ["x","y"]
  ```
* include: Incluye playbooks o tareas
* include_role: Incluye un rol
* include_vars: Incluye variables desde un fichero
  * Opciones:
    * name=nombre(obligatorio)
    * private=true/false
    * tasks_from=main
    * vars_from=main
    * defaults_from=main
    * allow_duplicates=true/false
  * Estructura de directorios
    * Tasks/main.yml:
      * Configure.yml
      * Install.yml
      * Post-install.yml
  ```yml
    - hosts: all
      tasks:
        - debug: msg="Playbook"
        - include_role:
            name: mirol
        - include_vars: variables.yml
    - include: otroplaybook.yml
  ```
* pause: Pausa la ejecución de un playbook
  * Opciones:
    * minutos=minutos
    * prompt="texto a mostrar"
    * seconds=segundos
  ```yml
  - pause:
  - pause: minutes=2
  - pause: prompt="Compruebe acceso aplicación"
  ```
* set_fact: Establece un fact
  * Sintaxis:
    * set_fact: clave=valor
    * set_fact:
      * clave1: valor1
      * clave2: valor2
  ```yml
  - set_fact:
    nombre: ansible_hostname
  - debug: var=nombre
  ```
* wait_for: Espera que se cumpla una condicion para continuar.
  * Opciones:
    * state=present/started/stopped/absent
    * port=puerto
    * delay=segundos
    * host=servidor
    * exclude_hosts=servidores
    * connection_timeout=segundos
    * path=/ruta/fichero
    * search_regex=patron
    * timeout=segundos
  ```yml
  - waitfor:
      port: 8080
      delay: 10
  - waitfor:
      path: /tmp/running
  - waitfor:
      port: 22
      host: "{{ansible_hostname}}"
      search_regex: OpenSSH
      delay: 10
  ```
### Modulos Notificaciones
Estos modulos permiten notificar de distintas maneras (chat,sms,mail...)
* cisco_spark: Envía un mensaje a un canal o usuario en Cisco Spark
* flowdock: Envia un mensaje a flowdock
* hipchat: Envia notificacion al popular Hitchat
  * Opciones:
    * Requeridos:
      * token=codigo
      * msg=mensaje
      * room=nombre o id del canal
    * Opcionales:
      * api="http://api.hipchat.com/v1"
      * color=yellow
      * from=Ansible
      * msg_format=text/html
      * notify=yes/no
      * validate_certs=yes/no
  ```yml
  - hipchat:
      token: codigo largo
      room: oforte
      msg: "notificacion desde ansible"
  ```
* irc: Mensaje a un canal IRC
* jabber: Mensaje a usuario o canal en Jabber
* mail: Mandar email
  * Opciones:
    * Requeridos:
      * subject="Asunto Correo"
    * Opcionales:
      * host=servidor
      * port=puerto
      * username=usuario
      * password=clave
      * to=direccion@correo.com
      * body=texto
      * cc=direccion
      * bcc=direccion
      * secure=always/never/try/startTLS
  ```yml
  - mail:
      host: correo.oforte.net
      port: 25
      subject: ansible-informe
      to: adamuz@plexus.es
      from: roger@plexus.es
      body: "Sistema {{ ansible_fqdn }} funcionando con éxito"
  ```
* mattermost: Otro chat tipo slack en codigo abierto
* mqt: Mensajeria para IoT
* nexmo: Envia SMS
* pushbullet: Notificaciones a dispositivos
  * Opciones:
    * Requeridos:
      * api_key=clave
      * title=titulo
    * Opcionales:
      * body=message
      * channel=canal
      * device=dispositivo
      * push_type=note/link
  ```yml
  - name: Instalar pushbulley.py
    pip: name=pushbullet.py state=present
  - name: Enviar notificacion
    pushbullet:
      api_key: clave
      device: dispositivo
      title: "Notificacion Ansible"
  ```
* pushover: Notificaciones a moviles
  * Opciones:
    * Requeridos:
      * app_token=clave
      * user_key=clave
      * msg="Mensaje"
    * Opcionales:
      * pri=prioridad
  ```yml
  - name: Usar pushover
    pushover:
      app_token: clave_api
      user_key: clave_usuario
      msg: "Mensaje desde Ansible"
  ```
* rocketchat: Notificacion al rocker.chat
  * Opciones:
    * Requeridos:
      * token=clave
      * domain=servidor:puerto
    * Opcionales:
      * msg=mensaje
      * channel=#canal
      * username=Ansible
      * color= normal/good/warning/danger
      * protocol=https/http
      * validate_certs=true/false
  ```yml
  - name: Usar rocket
    rockerchat:
      token: clave
      domain: cloud.oforte.com:8080
      msg: "Mensaje desde Ansible"
      channel: "#Oforte"
  ```
* sendgrid: Envia mails usando la API de Sendgrid
* slack: Notificaciones a Slack
  * Opciones:
    * Requeridos:
      * token=clave
    * Opcionales:
      * msg=mensaje
      * channel=#canal
      * username=Ansible
      * color= normal/good/warning/danger
      * validate_certs=true/false
  ```yml
  - name: Usar slack
    slack:
      token: clave
      msg: "Mensaje desde Ansible"
      channel: "#Oforte"
  ```
* Sns: Envia mensajes usando Amazon Simple Notification Service (SNS)
* Telegram: Envia mensajes usando telegram
* Twilio: Envia mensajes a telefonos moviles

### Modulos Bases de Datos
Para empezar, empezaremos con el mas popular, mysql.
* mysql tiene varios modulos:
  * mysql_db: Añade o elimina bases de datos
    * Opciones:
      * Requeridos:
        * name=nombrebd
      * Opcionales:
        * state=present/absent/dump/import
        * login_host=localhost
        * login_password=clave
        * login_port=3306
        * login_user=root
        * login_unix_socket=
        * encoding=codificacion
        * collation=colacion
        * target=destino
  ```yml
  - name: Instalar libreria requerida
    pip: name=python-mysql state=latest
  
  - name: Crear si no existe la base de datos
    mysql_db:
      name: plexus
      state: present
  
  - name: Copia de seguridad de todas las bbdd
    mysql_db:
      state: dump
      name: all
      target: /tmp/{{ansible_hostname}}.sql
  ```
  * mysql_replication: Administra replicacion
  * mysql_user: Añade o elimina usuarios
      * Opciones:
      * Requeridos:
        * name=nombrebd
      * Opcionales:
        * state=present/absent
        * password=clave
        * encrypted=yes/no
        * login_host=localhost
        * login_password=clave
        * login_port=3306
        * login_user=root
        * login_unix_socket=
        * priv=db tabla:priv1,priv2
        * append_privs=yes/no
  ```yml
  - name: Instalar libreria requerida
    pip: name=python-mysql state=latest
  
  - name: Crear usuario y darle permisos
    mysql_user:
      name: plexus
      password: plexus123
      state: present
      priv: "oforte.*:ALL"
  ```
  * mysql_variables: Administra variables globales

* Postgresql:
  * postgresql_db: Añade o elimina bases de datos
      * Opciones:
      * Requeridos:
        * name=nombrebd
      * Opcionales:
        * state=present/absent
        * login_host=localhost
        * login_password=clave
        * port=3306
        * login_user=root
        * login_unix_socket=
        * encoding=codificacion
        * collation=colacion
        * template=plantilla
  ```yml
  - name: Instalar libreria requerida
    pip: name=psycopg2 state=latest
  
  - name: Crear si no existe la base de datos
    postgresql_db:
      name: plexus
      state: present
      encoding: UTF-8
  
  - name: Copia de seguridad de todas las bbdd
    mysql_db:
      state: dump
      name: all
      target: /tmp/{{ansible_hostname}}.sql
  ```
  * postgresql_ext: Administra extensiones
  * postgresql_lang: Administra procedimientos almacenados
  * postgresql_privs: Administra privilegios
  * postgresql_schema: Añade o elimina esquemas
  * postgresql_user: Añade o elimina usuarios

* MongoDB:
  * mongodb_parameter: Gestiona parámetros
  * mongodb_user: Añade o elimina usuarios

* InfluxDB:
  * influxdb_database: Administra bases de datos
  * retention_policy: Administra politicas de retencion

* Vertica (HPE):
  * vertica_configuration: Gestiona configuración
  * vertica_facts: Obtiene información
  * vertica_role: Gestiona roles
  * vertica_schema: Gestiona esquemas
  * vertica_user: Añade o elimina usuarios

* Misc:
  * elasticsearch_plugin: Administra plugins
  * kibana_plugin: Administra plugins
  * redis: Comandos slave y flush
  * riak: Gestiona algunas operaciones comunes

### Modulos Gestionar Sistema

* Alternatives: Gestionar alternativas para comando
  * Opciones:
    * Requerido:
      * name=nombre
      * path=/ruta/fichero
    * Opcionales:
      * link=/ruta/fichero
      * priority=50
  ```yml
  - name: Fijar versio de Java
    alternatives:
      name: java
      path: /usr/bin/jum/java-8-openjdk-amd64/bin/java
  ```
* at: Programar ejecucion de comandos
* authorized_keys: Gestiona el fichero de SSH para las claves publicas autorizadas
  * Opciones:
    * Requerido:
      * user=usuario
      * key=clave_ssh
    * Opcionales:
      * state=present/absent
      * path=.../.ssh/authorized_keys
      * manage_dir=yes/no
      * key_options=opciones
      * exclusive=yes/no
  ```yml
  - name: Autorizar clave publica
    authorized_key:
      user: alberto
      state: present
      key: "sjksdfbhkbkdgbfd@flndgkdfb"
  ```
* cron: Gestiona tareas programadas
  * Opciones:
    * name=nombre
    * job=comando
    * state=present/absent
    * minute=0-59, */5
    * hour=0-24,*/2
    * special_time=reboot/yearly/annualy/monthly/weekly/daily/hourly
    * weekday=0-6
    * month=1-12,*2
    * day=1-31,*2
    * cron_file=nombre
    * backup=yes/no
  ```yml
  - name: Crear programacion
    cron:
      name: "copia de seguridad"
      minute: 0
      hour: 2
      job: /root/scripts/backup.sh
  ```
* crypttab: Cifrar dispositivos
* filesystem: Gestionar sistemas de ficheros
  * Opciones:
    * Requerido:
      * dev=dispositivo
      * fstype=sistemaficheros
    * Opcionales:
      * force=yes/no
      * opts=opciones_
      * resizefs=yes/no
  ```yml
  - name: Crear sistema de ficheros
    filesystem:
      dev: /dev/vdb1
      fstype: xfs
  ```
* firewalld: Gestionar firewall
  * Opciones:
    * Requerido:
      * state=enabled/disabled
      * permanent=true/false
    * Opcionales:
      * service=servicio
      * zone=zona
      * port=puerto/protocolo
      * source=origen
      * rich_rule="regla avanzada"
      * immediate="true/false"
  ```yml
  - name: Permitir acceso http/https
    firewalld:
      state: present
      service: "{{item}}"
      permanent: true 
  #Interesante hacer un handler que ejecute firewall-cmd --reload!!
  ```
* gluster_volume: Gestionar volumenes de GlusterFS
* group: Administrar grupos
  * Opciones
    * Requerido:
      * name=nombre
    * Opcionales:
      * state=present/absent
      * gid=idGrupo
      * system=yes/no
  ```yml
  - name: Crear grupo para aplicacion
    group:
      name: plexus
      state: present
      gid: 185
  ```
* hostname: Administar nombre de servidor
  * Opciones:
    * Requerido:
      * name=nombre
  ```yml
  - name: Cambiar nombre del servidor
    hostname:
      name: servidor.dominio.com
  ```
* iptables: reglas de iptables
  * Opciones:
    * state=present/absent
    * chain=INPUT/FORWARD/OUTPUT/PREROUTING...
    * source=direccion
    * jump=ACCEPT/DROP/...
    * table=filter/nat/mangle/raw/security
    * in_interface=interfaz
    * out_interface=interfaz
    * protocol=tcp/udp/icmp
    * destination_port=puerto
    * to_ports=puerto
    * cstate=INVALID/NEW/ESTABLISHED/RELATED/...
  ```yml
  - name: Permitir acceso al puerto 80
    iptables:
      chain: INPUT
      source: 0.0.0.0
      destination-port: 80
      jump: ACCEPT
      protocol: tcp
  ```
* known_hosts: Añadir o eliminar registros
* lvg: Grupos de volumenes
  * Opciones:
    * Requerido:
      * vg=grupovolumenes
    * Opcionales:
      * state=present/absent
      * pvs=/dev/vd1
      * pesize=4
      * vg_options=opciones_vgcreate
      * force=yes/no
  ```yml
  - name: Crear grupo de volumenes
    lvg:
      vg: datavg
      pvs: /dev/vda1
      state: present
  ```
* lvol: Configurar volumenes logicos
  * Opciones:
    * Requerido:
      * vg=grupovolumenes
      * lv=volumenLogico
    * Opcionales:
      * state=present/absent
      * size=tamaño
      * pvs=/dev/vd1
      * opts=opciones
      * active=yes/no
      * force=yes/no
  ```yml
  - name: Crear volumen logico
    lvol:
      vg: datavg
      lv: web
      size: 2G
      state: present
  ```
* mount: Montar sistema de ficheros
  * Opciones:
    * Requerido:
      * name=/ruta/dir
      * state=present/absent/mounted/unmounted
    * Opcionales:
      * fstype=tipo
      * opts=opciones
      * src=dispositivo/origen
      * dump=0
      * passno=0
  ```yml
  - name: Montar volumen lógico
    mount:
      name: /var/www
      state: mounted
      src: /dev/mapper/datavg-web
      fstype: xfs
  ```
* open_iscs: Gestionar targets iscsi desde open_iscsi
* openwrt_init: Gestionar servicios Wrt
* pam_limits: Gestiona limites PAM
* pamd: Gestiona modulos PAM
* ping: Comprobar conexion
```yml
- name: Comprobar conexion
  ping:
```
* seboolean,sefcontext,selinux,selinux_permissive,seport: Gestionar SELinux
* service: Gestiona servicios del sistema
  * Opciones:
    * Requerido:
      * name=servicio
    * Opcionales:
      * state=started/stopped/restarted/reloaded
      * enabled=yes/no
      * arguments=argumentos
      * sleep=segundos
  ```yml
  - name: Iniciar y habilitar servicio
    service:
      name: apache2
      state: started
      enabled: true
  ```
* setup: Obtiene información del sistema
  * Opciones:
    * fact_path=/etc/ansible/fact.d
    * filter=*
    * gather_subset=all/hardware/network/virtual
    * gather_timeout=10
  ```yml
  - name: Obtiene facts
    setup:
      gather_subset: all
  ```
* systemctl: Configura el fichero /etc/systemctl.conf
  * Opciones:
    * Requerido:
      * name=nombre
    * Opcionales:
      * value=valor
      * state=present/absent
      * reload=yes/no
      * sysctl_file=/etc/sysctl.conf
      * sysctl_set=yes/no
      * ignoreerrors=yes/no
  ```yml
  - name: Permitir redirigir el tráfico
    sysctl:
      name: net.ipv4.ipforward
      value: 1
      sysctl_set: yes
      state: present
      reload: yes
  ```
* systemd: Gestiona servicios en systemD
  * Opciones:
    * name=nombre
    * state=started/stopped/restarted/reloaded
    * enabled=yes/no
    * daemon_reload=yes/no
    * masked=yes/no
  ```yml
  - name: Habilitar servicio y recargar systemD
    systemd:
      name: apache2
      enabled: yes
      state: started
      daemon_reload: yes
  ```
* timezone: Administra la zona horaria
  * Opciones:
    * Requeridas:
      hwclock: true/false
      name=area/ciudad
  ```yml
  - name: Definir uso horario
    timezone:
      name: Europa/Madrid
  ```
* user: Gestiona usuarios
  * Opciones:
    * Requerido:
      * name=nombre
    * Opcionales:
      * state=present/absent
      * group=grupo
      * groups=grupo
      * append=yes/no
      * createhome=yes/no
      * uid=idusuario
      * home=directorio
      * shell=/bin/bash
      * password=clave
      * remove=yes/no
      * system=yes/no
  ```yml
  - name: Crear usuario Plexus
    user:
      name: Plexus
      uid: 185
      home: /opt/Plexus
      shell: /bin/false
      system: yes
  ```
### Modulos para Windows
* win_acl
* win_chocolatey
* win_command
```yml
# Lanzar comando y guardarlo en variable para mostrarlo en pantalla
- name: Modulos para Windows
  hosts: Windows
  tasks:
    - win_command: whoami
      register: usuario
    - name: devolucion quien soy
      debug: var=usuario
```
* win_copy
```yml
# Copiar fichero
- name: Copiar fichero
  hosts: Windows
  tasks:
    - win_copy: 
        src: /etc/apache2/apache2.conf
        dest: c:\Users\NomUsuario\Documentos\apache2.conf
```
* win_environment
* win_feature
* win_file
```yml
# Crear o eliminar ficheros o directorios
- name: Crear estructura de directorios
  hosts: Windows
  tasks:
    - win_file:
        path: c:\directorio\subdirectorio
        state: directory
```
* win_get_url
* win_group
* win_lineinfile
```yml
# Cambiar puerto que escucha apache2 de 80 a 8000
- name: Editar puerto
  hosts: Windows
  tasks:
    - win_lineinfile:
        path: c:\Apache2\apache2.conf
        state: present
        line: Listen 8000
        regexp: "^Listen"
```
* win_msi
* win_package
* win_ping
* win_reboot
* win_regedit
* win_schedule_task
* win_service
```yml
- name: Reiniciar servicio
  hosts: Windows
  tasks:
    - win_service:
        name: spooler
        state: sttoped
        start_mode: manual
```
* win_share
* win_shell
```yml
- name: Ejecutar script
  win_shell: c:\scripts.ps1
    args:
      chdir: c:\
```
* win_stat
* win_template
```yml
- name: Copiar plantilla
  win_template:
    src: info.j2
    dest: c:\info.txt
```
* win_timezone
* win_unzip
* win_updtes
* win_uri
* win_user
```yml
- name: Crear usuario
  win_user:
    name: plexus
    password: plexus123
    state: present
    groups:
      - users
```

### Modulos Control Versiones
* bzr: Obtiene ficheros desde bzr
* git: Obtiene ficheros desde git
```yml
- name: Obtener ejemplos
  git:
    repo: https://github.com/rogergomezz/ansibledoc
    dest: /root/ansible-examples
```
* git_config: Lee y escribe configuracion git
```yml
- name: Usar vim como editor
  git_config:
    name: core.editor
    value: vim
    scope: global
```
* github_hooks: Administra hooks de Github
* github_keys: Administra claves de Github
* github_release: Administra publicaciones de Github
* gitlab_group: Crea/Actualiza/Elimina grupos de Gitlab
* gitlab_project: Crea/Actualiza/Elimina proyectos de Gitlab
* gitlab_user: Crea/Actualiza/Elimina usuarios de Gitlab
* hg: Obtiene ficheros desde mercurial
```yml
- name: Obtener codigo
  hg:
    repo: "https://www.selenic.com/repo/hello-world"
    dest: /root/hg/
```
* subversion: Obtiene ficheros desde subversion
```yml
- name: Obtener codigo
  subversion:
    repo: "https://develop.svn.wordpress.org/trunk/src/"
    dest: /root/wordpress/
```
### Modulos Intra Web
* apache2_mod_proxy: Establece o obtiene atributos de miembros de apache2.4
* apache2_module: Habilita o deshabilita modulos de Apache2(y otros), deben ser Debian o Ubuntu
```yml
- name: Habilitar modulo wsgi
  apache2_module:
    name: wsgi
    state: present
```
* deploy_helper: Administra algunos pasos para desplegar aplicacion
* django_manage: Gestiona aplicaciones Django
* ejabberd_user: Gestiona usuarios de Ejabber
* htpasswd: Administra usuarios para autentifacion basica
```yml
- name: Instalar libreria passlib
  apt: name=python_passlib state=present
- name: Añadir usuario
  htpasswd:
    name: oforte
    path: /var/www/html/.htpasswd
    password: prueba123
```
* jboss: Despliega aplicaciones JBoss
* jeakins_job: Administra trabajos de Jenkins
* jeakins_plugin: Administra plugins de Jeakins
* jira: Añade o modifica incidencias en Jira
* letsencrypt: Crea certificados SSL usando LetsEncrypt
* supervisorctl: Administra el estado de un programa o grupo usando supervisord
```yml
- name: Detener aplicacion
  supervisorctl:
    name: long_script
    state: stopped
```
* taiga_issue: Crea o elimina incidencias en la plataforma tagging

### Modulos Cloud
* Amazon
  * Cloudformation
  * Cloudtrail
  * DynamoDB
  * ec2
  * elasticache
  * iam
  * route53
  * s3
  * sns/sqs/sts
* Atomic
  * Host
  * Image
* Azure
  * Network
    * Interfaces
    * Public IP
    * Subnets
    * Virtual Networks
  * Resource groups
  * Security groups
  * Storage accounts
  * Virtual machines
```yml
# IMPORTANTE
- name: Crear maquina con Azure
  pip: name=azure version="2.0.0rc5"
  azure_rm_virtualmachine:
    resource_group: Plexus
    name: UserPlexus1
    vm_size: Basic_A0
    admin_username: Roger
    admin_password: RogerAdmin123$
    ssh_public_keys:
      - path: /home/roger/.ssh/authorized_keys
        key_data: ssh-rsa sdfjfhfsdgw423423424254+fsdfsdf%%///
    image:
      offer: Windows     
```
* Centurylink
  * Alerts
  * Firewall
  * Load Balancer
  * Servers
  * Public IP
* Cloudstack
  * Account
  * Firewall
  * Host
  * Network
  * Roles
  * Routes
* Digital Ocean
  * Block Storage
  * Domain
  * SSH Key
  * Droplets
  * Tags
* Docker
  * Contenedor
  * Imagenes
  * Network
  * Servicios
```yml
- name: Conectar docker
  pip: name=docker-py state=latest
- name: Descargar imagen
  docker_image:
    name: centos
    state: present
- name: Ejecutar contenedor
  docker_container:
    name: micontenedor
    state: started
    detach: True
    interactive: True
    image: centos
```
* Google
  * Instancias
  * Storage
  * DNS
  * Load Balancer
  * Redes / Firewall
  * Tags
  * Backend Service
```yml
# Crear maquina virtual de google en europa, tamaño pequeño, con SO Debian, que este presente, y luego unos datos de autenticacion.
- name: Crear maquina Google
  gce:
    instance_names: ansible_plexus
    zone: europe-west1-b
    machine_type: g1-small
    image: debian-8
    state: present
    service_account_email: "roger@plexus.iam.gserviceaccount.com"
    credentials_file: "plexus-passwords.json"
    project_id: "proyectoPlexus-123"
```
* Openstack
  * Auth
  * Flavor
  * Image
  * Network
  * Nova
  * Project
  * Instances
  * Volumes
  * Zone
* VMWare
  * VCA
  * Cluster
  * Datacenter
  * vSwitchs
  * VM
  * vSAN
  * Datastore

### Modulos Virtualizacion
* datadog_event: Enviar eventos a DataDog
* datadog_monitor: Administra monitorizacion DataDog
* logicmonitor: Administra la cuenta de LogicMonitor
* logicmonitor_facts: Gestiona los facts de LogicMonitor
* monit: Gestiona estado ed servicios con Monit
* nagios: Realiza tareas comunes en Nagios
* newrelic_deployment: Notifica sobre despliegue de aplicaciones en Newrelic
* pagerduty: Crear ventanas de mantenimiento en PagerDuty
* pagerduty_alert: Gestiona alertas en PagerDuty
* sensu_check / sensu_subscription: Gestiona Sensu
* zabbix_group: Crea / Elimina grupo de hosts
* zabbix_host: Crea / Actualiza / Elimina hosts
* zabbix_hostmacro: Gestiona macros
* zabbix_maintenance: Crea ventanas de mante
* zabbix_screen: Gestiona pantallas

## Galaxy

### Ansible Galaxy Introduccion
Ansible Galaxy es un repositorio en linea, gratuito, donde se alojan roles a ser utilizados por nuestros playbooks.

### Ansible Galaxy
El comando ansible-galaxy permite realizar diversas tareas sobre los roles.

* Sintaxis:
```bash
ansible-galaxy [accion] [opciones] argumentos
```
* Acciones:
  * delete: Elimina un rol de la web de Galaxy
  * import: Importa un rol desde Github a Galaxy
  * info: Muestra informacion detallada de un rol
  * init: Inicializa estructura directorio
  * install: Descarga un rol
  * list: Lista los roles instalados
  * login: Autentifica en la web de Galaxy
  * remove: Elimina rol del servidor pero NO de la web de Galaxy
  * search: Hace una busqueda dentro del repo
  * setup: Crea una integracion con Travis CI

* Opciones:
  * -f / --force: Sobreescribe el rol
  * -i / --ignore-errors: Ignora errores
  * -n / --no-deps: No instala dependecias
  * -p / --roles-path: Directorio de roles "cambiar"
  * -r / --roles-file: Fichero con lista de roles a instalar

### Compartir Rol
Los pasos para compartir un rol en Galaxy son los siguientes:
1. Crear repositorio en GitHub
2. Inicializar el rol con ansible-galaxy (Opcional pero RECOMENDADO)
3. Editar meta/main.yml para especificar autor, descripcion, plataformas y etiquetas
4. Publicar nuestro código (commit && push) a GitHub
5. Iniciar sesión con ansible-galaxy login
6. Importar rol: ansible-galaxy import github_usuario github_repositorio

Para eliminar un rol de Galaxy:
* ansible-galaxy delete github_usuario github_repositorio

### Fichero Roles
Es posible crear un fichero con el listado de roles a instalar. Los atributos que se pueden especificar (uno o mas) son los siguientes:
* src: Origen del rol
* scm: Especifica el tipo de SCM -> git o hg
* version: Especifica un nombre distinto al original
* name: Especifica un nombre distinto al original

ansible-galaxy -r fichero.yml instalará todos los requerimientos.

## Tower

### Tower Introduccion
Ansible tower es una solucion web para facilitar las tareas a todos los equipos IT.
Los componentes son los siguientes:
* Proyectos (Repositorio de playbooks) -> Local o SCM (git)
* Inventarios (Estaticos o dinamicos)
* Plantillas de trabajo (Definicion de playbook,inventario y diversas opciones)
* Trabajos (Ejecucion de las plantillas)
* Configuraciones (Credenciales,equipos,organizaciones,etc...)

Ansible tower posee tres ediciones:
* self_support: Sin soporte ni caracteristicas especiales (LDAP,System Tracking o cuestionarios)
* standard: Con soporte 8x5 pero sin caracteristicas especiales
* premium: Soporte 24x7 y caracteristicas especiales.

Es gratuita la edicion self_support para 10 nodos o menos.

Es posible solicitar una licencia de prueba de un mes para premium.

### Tower Instalacion
Requisitos:
* Servidor especifico para tower (No compartir con otras aplicaciones)
* Distribuciones soportadas: { Red Hat Enterprise Linux 7, CentOS 7, Ubuntu 14.04/16.04 }
* Navegador Chrome o Firefox. Otros pueden funcionar pero no estan soportados.
* Al menos 2GB de RAM (Recomendados 4GB)
* Al menos 20 GB de disco, de los cuales 10GB deben ser dedicados a /var/
* En el caso de RHEL/CentOS es necesario habilitar el repo EPEL

Escenarios:
* Un servidor con aplicacion y base de datos en el mismo servidor
* Un servidor para la aplicacion y otro con instalacion de la base de datos PostgreSQL
* Alta disponibilidad: Varios servidores con tower instalado con un servidor separado actuando como base de datos

Pasos:

* En RHEL/CentOS instalar EPEL
* En ubuntu: apt install software-properties-common && apt-add-repository ppa:ansible/ansible

Metodos de instalacion:
* Instalacion normal: https://releases.ansible.com/ansible-tower/setup
* Bundle: https://releases.ansible.com/ansible-tower/setup_bundle

* Una vez descargado descomprimimos con ```tar -xvf ansible-tower-setup-[bundle]-latest.tar.gz``` y editamos el fichero inventory con los servidores y contraseñas.
* Acceder a la web a partir de http://direccionip
* Tambien es necesario acceder al puerto 8080 para los eventos en tiempo real.

### Instalar Ansible AWX
Instalar version Ansible AWX 17.1.0 sobre un Ubuntu 20.04 con 4GB RAM 2cpu. 
* Dependecias:
  * Docker
  * Docker-compose
  * Git

Url del repo: https://github.com/ansible/awx/archive/refs/tags/17.1.0.tar.gz

```
$ cd installer

$ ansible-playbook -i inventory install.yml
```
### Tower Elementos
Los elementos principales al trabajar con tower son los siguientes:
* Proyectos: Define donde los playbooks estan almacenados
* Inventarios: Lista de servidores con los que trabajar

### Tower Configuration
Una vez instalado Ansible Tower es posible configurar los siguientes elementos:
* Organizaciones: Agrupar contenido para administrar permisos de los siguientes departamentos
* Usuarios: Permitir a usuarios el uso de tower
* Equipos: Dividir una organizacion para asociar contenido y controlar permisos
* Credenciales: Añadir contraseñas, claves, SSH... Para acceder a servidores, inventarios o proyectos
* Trabajos de gestión: Gestionar trabajos administrativos
* Scripts de inventario: En el caso de no tener integracion nativa a su inventario, es posible definir un script personalizado para obtener los servidores
* Notificaciones: Crear plantillas para notificar el resultado de un trabajo
* Licencia: Vre o modificar licencia
* Configurar tower: Desde la autenticacion hasta el login

### Tower Proyectos
Un proyecto hace referencia a donde esta alojado los playbooks y todos los ficheros necesarios, pueden estar alojados en:
* Local
* SCM
En caso de necesitar autentificarse, existen dos maneras de credenciales:
* Clave SSH
* Usuario/Contraseña

### Tower Inventarios
Un inventario contiene una lista de servidores (con sus atributos). Es posible crear grupos y pueden ser dinamicos.

### Tower Credenciales
Las credenciales permiten acceder a los distintos componentes. Los tipos son:
* Maquina: Acceder a servidores {Clave SSH, Usuario/Clave, sudo}
* Red: Acceder a dispositivos de red {Clave SSH, Usuario/Clave}
* Control de versiones: Accede a SCM {Clave SSH, Usuario/Clave}
* Proveedor: Para acceder al inventario {Usuario/Clave, Datos de acceso}

### Tower Plantillas
Una plantilla contiene la definicion de todos los elementos para ejecutar un playbook en los servidores del inventario especificado. Se define:
* Nombre
* Descripcion
* Tipo de trabajo {Ejecutar,Comprobar,Escanear(sin playbook)}
* Inventario
* Proyecto
* Playbook
* Credencial
* Límite (Filtro de servidores)
* Verbosity
* Tags
* Opciones {Escalar privilegios,trabajos concurrentes, callback}
* Labels
* Variables extra


### Ansible Debug
El modulo debug nos ayuda a visualizar el contenido de variables o expresiones de las plantillas
```yml
- debug:
    msg: "Cadena de texto a interpretar"
- debug: var=nombreVariable
```

### Ansible Tags
Las etiquetas (tags) nos permite especificar selectivamente que tareas se ejecutaran o se omitiran
```yml
- modulo: acciones
  tags: etiqueta1
```

### Ansible Lookup
La directiva lookup nos permite obtener datos desde el servidor que se esta ejecutando el playbook
```yml
"{{lookup("file","/etc/motd")}}"
```

### Ansible Vault
El comando ansible-vault permite cifrar ficheros con una contraseña para proteger datos sensibles. La sintaxis es la siguiente:
```yml
ansible-vault create fichero.yml # Crea un fichero
ansible-vault edit fichero.yml # Edita un fichero
ansible-vault encrypt fichero.yml # Cifra un fichero
ansible-vault decrypt fichero.yml # Descifra un fichero
ansible-vault view fichero.yml # Ver un fichero
ansible-playbook --ask-vault-pass or --vault-password-file #Pregunta o introduces fichero para cifrar
```
### Ansible Tareas Asincronas
Cuando realizamos una tarea en un servidor a  traves de un modulo, la conexion permanece abierta esperando su finalizacion. Ansible nos da la opcion de realizarla en segundo plano y consultar el estado periodicamente.
```yml
#Metodo 1
- modulo: arguments
  async: Tiempomax
  poll: Tiempo cons
#Metodo 2
- modulo: arguments
  async: Tiempomax
  poll: 0
  register: tarea_async
- debug: msg="Otra Tarea" 
- async_stuts: jid={{tarea_async.ansible_job_id}}
  register: estado
  until: estado.finished
  retries: 30
```