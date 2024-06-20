![image](https://github.com/vdaular/tutoriales/assets/136828149/a34a86a2-6bf3-4ebc-ba89-e9b567dfd0b1)

# Generar una máquina virtual con Windows Server 2022 Standard

Guía de cómo generar una máquina virtual con Windows Server 2022 Standard Desktop Experience en Proxmox.

## Requerimientos
- Proxmox VE.
- Navegador web.
- ISO de Windows Server 2022.
- ISO de VirtIO-Win v0.1.240 o superior.

## Creación de la máquina virtual

- Clic en _Create VM_.
- Ajustar las opciones:
    - General > VM ID: Asignar el identificador que corresponda.
    - General > Name: Asignar el nombre a la máquina virtual.
    - OS > Use CD/DVD disc image file (iso) > Storage: Por defecto, es _local_.
    - OS > Use CD/DVD disc image file (iso) > ISO image: Cargar la ISO de _Windows Server 2022_.
    - OS > Guest OS: Seleccionar Type _Windows_ y marcar la opción _Add additional driver for VirtIO drivers_.
    - En las opciones de VirtIO drivers, Storage es _local_ por defecto y en _ISO image_ se carga la imagen de _VirtIO-Win_.
    - System > Graphic card: Default.
    - System > Machine: q35.
    - System > SCSI Controller: Cambiar a VirtIO SCSI.
    - System > Qemu Agent: Habilitado.
    - System > Firmware > BIOS: OVMF (UEFI).
    - System > Firmware > Add EFI Disk: Habilitado.
    - System > Firmware > EFI Storage: local-zfs.
    - System > Firmware > Add TPM: Deshabilitado.
    - Disks > Disk > Bus/Device: VirtIO Block
    - Disks > Disk > Disk size (GiB): 64.
    - Disks > Disk > Discard: Habilitado.
    - Disks > Disk > Backup: Deshabilitado.
    - Disks > Disk > IO thread: Deshabilitado.
    - CPU > Type: Valor por defecto o _host_ si no se planea mover dentro de cluster entre nodos con distinto procesador.
    - Memory > Balloning Device: Habilitado.
    - Network > Bridge: vnet0.
- Clic en Finish.

## Configuración de la máquina virtual

- Iniciar máquina virtual y presionar una tecla para el boot del setup de Windows.
- Seleccionar idioma de teclado y clic en _Install Now_.
- Clic en _I don't have a product key_.
- Seleccionar la edición de Windows Server. Para este caso Windows Server 2022 Standard con Desktop Experience y clic en _Next_.
- Aceptar acuerdos de licenciamiento y clic en _Next_.
- Seleccionar tipo de instalación _Custom_.
- Clic en _Load driver_ y luego clic _Browse_.
- Seleccionar unidad de disco de VirtIO-Win y abrir la carpeta _amd64/wk22_ y aceptar.
- Clic _Next_.
- Clic en _Load driver_ y luego clic _Browse_.
- Seleccionar unidad de disco de VirtIO-Win y abrir la carpeta _NetKVM/wk22/amd64_ y aceptar.
- Clic _Next_.
- Seleccionar disco duro y clic en _Next_.
- Esperar a que finalice la instalación. Durante este proceso, la máquina virtual reiniciará dos veces.
- Configurar la contraseña del usuario _Administrator_.
- Instalar actualizaciones.
- Instalar drivers VirtIO-Win.
- Configurar accesos remotos por SDN.
