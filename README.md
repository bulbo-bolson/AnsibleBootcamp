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
  - Expresiones regulares: `ansible '~(app|db).*' -m ping


- 2Âdo filtro: **limit**
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
