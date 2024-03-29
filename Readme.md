# Ansible con Google Cloud Platform
https://docs.ansible.com/ansible/latest/scenario_guides/guide_gce.html

En esta guía, utilizando Ansible, vamos a crear, en el cloud de Google, un balanceador de carga con tres máquinas (con Ubunut 20.04), dos de ellas harán de servidores web, en los que instalaremos Apache, PHP y una app web y la tercera será el balanceador, en la que instalaremos Apache, habilitaremos los módulos oportunos y estableceremos la configuración para que lleve a cabo el balanceo de carga.

## Creando infraestructura

### Instalación de los requisitos previos

Instala las librerías necesarias: requests y google-auth:

```
$ pip install requests google-auth
```

### Genera las credenciales con una cuenta de servicio

1.- En la consola web de GCP accede al Menu, IAM y Cuenta de Servicio.
Si se solicita, seleccione el proyecto o cree uno nuevo. Supongamos que el ID del proyecto es cc-2024.

2.- Clic en + Crear cuenta de servicio.
En los detalles de la cuenta de servicio, escriba un nombre, ID y descripción, luego haga clic en Crear y continuar.

3.- En Otorgar privilegios, seleccione las funciones de IAM para otorgar a la cuenta de servicio. Por ejemplo, en Rol seleccione Editor.
Haga clic en Continuar.

4.- Opcional: en Otorgar acceso a los usuarios a esta cuenta de servicio , agregue los usuarios o grupos que pueden usar y administrar la cuenta de servicio.
Haga clic en Listo .

5.- A continuación, cree una clave de cuenta de servicio:

- Haga clic en la dirección de correo electrónico de la cuenta de servicio que creó.
- Haga clic en la pestaña Claves .
- En la lista desplegable Agregar clave, seleccione Crear nueva clave. Elija JSON y Haga clic en Crear.

Esto generará un nuevo par de claves pública/privada que se descarga en su máquina. Supongamos que se guardó con el nombre credentials.json.

Asigna la variable de entorno GCP_SERVICE_ACCOUNT_FILE apuntando al fichero anterior con:

``` 
$ export GCP_SERVICE_ACCOUNT_FILE=<ruta>/credentials.json
```

### Inventario dinámico

Crea el fichero hosts.gcp.yaml:

```yaml
---
plugin: gcp_compute
zones:
  - us-central1-a
projects:
  - cc-2024
auth_kind: serviceaccount
# service_account_file: Variable de entorno GCP_SERVICE_ACCOUNT_FILE
filters:
  - status = RUNNING
groups:
  webservers: "'web' in name"
  loadbalancer: "'lb' in name"
```

Crea el fichero de configuración ansible.cfg:

```
# ansible.cfg

[defaults]
inventory = ./hosts.gcp.yaml

[inventory]
enable_plugins = gcp_compute
```

Comprueba que todo está correcto mostrando información del inventario, si no tienes máquinas virtuales activas debería aparecer algo similar a lo mostrado en el siguiente comando, en caso contrario aparecerá información con las máquinas virtuales que estén activas en el proyecto y región especificadas.

```
$ ansible-inventory --list

{
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "ungrouped"
        ]
    }
}
```
### Playbook para crear IP y MV

Implementa un playbook para crear una máquina virtual y su IP pública.
En este playbook, vamos a utilizar un fichero para las variables confidenciales, usuario y clave ssh para conectar con las MV que no expondremos en el repositorio git (incluyendo en .gitignore la carpeta secret); seguimos utilizando la ubicación de la cuenta de servicio mediante la variable de entorno (GCP_SERVICE_ACCOUNT_FILE) y se necesitarán dos variables extras, 'nombre' para el nombre de la máquina y 'accion' por si queremos crearla (present) o eliminarla (absent).

Vamos a crear primero nuestras variables confidenciales en ./secret/vars.yaml:

```yaml
# Usuario 
usuario: usuario
# Clave pública SSH (id_rsa.pub)
clave_ssh: ssh-rsa AAAA ... usuario@HOST
```

Creemos ahora el playbook:


```yaml
- name: Compute Engine instances
  hosts: localhost
  gather_facts: no  
  vars_files:
    - /secret/vars.yaml
  vars:
      gcp_project: cc-2024
      gcp_cred_kind: serviceaccount
      # gcp_cred_file: definida mediante variable de entorno
      zone: "us-central1-a"
      region: "us-central1"
      machine_type: "e2-micro"
      image: "projects/ubuntu-os-cloud/global/images/family/ubuntu-2004-lts"
      # required extra-vars: nombre and accion. Use ansible-playbook ... --extra-vars "nombre=server accion=present/absent"

  tasks:

  - name: Gestiona IP address para VM
    gcp_compute_address:
      name: "{{ nombre }}-ip"
      state: "{{accion}}"
      region: "{{ region }}"
      project: "{{ gcp_project }}"
      #service_account_file: "{{ gcp_cred_file }}"
      auth_kind: "{{ gcp_cred_kind }}"
    register: gce_ip

  - name: Gestiona instancia de VM
    gcp_compute_instance:
      name: "{{ nombre }}-vm"
      state: "{{accion}}"
      machine_type: "{{ machine_type }}"
      disks:
        - auto_delete: true
          boot: true
          initialize_params:
            source_image: "{{ image }}"
      network_interfaces:
        - access_configs:
            - name: External NAT  # public IP
              nat_ip: "{{ gce_ip }}"
              type: ONE_TO_ONE_NAT
      tags:
        items:
          - http-server
          - https-server
      metadata:
        ssh-keys: "{{usuario}}:{{clave_ssh}}"
      zone: "{{ zone }}"
      project: "{{ gcp_project }}"
      # service_account_file: "{{ gcp_cred_file }}"
      auth_kind: "{{ gcp_cred_kind }}"
    register: gce


  - name: Mensaje recursos eliminados
    ansible.builtin.debug:
      msg:
        - Recursos eliminados {{ nombre }} IP y VM
    when: accion=="absent"

  - name: Mensaje recursos creados (IP y VM)
    ansible.builtin.debug:
      msg:
        - IP {{gce_ip.address}}
        - VM {{gce.name}}
    when: accion=="present"
```

Utilizando el playbook anterior, crearemos tres máquinas virtuales, mediante:

```
$ ansible-playbook vm.yaml --extra-vars "nombre=web1 accion=present"

$ ansible-playbook vm.yaml --extra-vars "nombre=web2 accion=present"

$ ansible-playbook vm.yaml --extra-vars "nombre=lb accion=present"

```
Comprueba como ha quedado el inventario

```
$ ansible-inventory --list
{
    "_meta": {
        ...
    }
...
    "loadbalancer": {
        "hosts": [
            "34.132.158.64"
        ]
    },
    "webservers": {
        "hosts": [
            "34.31.221.147",
            "34.132.184.113"
        ]
    }
}
```
Vamos a crear un playbook que haga ping a los servidores para comprobar que podemos establecer conexión con ellos:

```yaml
# ping.yml
---
- name: Ping all servers
  hosts: all
  tasks:
  - name: Ping a todos los servidores
    action: ping
```
Ejacútalo con:

```
$ ansible-playbook ping.yaml 
```

Si no has incluido aún las máquinas en la fichero .ssh/known_hosts, deberán aparecer tres mensajes que tendremos que confirmar con yes.

```
The authenticity of host '34.132.184.113 (34.132.184.113)' can't be established.
ED25519 key fingerprint is SHA256:UgeT9pOIvK3YCiV4GMzglxahZYCoctkPzSnGEudSoes.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```
La salida del playbook debe ser:

```
$ ansible-playbook ping.yaml 

PLAY [Ping all servers] *****************************************************************************

TASK [Gathering Facts] ******************************************************************************
ok: [34.132.184.113]
ok: [34.132.158.64]
ok: [34.31.221.147]

TASK [Ping a todos los servidores] ******************************************************************
ok: [34.31.221.147]
ok: [34.132.184.113]
ok: [34.132.158.64]

PLAY RECAP ******************************************************************************************
34.132.158.64              : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
34.132.184.113             : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
34.31.221.147              : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Cuando no necesites la infraestructura puedes eliminarla con:

```
$ ansible-playbook vm.yaml --extra-vars "nombre=web1 accion=absent"

$ ansible-playbook vm.yaml --extra-vars "nombre=web2 accion=absent"

$ ansible-playbook vm.yaml --extra-vars "nombre=lb accion=absent"

```

## Instalando servicios

Crea un playbook, en el fichero install-services.yaml, para instalar Apache en el balanceador y Apache y PHP en los webservers.

```yaml
# install-services.yaml

---
- name: Install Apache on loadbalancer
  hosts: loadbalancer
  become: true
  tasks:

  - name: Install the latest version of Apache2
    ansible.builtin.apt:
      name: apache2
      state: present
  - name: Ensure apache starts
    service: name=apache2 state=started enabled=yes

- name: Install Apache and PHP on WebServers
  hosts: webservers
  become: true
  tasks:
  - name: Installing services
    ansible.builtin.apt:
      name:
      - apache2
      - php
      state: present
  - name: Ensure apache starts
    service: name=apache2 state=started enabled=yes

```
Ejecútalo con:

```
$ ansible-playbook install-services.yaml
```

## Despliega la aplicación en los webservers

Obviamente se trata de una aplicación muy simple, en la carpeta app/ vamos a crear el fichero index.php con el siguiente contenido:

```php
<?php

 echo "<h1>Hello, World! This is my WebAPP deployed with Ansible</h1>";
 echo "<h2>" . gethostname() . "</h2>";

?>
```
Crea el playbook para desplegar la aplicación en los webservers:

```yaml
# deploy-app.yaml

---
- name: Upload App and Confgure PHP
  hosts: webservers
  become: true
  tasks:

  - name: Delete Old App
    ansible.builtin.file:
      state: absent
      path: /var/www/html/index.html

  - name: Upload application file
    copy:
      src: app/index.php
      dest: /var/www/html
      mode: 0755

  - name: Configure php.ini file
    lineinfile:
      path: /etc/php/7.4/cli/php.ini
      regexp: ^short_open_tag
      line: 'short_open_tag=Off'
    notify: restart apache

  handlers:
  - name: restart apache
    service: name=apache2 state=restarted

```
Ejecuta:

```
$ ansible-playbook deploy-app.yaml
```

## Configura el Balanceador en el loadbalancer

En la carpeta config/ crea el fichero lb.conf con la configuración del balanceador:

```
ProxyRequests off
<Proxy balancer://webcluster >
  {% for host in hostvars[inventory_hostname]['groups']['webservers'] %}
    BalancerMember http://{{host}}
  {% endfor %}
    ProxySet lbmethod=byrequests
</Proxy>

# Optional
<Location /balancer-manager>
  SetHandler balancer-manager
</Location>

ProxyPass /balancer-manager !
ProxyPass / balancer://webcluster/

```

Crea el playbook para iniciar los módulos y realizar la configuración del balanceador:

```yaml
# setup-lb.yaml
---
- name: Configure loadbalancer
  hosts: loadbalancer
  become: true
  tasks:

  - name: Enable Apache2 Lb Modules
    community.general.apache2_module:
      name: "{{ item.module }}"
      state: present
      warn_mpm_absent: false
      ignore_configcheck: true
    loop:
      - module: proxy
      - module: proxy_http
      - module: proxy_balancer
      - module: lbmethod_byrequests

  - name: Creating template
    ansible.builtin.template:
      src: config/lb.conf
      dest:  /etc/apache2/conf-enabled/lb.conf
      owner: root
      group: root
      mode: 064
    notify: restart apache2

  handlers:
  - name: restart apache2
    ansible.builtin.service:
      name: apache2
      state: restarted

```
Ejecuta:

```
$ ansible-playbook setup-lb.yaml
```

# Resumen de comandos

```
pip install requests google-auth
```

```
ansible-inventory --list
```

```
ansible-playbook vm.yaml --extra-vars "nombre=web1 accion=present"

ansible-playbook vm.yaml --extra-vars "nombre=web2 accion=present"

ansible-playbook vm.yaml --extra-vars "nombre=lb accion=present"

```


```
ansible-playbook ping.yaml
```

```
ansible-playbook install-services.yaml
```

```
ansible-playbook deploy-app.yaml
```

```
ansible-playbook setup-lb.yaml
```

```
ansible-playbook vm.yaml --extra-vars "nombre=web1 accion=absent"

ansible-playbook vm.yaml --extra-vars "nombre=web2 accion=absent"

ansible-playbook vm.yaml --extra-vars "nombre=lb accion=absent"

```