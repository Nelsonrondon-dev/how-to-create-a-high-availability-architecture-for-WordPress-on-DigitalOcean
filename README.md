# Guía para crear una arquitectura de alta disponibilidad para WordPress en DigitalOcean

Este proyecto describe cómo crar una arquitectura de alta disponibilidad para WordPress en DigitalOcean

## Instalación

### Paso 1

Inicia sesión en tu cuenta de DigitalOcean y crea dos nodos para esto usaremos dos Droplet con plesk instalado [(Aqui adjunto una guia de como puedes realizarlo)](https://github.com/Nelsonrondon-dev/plesk-in-Droplet-of-DigitalOcean). Puedes elegir la distribución de Linux que prefieras, pero en este ejemplo usaremos Ubuntu.

### Paso 2

Ahora deberas configurar un Load Balancer de carga para distribuir el tráfico entre los nodos.

1. Inicia sesión en tu cuenta de DigitalOcean y haz clic en el botón "Create" en la esquina superior derecha.
2. Selecciona "Load Balancers" en el menú desplegable y haz clic en "Get started".
3. En la página siguiente, selecciona la región en la que deseas crear el balanceador  de carga (Debe ser el mismo que los nodos) y el tipo de plan que prefieras. A continuación, haz clic en "Continue".
4. En la página de configuración, proporciona un nombre para el balanceador de carga y selecciona los nodos de WordPress que deseas agregar al balanceador. También puedes configurar opcionalmente la configuración de red y la protección contra ataques DoS.
5. Una vez que hayas terminado de configurar el balanceador de carga, haz clic en el botón "Create Load Balancer".
6. Una vez que el balanceador de carga esté creado, verás una pantalla de resumen con la información del balanceador y las instrucciones para configurarlo. 


### Paso 3

Crear la base de datos.

Para garantizar la integridad de la base de datos, puedes replicar la base de datos a través de los nodos. Esto se puede hacer mediante la configuración de la replicación maestro-esclavo de MySQL  o puedes crear un droplet donde  se cree el servidor de mysql de forma independiente a los nodos del sistema, lo cual permita mantener la uniformidad de los datos.

### Paso 4 

Configura un sistema de replicación de archivos para asegurar que ambos servidores droplets tengan la misma información de archivos en tiempo real. Para esot puedes utilizar herramientas como rsync o GlusterFS para hacer esto, en este caso lo haremos con GlusterFS

1. Asignamos nombres de host a direcciones IP, para esto ejecutamos el siguiente comando
 
 `sudo nano /etc/hosts`

aqui pegamos, las siguientes lineas sustituyendo `IP_DEL_DROPLET` por la IP privada de cada droplet y modificando el `ejemplo.com` por su dominio


`IP_DEL_DROPLET` server0.ejemplo.com server0
`IP_DEL_DROPLET` server1.ejemplo.com server1


2. Instala GlusterFS en ambos servidores. En un sistema basado en ubuntu, puedes hacerlo ejecutando los siguientes comando:

`sudo add-apt-repository ppa:gluster/glusterfs-7`

`sudo apt update`
`sudo apt install glusterfs-server`


3. Inicia el servicio GlusterFS en ambos servidores ejecutando el siguiente comando:


`sudo systemctl start glusterd.service`

4.  Verifica que se ha iniciado ejecutando el siguiente comando:

`sudo systemctl status glusterd.service`


### Paso 5

Abrir los puertos y permitir la conexión entre dos servidores utilizando iptables y ufw.

1. Abre las IP para conectarse al otro servidor ejecutando el siguiente comando en el nodo 0:

`sudo iptables -I INPUT -p all -s servidor1 -j ACCEPT`
`sudo ufw allow from servidor1 to any port 24007`

2. Abre los puertos en el firewall del servidor 0 ejecutando el siguiente comando en el nodo 1:

`sudo iptables -I INPUT -p all -s servidor0 -j ACCEPT`
`sudo ufw allow from servidor0 to any port 24007`

3. Deniega el puerto 24007 en ambos nodos ejecutando el siguiente comando:

`sudo ufw deny 24007`

4.  Probar la conexión ejecutando el siguiente comando en cada nodo:

En el nodo 0  `sudo gluster peer probe server1`

En el nodo 1 `sudo gluster peer probe server0`


5. Crea una partición o directorio en cada servidor que se utilizará como almacenamiento para el volumen distribuido. Por ejemplo, si quieres utilizar el directorio /gluster-storage en ambos servidores, puedes crearlo ejecutando el siguiente comando en cada servidor:

`sudo mkdir /gluster-storage`


6. Crea el volumen e inicializalo ejecutando los siguientes comando en cualquiera de los dos nodos:


`sudo gluster volume create volume1 replica 2 server0.ejemplo.com:/gluster-storage server1.ejemplo.com:/gluster-storage force`

donde  `server0` y `server1` son los nombres de cada host de tus servidores, `ejemplo.com` es el nombre de vuestro dominio y `volume1` es el nombre que quieres darle al volumen distribuido. La opción `replica 2 ` indica que se creará una réplica del volumen en ambos servidores, lo que aumenta la disponibilidad de los datos.

7.  Inicia el volumen distribuido ejecutando el siguiente comando:

`sudo gluster volume start volume1`

Reemplaza `volume1` con el nombre del volumen distribuido que acabas de crear o si has usado el mismo deja este tal cual.

8. Monta el volumen GlusterFS en el nodo 0 utilizando el siguiente comando

`sudo mount -t glusterfs server0.ejemplo.com:/volume1 /var/www/html`

9. Monta el volumen GlusterFS en el nodo 1 utilizando el siguiente comando

`sudo mount -t glusterfs server1.ejemplo.com:/volume1 /var/www/html`

10. Verifica que el volumen esté montado correctamente en el otro servidor ejecutando el siguiente comando:

`df -h`


11. Por ultimo crea un archivo de prueba en el directorio montado en el otro servidor y verifica que se replique correctamente en el otro servidor ejecutando el siguiente comando:


`sudo touch file_{0..9}.prxxx`


