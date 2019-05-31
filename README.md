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

- Prueba de conectividad
```
ansible all -m ping
```

- Filtrar por hosts: ejemplos
  - Filtrar por hosts / grupos: `ansible 'app1:app2:lb' -m ping`
  - Excluir hosts / grupos: `ansible '!db:!lb'`
  - Expresiones regulares: `ansible '~(app|db).*' -m ping`


- 2do filtro: **limit**
```
ansible all -m ping --limit 'lb:app
```

- Usar comandos ad hoc, **No son idempotentes como los modulos, no abusar**
```
ansible all -a 'uname -a'
```

- Definir paralelizaciones o forks
```
ansible all -f 1 -a "free"
```

## M√dulos
- Representa un recurso: package, , mongodb, sshd ... etc
- Es an√logo a resources de puppet
- Cada recurso tiene un modulo, hay cientos
- https://galaxy.ansible.com/
- Listar m√dulos: `ansible-doc --list`

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


