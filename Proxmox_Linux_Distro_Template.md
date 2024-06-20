![image](https://github.com/vdaular/tutoriales/assets/136828149/5104609b-f977-4038-97d0-fb176daba810)

# Generar una plantilla de distribución Linux específica a partir de una plantilla de Linux genérica

Guía de cómo generar una plantilla de distrobución Linux, como Debian u Oracle Linux, utilizando como base una plantilla de Linux genérica en Proxmox.

## Requerimientos
- Proxmox VE.
- Navegador web.
- ISO de Debian.

## Creación de la máquina virtual

- Clic derecho sobre la plantilla _Template.Linux.Generic_.
- Clic en _Clone_.
- Ajustar las opciones:
    - VM ID: Asignar el identificador que corresponda.
    - Name: Asignar el nombre a la plantilla.
    - Mode: Seleccionar _Full Clone_.
- Clic en _Clone_.

## Configuración de la máquina virtual

- Ingresar por SSH al servidor Proxmox para descargar la imagen de disco de nube:

Debian:
```sh
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2
```

Oracle Linux 8:
```sh
wget https://yum.oracle.com/templates/OracleLinux/OL8/u10/x86_64/OL8U10_x86_64-kvm-b237.qcow2
```

Oracle Linux 9:
```sh
wget https://yum.oracle.com/templates/OracleLinux/OL9/u4/x86_64/OL9U4_x86_64-kvm-b234.qcow2
```

- Para convertir dicha imagen en un disco duro para la máquina virtual, usaremos el siguiente comando:

```sh
qm importdisk <ID de VM> <archivo .qcow2> local-zfs --format qcow2
```

- Ir a la máquina virtual, sección Hardware y seleccionar la línea _Unused Disk 0_.
- Clic en _Edit_.
- Clic en _Add_.
- Luego, seleccionar el nuevo _Hard Disk (scsi0)_ y clic en _Disk Action > Resize_.
- Agregar 30 GB más de espacio o la cantidad de espacio adicional deseado.
- Clic en _Resize disk_.
- Dentro de la máquina virtual, ir a la sección _Options_ y seleccionar _Boot Order_.
- Clic en _Edit_.
- Mover _scsi0_ al inicio y habilitarlo.
- Deshabilitar el resto de dispositivos, en caso de no requerirlos.
- Luego, dentro de la máquina virtual, ir a _Hardware_.
- Clic en _Add_ y luego en _CloudInit Drive_.
- Seleccionamos el Storage deseado. Si se siguió el tutorial de Proxmox en servidor dedicado Hetzner, la opción es _local-zfs_.
- Dentro de la máquina virtual, ir a _Cloud-Init_.
- Configurar los siguientes apartados:
    - User
    - Password: Por seguridad, se sugiere una genérica, como _'password'_, para que luego de generar una nueva máquina virtual, modificarla a la deseada.
    - SSH public keys
    - IP Config (net0): Habilitar DHCP en IPv4.

## Conversión de Máquina Virtual a Template

- Clic derecho sobre la nueva máquina virtual generada.
- Clic en _Convert to template_. (Esta opción también se encuentra al seleccionar la máquina virtual y luego a la derecha en los botones superiores, bajo los principales, dentro del botón _More_).

## Prueba y Acceso SSH

- Generar máquina virtual a partir de la plantilla generada.
- Iniciar máquina virtual e iniciar sesión con la clave genérica.
- Cambiar clave genérica a la contraseña deseada utilizando _passwd_.
- Verificar la IP de la nueva máquina virtual  en Datacenter > SDN > IPAM
- Agregar regla al archivo _/etc/network/interfaces.d/sdn_ según indicaciones para poder acceder por SSH a la nueva máquina virtual.
- Pronar conexión por SSH al puerto asignado a la nueva máquina virtual.

## Distribuciones basadas en RHEL

- El Display para estas máquinas virtuales debe ser _Default_ en lugar de _Serial terminal 0_ que trae de la imagen genérica.