# Instalación de Proxmox en Servidor Dedicado de Hetzner

Guía de cómo instalar Proxmox en un servidor de dedicado de Hetzner, partiendo del Rescue System de Hezner hasta una instalación completa y segura.

## Requerimientos
- Servidor dedicado Hetzner (este tutorial se basará en el tipo de servidores AX41-NVMe y no se garantiza que funcione con otros tipos de servidores de este proveedor).
- Cliente VNC (se utilizará RealVNC).
- Cliente SSH (se utilizará WinSCP+Putty).


## Preparación del servidor
- Activar el Rescue System en Hetzner.
- Realizar un Hardware Reset
- Conectar al servidor por SSH
- Ejecutar los siguientes comandos para la generación de las variables de entorno que facilitarán la instalación y configuración previa:

```sh 
# Descarga de última versión de Proxmox (en este caso es las 8.2-1)
ISO_VERSION=$(curl -s 'http://download.proxmox.com/iso/' | grep -oP 'proxmox-ve_(\d+.\d+-\d).iso' | sort -V | tail -n1) && \
ISO_URL="http://download.proxmox.com/iso/$ISO_VERSION" && \
curl $ISO_URL -o /tmp/proxmox-ve.iso && \

# Obtención de datos de red
INTERFACE_NAME=$(udevadm info -q property /sys/class/net/eth0 | grep "ID_NET_NAME_PATH=" | cut -d'=' -f2) && \
IP_CIDR=$(ip addr show eth0 | grep "inet\b" | awk '{print $2}') && \
GATEWAY=$(ip route | grep default | awk '{print $3}') && \
IP_ADDRESS=$(echo "$IP_CIDR" | cut -d'/' -f1) && \
CIDR=$(echo "$IP_CIDR" | cut -d'/' -f2) && \

# Obtención de datos de almacenamiento para AX41-NVMe
PRIMARY_DISK=$(lsblk -dn -o NAME,SIZE,TYPE -e 1,7,11,14,15 | sed -n 1p | awk '{print $1}') && \
SECONDARY_DISK=$(lsblk -dn -o NAME,SIZE,TYPE -e 1,7,11,14,15 | sed -n 2p | awk '{print $1}') && \

# Configuración de idioma del teclado a utilizar en la instalación
KEYBOARD_LANGUAGE=en-us && \

# Inicio de máquina virtual con Qemu e inicio de VNC Server para la instalación de Proxmox en el almacenamiento utilizando la imagen ISO descargada previamente
qemu-system-x86_64 -daemonize -enable-kvm -m 10240 -k $KEYBOARD_LANGUAGE \
-hda /dev/$PRIMARY_DISK \
-hdb /dev/$SECONDARY_DISK \
-cdrom /tmp/proxmox-ve.iso -boot d -vnc :0,password=on -monitor telnet:127.0.0.1:4444,server,nowait

# Reemplaza VNC_PASSWORD por la contraseña deseada para realizar la conexión VNC
echo "change vnc password VNC_PASSWORD" | nc -q 1 127.0.0.1 4444
```

- Abrir el cliente VNC y conectar utilizando la dirección IP pública del servidor dedicado y el puerto 5900.


## Instalación gráfica de Proxmox
- Al lograr la conexión por VNC, se selecciona la primera opción, instalación gráfica de Proxmox.
- Esperar al inicio del instalador.
- Ignorar el mensaje que indica _"No support for hardware-accelerated KVM virtualization detected"_ haciendo clic en _Ok_.
- Leer haste el final el Acuerdo de Licenciamiento de Usuario Final.
- Clic en _I Agree_.
- Para el almacenamiento, hacer clic en _Options_.
- Para aprovechar al máximo el espacio en disco, seleccionar _zfs (RAID0)_. Para escenarios de producción, se sugiere la opción _zfs (RAID1)_.
- Clic en _Next_.
- Seleccionar país, zona horaria y distribución del teclado.
- Clic en _Next_.
- Configurar una contraseña. Se sugiere una contraseña simple inicialmente para evitar problemas de reconocimiento de caracteres especiales.
- Indicar el correo electrónico. Se sugiere no eliminar el texto completo debido al caracter especial '@' que puede no ser reconocido correctamente luego de eliminarlo. Si por accidente se eliminó, volver a la pantalla anterior y luego nuevamente cargar la configuración de contraseña y correo electrónico.
- Clic en _Next_.
- Cargar Hostname y dejar la configuración de red que viene por defecto.
- Clic en _Next_.
- En el resumen de instalación, desmarcar la opción _"Automatically reboot after successfull installation"_.
- Clic en _Install_.
- Al finalizar la instalación no se debe reiniciar, sino que se vuelve a la terminal de SSH para introducir los siguientes comandos:

```sh
# Detener máquina virtual QEMU
printf "quit\n" | nc 127.0.0.1 4444
```

- Ahora, se configura una nueva máquina virtual de QEMU sin el uso de la ISO descargada, para lo cual se ejecutará lo siguiente:

```sh
# Configuración de idioma del teclado a utilizar en la instalación
KEYBOARD_LANGUAGE=en-us && \

# Inicio de máquina virtual con Qemu e inicio de VNC Server para la instalación de Proxmox en el almacenamiento utilizando la imagen ISO descargada previamente
qemu-system-x86_64 -daemonize -enable-kvm -m 10240 -k $KEYBOARD_LANGUAGE \
-hda /dev/$PRIMARY_DISK \
-hdb /dev/$SECONDARY_DISK \
-vnc :0,password=on -monitor telnet:127.0.0.1:4444,server,nowait \
-net user,hostfwd=tcp::2222-:22 -net nic

# Reemplaza VNC_PASSWORD por la contraseña deseada para realizar la conexión VNC
echo "change vnc password VNC_PASSWORD" | nc -q 1 127.0.0.1 4444
```

- Conectar nuevamente con VNC utilizando la IP pública del servidor dedicado y la dirección IP 5900.
- Esperar a que finalice el inicio del servidor.

## Configuración de red inicial

- Una vez iniciado el servidor Proxmox, se vuelve a la terminal SSH para instalar sshpass con el siguiente comando:

```sh
apt install sshpass -y
```

- Los siguientes comandos permitirán realizar la configuración de red para que Proxmox inicie correctamente luego del reinicio. Se deberá reemplazar ROOT_PASSWORD por la contraseña generada en la instalación gráfica de Proxmox.

```sh
cat > /tmp/proxmox_network_config << EOF
auto lo
iface lo inet loopback

iface $INTERFACE_NAME inet manual

auto vmbr0
iface vmbr0 inet static
  address $IP_ADDRESS/$CIDR
  gateway $GATEWAY
  bridge_ports $INTERFACE_NAME
  bridge_stp off
  bridge_fd 0
EOF

# transfer the network configuration file to Proxmox VE system
sshpass -p "ROOT_PASSWORD" scp -o StrictHostKeyChecking=no -P 2222 /tmp/proxmox_network_config root@localhost:/etc/network/interfaces

# update the nameserver
sshpass -p "ROOT_PASSWORD" ssh -o StrictHostKeyChecking=no -p 2222 root@localhost "sed -i 's/nameserver.*/nameserver 1.1.1.1/' /etc/resolv.conf"
```

- Lo siguiente será apagar la máquina virtual de QEMU con el siguiente comando:

```sh
printf "system_powerdown\n" | nc 127.0.0.1 4444
```

- Ahora quedó todo listo para poder iniciar Proxmox directamente, reiniciando el servidor lo que generará que salga del Rescue System. Usaremos el siguiente comando:

```sh
shutdown -r now
```

## Configuración de Proxmox

Ahora, es posible acceder a Proxmox en el navegador por https utilizando la dirección IP del servidor dedicado y el puerto 8006 y se usará para el inicio de sesión el usuario _root_ y la contraseña elegida durante la instalación gráfica.

- Para lo siguiente, se requiere hacer fork al repositorio _https://github.com/vdaular/Proxmox_Helper_Scripts_, que es fork del repositorio original _https://github.com/tteck/Proxmox_. Este repositorio contiene un script de configuración post-instalación  que realiza varios ajustes dentro de Proxmox, como por ejemplo recordatorios de suscripción y configuración de repositorios de paquetes de la comunidad.
- A la izquierda, seleccionar el Host dentro del Datacenter.
- Ejecutar el siguiente comando con la información según el repositorio resultante. Se debe reemplazar _USUARIO_ y _REPOSITORIO_ por los datos que correspondan.

```sh
bash -c "$(wget -qLO - https://github.com/USUARIO/REPOSITORIO/raw/main/misc/post-pve-install.sh)"
```

- Escribir _'y'_ (sin comillas) y presionar Enter para iniciar el script.
- Responder _'yes'_ para corregir fuentes de Proxmox VE (Correct Proxmox VE sources?). Se puede responder _'no'_ si se cuenta con una licencia.
- Responder _'yes'_ para deshabilitar el repositorio 'pve-enterprise' (Disable 'pve-enterprise' repository?). Se puede responder _'no'_ si se cuenta con una licencia.
- Responder _'yes'_ para habilitar el repositorio 'pve-no-subscription' (Enable 'pve-no-suscription' repository?). Se puede responder _'no'_ si se cuenta con una licencia.
- Responder _'yes'_ para corregir fuentes de paquetería ceph (Correct 'ceph' package sources?). Se puede responder _'no'_ si se cuenta con una licencia.
- Responder _'yes'_ o _'no'_ para agregar repositorio 'pvetest' (Add [Disabled] 'pvetest' repository?), según decida el usuario dado que permite agregar nuevas funciones y actualizaciones para usuarios avanzados.
- Responder _'yes'_ para deshabilitar el recordatorio de suscripción (Disable subscription nag?). Se puede responder _'no'_ si se cuenta con una licencia.
- Responder Ok al aviso _Support Subscriptions_.
- Responder _'yes'_ para deshabilitar la alta disponibilidad (Disable High Availability?), a no ser que se requiera mantener habilitada.
- Responder _'yes'_ para actualizar Proxmox (Update Proxmox VE now?).
- Responder _'yes'_ para reiniciar Proxmox (Rebot Proxmox VE now?).


## Configuración de Redes Definidas por Software (Software Defined Network - SDN)

Los siguientes comandos, a ejecutar desde el Shell del host, permitirán hacer uso del servidor DHCP en la sección SDN para lo que se requiere desinstalar _dnsmasq_ y deshabilitar el servidor dnsmasq por defecto:

```sh
apt update
apt -y install dnsmasq
systemctl disable --now dnsmasq
```

Ahora, se ejecuta el siguiente comando:

```sh
cat /etc/network/interfaces
```

Si la última línea no es _source /etc/network/interfaces.d/*_ se deberá agregar con el siguiente comando:

```sh
printf "\n\nsource /etc/network/interfaces.d/*" >> /etc/network/interfaces
```

### Creación del SDN (Red Virtual)

#### Zona Simple
- En la interfaz web de Proxmox, se debe ir a la siguiente ruta _Datacenter > SDN > Zones_
- Crear una zona simple (Simple Zone) y asignarle un nombre corto en minúsculas. Adicionalmente, se deberá tildar la opción de DHCP automático (automatic DHCP) en la sección Advanced (Advanced debe estar tildado para visualizarlo).

#### VNet
- Ahora, desde la sección _Datacenter > SDN > VNets_ se crea una nueva VNet con el nombre deseado, por ejemplo vnet0, y seleccionando la zona simple creada en el paso anterior.

#### Subnet
- Para crear la subnet, a la derecha de la vista de VNets y manteniendo seleccionada la VNet recién creada, se hace clic en Create, con la siguiente configuración:
    - *Subnet:* 10.0.0.0/24
    - *Gateway:* 10.0.0.1
    - *SNAT:* habilitado
    - *DHCP Ranges*: _Start Address_ 10.0.0.100; _End Address_ 10.0.0.200

#### Activación de SDN

- Volviendo a la sección _Datacenter > SDN_ y hacer clic en aplicar (Apply).
- Luego, desde la sección _Datacenter > SDN > IPAM_ se podrán gestionar las reservas de direcciones IP de DHCP.

## Configuración de Firewall

*AVISO: NO HABILITAR EL FIREWALL HASTA CREAR LAS REGLAS PARA SSH Y/O HTTPS (8006) O QUEDARÁ SIN ACCESO DEBIENDO VOLVER A INICIAR TODO ESTE TUTORIAL DESDE EL PRINCIPIO.*

Para que las máquinas virtuales y contenedores puedan contactar con el servidor DHCP, se deberá realizar la configuración del firewall previamente.

- Para hacerlo a nivel de datacentes, se ingresa a la sección _Datacenter > Firewall_. Si se desea hacer a nivel de Host, se ingresa a la sección _Host > Firewall_.
- Se agregan las reglas para SSH y el puerto 8006 con las siguientes configuraciones:
    - Direction: in, Action: ACCEPT, Macro: SSH, Enable: tildado
    - Direction: in, Action: ACCEPT, Protocol: TCP, Dest. port.: 8006, Enable: tildado
- Para habilitar el redireccionamiento de DHCP se agrega otra regla con la siguiente configuración:
    - Direction: in, Action: ACCEPT, Interface: vnet0 (nombre de la VNet creada anteriormente), Macro: DHCPfwd, Enable: tildada
- Para habilitar los DNS se agrega otra regla con la siguiente configuración:
    - Direction: in, Action:ACCEPT, Interface: vnet0 (nombre de la VNet creada anteriormente), Macro: DNS, Enable: tildada
- Luego de configurar las reglas, desde la sección Options de Firewall, seleccionar  Firewall y hacer en Edit para finalmente tildar la casilla de activación. 


### Prueba de conectividad

Ahora, es posible crear un contenedor para probar la conectividad.

- Desde la sección _Datacenter > Hostname > local_ (almacenamiento local del host), seleccionar _CT Templates_ y presionar el botón Templates para descargar una plantilla de prueba como la de Debian 12, por ejemplo.
- Una vez creada la plantilla, se hace clic en _Create CT_ en la parte superior derecha de la interfaz.
- Cargar una contraseña de acceso para el contenedor y otros ajustes según se desee.
- En la sección Template, se selecciona la plantilla descargada en pasos anteriores.
- En la secciones Disks, CPU y Memory, hacer los ajustes que se deseen.
- En la sección Network se configura lo siguiente:
    - *Bridge:* nombre de la VNet (creada anteriormente)
    - *IP4:* DHCP
- Finalizado lo anterior, iniciar contenedor e ingresar a su sección Console.
- Iniciar sesión en el contenedor con la credencial elegida en la creación del mismo.
- Hacer pruebas de ping a Google, por ejemplo.
- Verificar en la configuración de _Datacenter > SDN > IPAM_ la reserva de IP de DHCP para este contenedor.

## Redirección de Puertos

Para poder disponibilizar accesos, aplicaciones o servicios de contenedores o maquinas virtuales, no existe una interfaz gráfica, por lo que debe utilizarse el Shell del datacenter o a través de una terminal SSH al servidor dedicado.

La forma más eficiente y permanente de hacerlo, sin ingresar comandos y con la configuración realizada en el SDN, es acceder al archivo de configuración de red de SDN, con el siguiente comando:

```sh
nano /etc/network/interfaces.d/sdn
```

Se sugiere actualizar este archivo, identificando las secciones para que luzcan como en el siguiente ejemplo:

```
#version:1

auto vnet0
iface vnet0
    # BRIDGE
	address 10.0.0.1/24
	post-up iptables -t nat -A POSTROUTING -s '10.0.0.0/24' -o vmbr0 -j SNAT --to-source IP_DE_SERVIDOR_DEDICADO
	post-down iptables -t nat -D POSTROUTING -s '10.0.0.0/24' -o vmbr0 -j SNAT --to-source IP_DE_SERVIDOR_DEDICADO
	post-up iptables -t raw -I PREROUTING -i fwbr+ -j CT --zone 1
	post-down iptables -t raw -D PREROUTING -i fwbr+ -j CT --zone 1

	# SSH
	post-up iptables -t nat -A PREROUTING -p tcp -d IP_DE_SERVIDOR_DEDICADO --dport 2220 -i vmbr0 -j DNAT --to-destination IP_DE_VM_O_CONTENEDOR:22

	# APPS
	post-up iptables -t nat -A PREROUTING -p tcp -d IP_DE_SERVIDOR_DEDICADO --dport 30080 -i vmbr0 -j DNAT --to-destination IP_DE_VM_O_CONTENEDOR:PUERTO_APP

    # BRIDGE
	bridge_ports none
	bridge_stp off
	bridge_fd 0
	ip-forward on
```

Referencias:

[Install Proxmox On A Hetzner Dedicated Server With 1 IP Using SDN And Without KVM Using QEMU - CyanLabs](https://cyanlabs.net/tutorials/install-proxmox-on-a-hetzner-dedicated-server-with-1-ip-using-sdn-and-without-kvm-using-qemu/)