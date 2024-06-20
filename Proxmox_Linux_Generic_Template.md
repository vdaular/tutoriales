![image](https://github.com/vdaular/tutoriales/assets/136828149/5104609b-f977-4038-97d0-fb176daba810)

# Generar una plantilla de Linux genérica

Guía de cómo generar una plantilla de Linux genérica en Proxmox como base para generar nuevas máquinas virtuales o plantillas de distribuciones Linux específicas.

## Requerimientos
- Proxmox VE.
- Navegador web.

## Creación de la máquina virtual

- Presionar el botón _Create VM_
- Tildar checkbox _Advanced_
- Configurar los siguientes apartados:
    - General > Nodo: Nodo de Proxmox donde se hospedará la máquina virtual.
    - General > VM ID: Identificador para la máquina virtual.
    - General > Name: Nombre de la máquina virtual.
    - OS > Do not use any media
    - System > Graphic card: Seleccionar _Serial terminal 0_.
    - System > Qemu Agent: Tildar para habilitar.
    - Disks: Eliminar discos.
    - CPU > Type: Seleccionar _host_ o dejar Type por defecto.
    - Memory > Balloning Device: Deshabilitar para evitar problemas de rendimiento en la asignación dinámica de memoria.
    - Network > Bridge: Si se implementó el tutorial de instalación de Proxmox en un servidor dedicado de Hetzner, debe cambiarse a _vnet0_. 
- Hacer clic en Finish previa verificación de que la opción _Start after created_ esté deshabilitada.

## Conversión de Máquina Virtual a Template

- Clic derecho sobre la nueva máquina virtual generada.
- Clic en _Convert to template_. (Esta opción también se encuentra al seleccionar la máquina virtual y luego a la derecha en los botones superiores, bajo los principales, dentro del botón _More_).