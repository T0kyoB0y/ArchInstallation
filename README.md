# Guia de Instalación de Arch Linux Dualboot (UEFI)
__Asumiendo que ya se haya cargado correctamente el medio de instalación, se procederá a mencionar comandos a utilizar para poder cargar el sistema operativo dentro de su equipo:__

### Indice

- Preparación Inicial:   
	- [Configurando el Teclado](#disposición-del-teclado)
	- [Particionado](#particionado-y-formateo)
	- [Formateo de Particiones](#formateo-de-particiones)
- Montaje, Instalación y Configuración:
	* [Montaje de Discos](#montaje-de-discos)
	* [Internet](#internet)
	* [Instalación de paquetes](#descargamos-paquetes-en-mnt)
	* [Archivo fstab](#generamos-el-archivo-fstab)
	* [Añadimos usuarios y contraseñas](#ingresamos-al-sistema)
	* [Inicio de Servicios](#iniciamos-servicios-de-wifi)
	* [Permisos y Archivos](#configuración-de-permisos)
	* [Configuramos Bootloader](#configuramos-el-bootloader)
- Más Configuración
	* [Drivers de Video](#descargar-drivers-de-video)
	* [Paquetes AUR](#arch-user-repository)
	* [Entorno de Escritorio](#entorno-de-escritorio)


# Preparación Inicial
### Disposición del Teclado
##### Para cambiar la disposición de tu teclado:  

###### **QUERTY Latam**    
```bash
loadkeys la-latin1
```  
### Particionado y Formateo

##### Comenzamos listando las particiones del sistema con las que trabajaremos:

```bash
lsblk
```
```bash
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    1   7.5G  0 disk 
└─sda1        8:1    1   7.5G  0 part /run/archiso/bootmnt
nvme0n1     259:0    0 238.5G  0 disk 
├─nvme0n1p1 259:1    0   100M  0 part
├─nvme0n1p2 259:2    0    16M  0 part
├─nvme0n1p3 259:3    0 140.1G  0 part
└─nvme0n1p4 259:4    0   638M  0 part

```

#### Usamos cfdisk con nuestro disco principal
##### Ejemplo usando `/dev/nvme0n1`
```bash
cfdisk /dev/nvme0n1
```
```bash
			Disk: /dev/nvme0n1

    Device               Size Type
    /dev/nvme0n1p1       100M EFI Systemb
    /dev/nvme0n1p2       16M Microsoft reserved
    /dev/nvme0n1p3       140.1G Microsoft basic data
    /dev/nvme0n1p4       638M Windows recovery environment
	Free Space		  	 97.7G
```

##### Creamos particiones siguiendo el siguiente esquema y recomendaciones:

#### Esquema:
```bash
	Size    Type
    512M 	EFI System
    XG   	Linux filesystem
    XG 		Linux swap
```
##### Recomendacion para el Swap

| Cantidad de RAM | Swap	 	   	|
|-----------------|-----------------|
| < 1 GB RAM   	  | 2 Gb swap       |
| 2 - 4 GB RAM    | 2-4 Gb swap     |
| 8 GB RAM   	  | 4 Gb swap       |
| >8 Gb RAM   	  | 2-4 Gb swap     |

##### Finalmente se verá algo así
```bash
    Device               Size Type
    /dev/nvme0n1p1       100M EFI System
    /dev/nvme0n1p2       16M Microsoft reserved
    /dev/nvme0n1p3       140.1G Microsoft basic data
    /dev/nvme0n1p4       638M Windows recovery environment
	/dev/nvme0n1p5		 512M EFI System
	/dev/nvme0n1p6		 93G Linux filesystem
	/dev/nvme0n1p7		 4G Swap
```

##### Le damos a `[Write]` y luego a `[Quit]`

## Formateo de Particiones

##### Para el `/dev/nvme0n1p5` `EFI System`

```bash
mkfs.vfat -F32 /dev/nvme0n1p5
```

##### Para el `/dev/nvme0n1p6` `Linux filesystem`

```bash
mkfs.ext4 /dev/nvme0n1p6
```

##### Para el `/dev/nvme0n1p7` `Swap`
```bash
mkswap /dev/nvme0n1p7
```
##### Activamos partición swap
```bash
swapon /dev/nvme0n1p7
```


# Montaje, Instalación y Configuración:
### Montaje de Discos
##### Primero montamos la particion `/dev/nvme0n1p6`  `Linux filesystem`

```bash
mount /dev/nvme0n1p6 /mnt
```
##### Creamos directorios `/mnt/boot/efi`
```bash
mkdir /mnt/boot
```
```bash
mkdir /mnt/boot/efi
```
##### Montamos partición  `/dev/nvme0n1p5` `Efi System`

```bash
mount /dev/nvme0n1p5 /mnt/boot/efi
```

### Internet

#### Conexión a Internet con `iwctl`

##### Listamos los dispositivos
```bash
device list
```
##### En caso de estar apagado `wlan0` lo encendemos
```bash
device wlan0 set-property Powered on
```
```bash
adapter phy0 set-property Powered on
```
##### Escaneamos redes Wifi
```bash
station wlan0 scan
```
##### Listamos las redes Wifi

```bash
station wlan0 get-networks
```
##### Nos conectamos a un Wifi
```bash
station wlan0 connect "SSID"
```
##### Salimos del modo `iwctl`
```bash
exit
```
##### Probamos la conexión a Internet
```bash
ping -c 1 duckduckgo.com
```
```bash
PING duckduckgo.com (191.235.123.80) 56(84) bytes of data.
64 bytes from 191.235.123.80: icmp_seq=1 ttl=113 time=70.8 ms

--- duckduckgo.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 70.756/70.756/70.756/0.000 ms
```
### Instalación
##### Descargamos paquetes en `/mnt`

```bash
pacstrap /mnt linux linux-firmware networkmanager grub wpa_supplicant base base-devel efibootmgr dhcpcd
```

### Configuración
##### Generamos el archivo `fstab`
```bash
genfstab -U /mnt > /mnt/etc/fstab
```
##### Revisamos el archivo
```bash
cat /mnt/etc/fstab
```
```bash
# UUID=9a5e1d8b-d0da-4da5-804e-950964302b62
/dev/nvme0n1p6      	/         	ext4

# UUID=6972-BBF8
/dev/nvme0n1p5      	/boot/efi 	vfat

# UUID=c79b9f7a-71f0-4d31-be71-ac70c478ae91
/dev/nvme0n1p7      	none      	swap
```

##### Ingresamos al Sistema
```bash
arch-chroot /mnt
```
##### Definimos el nombre del PC

```
echo PC-Name > /etc/hostname
```


##### Definimos una contraseña para el root (superusuario)
```bash
passwd
```

##### Creamos un usuario
```
useradd -m usuario
```
##### Creamos una contraseña para el usuario
```
passwd usuario
```

##### Añadimos al usuario al grupo `wheel`
```
usermod -aG wheel usuario
```
##### Comprobamos
```
groups usuario
```
##### Activamos servicios de Wifi

```
systemctl enable NetworkManager
```
```
systemctl enable wpa_supplicant
```


#### Configuramos el bootloader

##### Instalamos grub

```
grub-install --efi-directory=/boot/efi --bootloader-id='Arch Linux' --target=x86_64-efi
```

##### Actualizar la configuración de grub

```
grub-mkconfig -o /boot/grub/grub.cfg
```

##### Salimos y Reiniciamos

```
exit
```
```
reboot now
```
Luego se debe de ingresar a la sesión con el usuario root

### Más Configuraciones

### Nos conectamos a internet
##### Listamos las red Wifi
```
nmcli device wifi list
```
##### Nos conectamos nuevamente a una red Wifi
```
nmcli device wifi connect "SSID" password "PASS"
```

##### Descargamos un editor de texto
```
pacman -S nano
```
##### Para dualboot se debe de instalar `os-prober`
```
pacman -S os-prober
``` 

##### Acceder al archivo, descomentar la linea y salir 
`/etc/default/grub`
```
GRUB_DISABLE_OS_PROBER=false
```
##### Luego ejecutamos os-prober
```
os-prober
```

##### Actualizar la configuración de grub

```
grub-mkconfig -o /boot/grub/grub.cfg
```



## Configuración de Permisos

##### Acceder al archivo, descomentar la linea y salir 

`/etc/sudoers`
```
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL:ALL) ALL
```

##### Acceder al archivo, descomentar la linea y salir 
`/etc/locale.gen`
```
en_US.UTF-8 UTF-8
```
```
es_CL.UTF-8 UTF-8
```
##### Ejecutamos el comando
```
locale-gen
```

##### Acceder al archivo, crear una configuración y salir 
`/etc/vconsole.conf`
```
KEYMAP=la-latin1
```

#### Descargar drivers de video
###### Intel
```
sudo pacman -S xf86-video-intel intel-ucode
```
###### AMD
```
sudo pacman -S xf86-video-amdgpu
```
###### Nvidia
```
sudo pacman -S xf86-video-nouveau 
```
###### Genericos
```
sudo pacman -S mesa
```
###### Maquina Virtual
```
sudo pacman -S open-vm-tools xf86-vide-vmware xf86-input-vmouse
```
```
systemctl enable vmwtoolsd
```


#### Arch User Repository
##### Instalamos git
```
sudo pacman -S git
```
##### Clonamos un repositorio 
```
git clone https://aur.archlinux.org/paru-bin.git
```
##### Entramos a la carpeta clonada y compilamos el paquete
```
cd paru-bin/
```
```
makepkg -si
```
Aquí tienes la lista con los comandos de instalación exclusivos de Arch Linux para cada entorno de escritorio:

#### Entorno de Escritorio

##### Primero instalamos paquetes necesarios
```
sudo pacman -S xorg xorg-xwayland wayland
```



##### Wayland:
###### Hyprland/Hyprland Nvidia
```
paru -S hyprland-git
```
##### Tutorial para nvidia en este [link](https://wiki.hyprland.org/Nvidia/)
```
paru -S hyprland-nvidia-git
```
###### GNOME
```
sudo pacman -S gnome
```
###### KDE Plasma
```
sudo pacman -S plasma-wayland-session
```
##### Xorg/x11

###### Xfce
```
sudo pacman -S xfce4
```
###### Cinnamon
```
sudo pacman -S cinnamon
```
###### LXQt
```
sudo pacman -S lxqt
```
###### MATE
```
sudo pacman -S mate
```
##### Wayland / Xorg
###### GNOME
```
sudo pacman -S gnome
```
###### KDE Plasma
```
sudo pacman -S plasma
```