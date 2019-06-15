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

