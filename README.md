# AnsibleBootcampJav
Curso de Udemy de Ansible Bootcamp

## Primeros pasos
- Definir fichero de configuracion por cada capitulo, en el definimos localizacion del inventario
```
cd chap4
vi ansible.cfg
[defaults]
remote_user = devops
inventory = environments/prod
retry_files_save_path = /tmp
host_key_checking = False
log_path=~/ansible.log
```

- Definir inventario estatico
```
mkdir environments
vi prod
[local]
localhost ansible_connection=local

[app]
app1
app2

[lb]
lb

[db]
db

[prod:children]
lb
app
db
```

### Prueba de conectividad
```
ansible all -m ping
```

- Filtrar por hosts: ejemplos
  - Filtrar por hosts / grupos: `ansible 'app1:app2:lb' -m ping`
  - Excluir hosts / grupos: `ansible '!db:!lb'`
  - Expresiones regulares: `ansible '~(app|db).*' -m ping`


### 2do filtro: **limit**
```
ansible all -m ping --limit 'lb:app
```

### Usar comandos ad hoc, **No son idempotentes como los modulos, no abusar**
```
ansible all -a 'uname -a'
```

### Definir paralelizaciones o forks
```
ansible all -f 1 -a "free"
```

## Modulos
- Representa un recurso: package, , mongodb, sshd ... etc
- Es analogo a resources de puppet
- Cada recurso tiene un modulo, hay cientos
- https://galaxy.ansible.com/
- Listar modulos: `ansible-doc --list`

### Documentacion
Ver opciones de uso del modulo:
- Por linea de comandos: `ansible-doc <modulo>` ejemplo: `ansible-doc user`
- Short description: `ansible-doc -s user`
- Documentacion online: [docs.ansible.com](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

### Uso
Vamos a instalar el paquete vim en la maquina del loadbalancer. Para ello leemos las opciones del modulo yum: `ansible-doc yum`
- m yum -> modulo yum
- -b -> become root (sudo)
- -a -> arguments
```
ansible lb -m yum -b -a "name=vim state=present"
```

### Ad Hoc approach. Command modules
- raw
  - para cuando no esta python instalado en el server destino
  - ejecuta comandos ssh raw
- command
  - ejecuta comando remotamente sin una shell
  - no tiene acceso a env
  - no se pueden usar operadores: `| < > ...`
  - `ansible app -m command -a 'free'`
- shell
  - ejecuta comando remotamente con /bin/sh
  - `ansible app -m shell -a 'free | grep -i swap'`
- script
  - copia script a nodo remoto y tambien ejecuta
- expect
  - ejecuta comando interactivo y automatiza respuestas

#### Estos modulos no son idempotentes por defecto
Pero hay una opcion para hacerlos idempotentes, definiendo un fichero que especifica lo que se ha de crear con **creates**
Aunque en nuestro script no creemos un fichero, lo podemos usar como *flag file*
```
ansible app -m command -a "mkdir /tmp/dir1 creates=/tmp/dir1"
```

## Playbooks
Definiendo infraestructura como codigo.
Diferencia importante con puppet: Aqui los pasos que definamos van en orden secuencial

### Anatomia de un playbook
- name # nombre del play 1
- hosts # donde correr
- become # sudo
- vars # variables
- tasks # lista de tareas. Que hacer
  - name: nombre de tarea # opcional. Recomended for self-document
    <modulo>: <argumentos>
- name # nombre del play 2
- ...
```ansible
---
- name: App server configurations
    hosts: app
    become: true
    become_user: admin
    become_method: sudo
    vars:
      apache_port     = 8080
      max_connections = 4000
      ntp_conf        = /etc/ntp.conf
    tasks:
      - name: create app user
        user: 
          name: app 
          state: present 
          uid: 5002
 
      - name: install git
        yum: 
          name: tree 
          state: present
```

### Opciones extra de ansible-playbook
- Veremos todas las opciones con: `ansible-playbook --help | less`
- Validar sintaxis: `ansible-playbook systems.yml --syntax-check`
- check. Tipo noop, ves lo que haria pero no aplica cambios: `ansible-playbook systems.yml --check`
- Imprimir tasks: `ansible-playbook systems.yml --list-tasks`
- Imprimir hosts sobre los que correria el playbook: `ansible-playbook systems.yml --list-hosts`
- Ejecutar a partir de task especifica: `ansible-playbook systems.yml --start-at-task=START_TASK`
- Retry failed steps: `ansible-playbook systems.yml --limit @/tmp/systems.retry`
- Step by step execution `ansible-playbook systems.yml --step`
- Limitar scope a hosts sin importar lo que ponga en el campo hosts: `ansible-playbook systems.yml --limit app`

## Roles

### Ansible Galaxy
Repositorio de roles. Podemos usar roles sin tener que escribir uno.
Además viene con una utilidad para generar la estructura (carpetas) de un rol: **ansible-galaxy init**
- ayuda: `ansible-galaxy --help`
```
ansible-galaxy init --offline --init-path=roles apache
- apache was created successfull
```
```
tree
.
`-- roles
    `-- apache
        |-- README.md
        |-- defaults
        |   `-- main.yml
        |-- files
        |-- handlers
        |   `-- main.yml
        |-- meta
        |   `-- main.yml
        |-- tasks
        |   `-- main.yml
        |-- templates
        |-- tests
        |   |-- inventory
        |   `-- test.yml
        `-- vars
            `-- main.yml
```
Dentro de tasks/main.yml incluimos todas las tasks:
```
cat tasks/main.yml
--- tasks file for apache
  - import_tasks: install.yml
  - import_tasks: service.yml
```

### Aplicar roles desde un playbook
Para aplicar la configuración de un rol, debemos hacerlo apuntando desde un playbook a ese rol. 
Usamos para ello el keyword **roles** Ejemplo:
```
cat app.yml
  - hosts: app
    become: true
    roles:
      - apache
```

### Handlers
Cuando queremos notificar que un cambio debe notificarse para realizar alguna acción en consecuencia, lo hacemos de forma separada en los handlers. Ejemplo: cuando el fichero de configuración httpd cambie, queremos que se reinicie el servicio apache para tomar los cambios.
- handler:
```
cat handlers/main.yml
---
# handlers file for apache
  - name: restart apache service
    service:
      name: httpd
      state: restarted
```
- desde la task referenciamos el handler:
```
cat tasks/config.yml
---
  - name: copy apache config
    copy:
      src: httpd.conf
      dest: /etc/httpd.conf
      owner: root
      group: root
      mode: 0644
  
    # IMPORTANTE: el argumento del notify debe coincidir con el string de la descripcion del handler
    notify: restart apache service
```
- también podemos referenciar handlers de otros roles diferentes al que estamos


### Dependencias: meta
Podemos definir que un rol depende de otro en **meta/main.yml** en la sección dependencies:
Por ejemplo, queremos que el rol apache tenga como dependencia el rol systems, lo añadimos al array:
dependencies[systems]

### VARS

#### Different vars
- user defined
- facts:
  - las podemos descubrir con: `ansible app1 -m setup`

#### Var precedence
- e switch
- role vars
- playbook vars
- host vars: específicas de un host
- group vars: agrupamos por entorno por ejemplo
- role defaults: ejemplo: para apache por defecto puerto 80

Por defecto, las vars de hashes son sobreescritas, no mergeadas. Podemos cambiar esto cambiando la configuracion:
```
cat ansible.cfg
hash_behaviour = merge
```
#### Podemos definir vars en diferentes niveles
##### Role defaults
```
root@control:/workspace/AnsibleBootcampJav/chap7# cat roles/frontend/defaults/main.yml
---
# defaults file for frontend
  app:
    version: 1.5
    env: LOCALDEV

  fav:
    color: white
    fruit: orange
    car: chevy
    laptop: toshiba

  dbconn:
    host: localhost
    username: root
    password: changeme
    dbname: devopsdemo
```

##### Group vars
Podemos definir variables para cada entorno:
```
root@control:/workspace/AnsibleBootcampJav/chap7/environments# ls -l
total 4
-rw-r--r--. 1 root root 105 Jun 19 10:59 prod
root@control:/workspace/AnsibleBootcampJav/chap7# pwd
/workspace/AnsibleBootcampJav/chap7
root@control:/workspace/AnsibleBootcampJav/chap7# mkdir -p group_vars/prod.yml
root@control:/workspace/AnsibleBootcampJav/chap7# cat group_vars/prod.yml
---
  fav:
    color: yellow
    fruit: banana
```

##### A nivel de playbook
```
---
  - hosts: app
    become: true
    vars:
      fav:
        fruit: mango
```

#### Registered variables
Podemos registrar el retorno de algo para tomar una decisión en base al output
Ejemplo:
```
- name: register variable example
  hosts: local
  tasks:
    - name: run a shell command and register result
      shell: "/sbin/ifconfig eth1"
      register: result
    - name: print registered variable
      debug: var=result
```

#### Fact variables
Con el módulo setup podemos consultar los facts de todos o un host:
(Podemos filtrar por inventario para sacar los hosts que nos interesen)
- Obtener todos los facts: `ansible all -m setup`
- Filtrar por un fact: `ansible all -m setup -a "filter=ansible_os_family"`
- Filtrar por un patron: `ansible all -m setup -a "filter=ansible_mem*"`
- Obtener todos los facts en formato json (los almacena en el directorio pasado como argumento a --tree y losordena por host): `ansible all -m setup --tree /tmp/facts`

## Galaxy
- Web: https://galaxy.ansible.com
- Ejemplo de instalacion de rol de galaxy: ansible-galaxy install <username.role>: `ansible-galaxy install geerlingguy.docker`
- Por defecto se descargan en **~/.ansible/roles** ya que hay una variable de entorno que lo define. La podemos ver con: `ansible-config dump | grep -i role`
- Aunque podemos editar ansible.cfg para dejarlos donde queramos, cada path separado por ':' : `roles_path = roles:galaxy-roles`
- Para personalizar la configuracion de un role, podemos hacer un override de las variables que están como default en el role, 
  una buena práctica es dejarlas en group_vars:
```
root@control:/workspace/AnsibleBootcampJav# cat chap8/group_vars/prod.yml
---
  fav:
    color: yellow
    fruit: banana

  haproxy_backend_servers:
    - name: app1
      address: 192.168.61.12:80
    - name: app2
      address: 192.168.61.13:80
  haproxy_backend_httpchk: ''

  mysql_databases:
    - name: devopsdemo
  mysql_users:
    - name: devops
      password: devops
      priv: "devopsdemo.*:ALL"
  mysql_root_password: root
```  
## Tags
Podemos etiquetar nuestras tasks con tags específicos, para llamar solo porciones de roles por ejemplo:
Taggeamos:
```
vi chap8/roles/apache/tasks/config.yml
tags:
      - apache
      - config
```
Se puede etiquetar:
- A nivel de task
- A nivel de include_task
- A nivel de rol (todas las tasks de ese rol lo heredan)

Lo llamamos de forma selectiva:
```
ansible-playbook site.yml --tags=app,config
```

También podemos mostrar todas las tags de un playbook:
```
root@control:/workspace/AnsibleBootcampJav/chap8# ansible-playbook site.yml --list-tags
 [WARNING]: Found both group and host with same name: db

 [WARNING]: Found both group and host with same name: lb


playbook: site.yml

  play #1 (app): app server playbook    TAGS: []
      TASK TAGS: [apache, config]
```
