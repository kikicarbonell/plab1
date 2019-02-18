
# Hands on Lab

**Tema:** Introducción a Puppet

**Descripción:** Laboratorio práctico para guiar primeros pasos con el ecosistema de trabajo Puppet y Git.


**Pre-condiciones**

- Conexión a Internet en los nodos(puertos: 80, 443, 22)

- Configurar host con acceso ssh a los nodos:

	1. Descargar la key de conexión ssh 
	2. Crear alias de conexión ssh en el perfil del usuario: 
	Ej: `nano ~/.ssh/config`
	 ```
	 Host pmaster
	   HostName <master-ip>
	   User centos
	   IdentityFile ~/.ssh/<host-key.pem>

	Host pslave
	   HostName <slave-ip>
	   User centos
	   IdentityFile ~/.ssh/<host-key.pem>
    ```
    Verificar conexión: 
    `ssh pmaster`
    `ssh pslave-aws`
    
	 3. Registrar alias al `/etc/hosts` para resolución local de nombres 
	 ```
	 <master-ip> pmaster-aws
 	 <slave-ip>  pslave-aws
  ```

## EJERCICIOS

### Ejercicio 1: Hello World Puppet!
El ejercicio guía la construcción de un módulo *nix-based de Puppet que notifica mensaje durante la ejecución y generar un file en directorio `/tmp/` cuyo contenido será el mensaje "Hello World!...I'm a puppet follower". 

Conocerá la estructura básica de un módulo y su ubicación en el directorio relativo de Puppet.

P1: **[master]** ubicarse al directorio: `/etc/puppetlabs/code/environments/production/modules` y listar el contenido actual del directorio. 
En este directorio se muestran los módulos que se han instalados desde **Forge** y los que se crean manualmente.

P2: **[master]**  Crear la estructura de archivos en el directorio actual (modules):
```
helloworld/ (nombre del módulo)

-   manifests/
    -   init.pp  (manifest que contiene la clase `helloworld`)
    -   motd.pp  (manifest que contiene el file de recursos que garantiza la creación del file temporal con el mensaje de saludo)
```
P3: **[master]** Edite `init.pp` con el contenido:
```
class  helloworld {
  notify { 'hello, world!': }
}
```
P4: **[master]** Edite `motd.pp` con el contenido:
```
class  helloworld::motd { 
  file { '/etc/motd': 
  owner => 'root', 
  group => 'root', 
  mode => '0644', 
  content => "Hello, world!, I'm a puppet follower\n", } }
```
P5: **[master]** En este punto ya ha completado la creación de un nuevo módulo en Puppet. Se necesita incluirlo en el **manifest** principal para que se distribuya la acción sobre los nodos esclavos. Para lograrlo, cambie al directorio:
`/etc/puppetlabs/code/environments/production/manifests/` y edite el file `site.pp` con el contenido siguiente:
```
node  default {
  class { 'helloworld': }
  class { 'helloworld::motd': }
}
```
P6: **[master]**  Valide la configuración con el comando `puppet parser validate site.pp`

P7: **[master]** Si el resultado fue satisfactorio puede planificar una ejecución del procedimiento implementado sobre los nodos: `puppet agent -t` Con un resultado similar al siguiente:
```
[root@pslave ~]# puppet agent -t  
Info: Retrieving pluginfacts 
Info: Retrieving plugin 
Info: Loading facts 
Info: Caching catalog for agent1.example.com 
Info: Applying configuration version '1437172035'  
Notice: hello, world! 
Notice: /Stage[main]/Main/Node[default]/Notify[hello, world!]/message:  defined  'message' as 'hello, world! I'm a puppet fallower'  Notice: Applied catalog in  1.25 seconds
```
Puede verificar que en el directorio temporal existe un file `cat /etc/motd` con el contenido esperado.

P8: **[slave]** aplique los pasos del **P7** y puede verificar que en un agente diferente se produce el mismo resultado.

### Ejercicio 2: Instalación de paquetes
El ejercicio muestra como instalar un paquete especifico empleando los recursos de Puppet y se verifica adicionalmente mediante los **facts** reportados por los agentes en la interfaz web.

**Pre-condiciones**:  Módulo a instalar:

 - `puppet module install puppetlabs-java`

P1: **[master]** Ubicarse en el directorio `/etc/puppetlabs/code/environments/production/manifest` y listar contenido

P2: **[master]** Editar `site.pp` con el contenido:
```
node default {
  # This is where you can declare classes for all nodes.
  # Example:
  #   class { 'my_class': }
  class { 'java' :
           package => 'java-1.8.0-openjdk-devel',
  }
}

```
P3: **[master | slave]**  Verificar primeramente que el paquete java no esta instalado `java -version` 

P4: **[master]** Validar configuración con el comando `puppet parser validate site.pp`   

P5: **[master |  slave]** Aplicar cambios  con el comando `puppet agent -t`

P6: **[master | slave]**  Verificar que el paquete java se ha instalado `java -version`  y desde la gui verificar informadcion de paquetes reportados por cada nodo.
```
Acceso a la interfaz web:
URL: https://<master-ip>
User: <provistas>
Pass: <provistas>
```
La sección donde se reportan los paquetes instalados es: **INSPECT/Packages** Como se mostró en el demo de la capacitación de introducción a Puppet.

### Ejercicio 3: Aplicación de algunas pautas de PCI simplificadas
El ejercicio muestra como se puede implementar con los recursos de Puppet las pautas de PCI, para ello se seleccionaron como muestra la creación de grupo y usuarios de conjunto con la especificación de alias de los nodos en los hosts y la configuración de servicio de hora sincronizado contra una lista de servidores especifica.

**Pre-condiciones**: Módulos a instar:

 - `puppet module install puppetlabs-ntp --version 7.4.0 --ignore-dependencies`
 - `puppet module install puppetlabs-accounts --version 3.2.0 --ignore-dependencies`
 - `puppet module install puppetlabs-vcsrepo --version 2.4.0`

#### 3.1 Adición de alias en los hosts

P1: **[master]** Ubicarse en el directorio `/etc/puppetlabs/code/environments/production/manifest/`

P2: **[master]** Editar file: `site.pp` con el siguiente contenido
```
# configuracion para despliegue especifico en el nodo o grupos de nodos

node <registered_name in master> {

  host { 'pslave22':
    ensure       => 'present',
    host_aliases => ['pslave22.localdomain'],
    ip           => '10.123.123.123',
    target       => '/etc/hosts',
  }
}
```
P3: **[slave]** Validar: `puppet parser validate site.pp`

P4: **[slave]** Aplicar cambios: `puppet agent -t`

P5: **[slave]** Verificar los cambios `sudo cat /etc/hosts`

#### 3.2 Adición de grupo de usuarios 

P1: **[master]** Ubicarse en el directorio `/etc/puppetlabs/code/environments/production/manifest/`

P2: **[master]** Editar el file `site.pp` adicionando a la especificación del nodo a configurar:
```
  group { 'webb':
    ensure => 'present',
    gid    => '5022',
  }

  user { 'maria2':
    uid      => '4003',
    ensure => present,
    gid      => '5022',
    groups    => 'webb',
    shell    => '/bin/bash',
    password => '123456789',
    password_max_age => '90',
    password_warn_days => '7',
 }
```
P3: **[slave]** Validar: `puppet parser validate site.pp`

P4: **[slave]** Aplicar cambios: `puppet agent -t`

P5: **[slave]** Verificar los cambios `cat /etc/group` y el grupo debe aparecer listado.

P6: **[master]** Verificar la creación de usuario: `cat /etc/passwd|grep maria2` el suario debe estar listado con la información configurada.

#### 3.3 Intalación/Configuración servicio NTP para sincronización de hora.

P1: **[master]** Ubicarse en el directorio `/etc/puppetlabs/code/environments/production/manifest/`

P2: **[master]** Editar el file `site.pp` adicionando a la especificación del nodo a configurar:
```
  class { '::ntp':
    servers => [ '0.south-america.pool.ntp.org', '1.south-america.pool.ntp.org' ],
  }
```
P3: **[slave]** Verificar configuración de servicio NTP previamente con el comando `ntpq -p`. **NOTA:** Seguramente el paquete no estará instalado y de estarlo apuntará los servicios de sincronización a otras URLs.

P4: **[slave]** Validar: `puppet parser validate site.pp`

P5: **[slave]** Aplicar cambios: `puppet agent -t` y repetir verificación **P3** donde se mostrará la configuración de los servidores de hora declarados.
