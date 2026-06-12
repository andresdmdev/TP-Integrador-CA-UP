# Trabajo Práctico Integrador

**Integrantes:**
- Andres David Marquez Fortique
- Evangelina Mendez
---

## 1. Configuración del entorno

**Ensamblaje de la Máquina Virtual**
Se descargaron y descomprimieron los archivos. Luego se abrio en VirtualBox.

**Blanqueo de clave root**
Debido a que la clave de root era desconocida inicialmente, se realizó el siguiente procedimiento para blanquearla y cambiarla por `palermo`:
1. En el arranque, se presionó la letra **E** para editar temporalmente las opciones de GRUB.
2. Se editó la línea `linux /boot/vmlinuz-... ro quiet`, quitando el `ro quiet` y reemplazándolo por `rw init=/bin/bash`.
3. Se presionó **F10** para iniciar directamente con el intérprete de comandos bash en modo lectura y escritura.
4. Se ejecutó el comando `passwd` para configurar la nueva contraseña.
5. Finalmente, se ejecutó `exec /sbin/init` para reiniciar la máquina.

**Configuración del Hostname y Repositorios**
Se estableció el nombre del equipo como `TPServer` editando el archivo `/etc/hosts`, reemplazando el nombre predeterminado por TPServer, y se reinició la bash para aplicar los cambios. Luego, se actualizaron las listas de los repositorios en `/etc/apt/sources.list` ya que se encontraban desactualizadas.

---

## 2. Servicios

### 2.1 Actualización del Sistema Operativo
Se actualizó la distribución del sistema operativo completamente a Debian 12. Primero se actualizaron los paquetes instalados y luego se modificó `/etc/apt/sources.list` (cambiando bullseye por bookworm).

```bash
apt upgrade -y # Actualiza paquetes ya instalados
apt full-upgrade -y # Actualización profunda
apt autoremove -y

# Tras modificar sources.list a bookworm:
apt update
apt upgrade --without-new-pkgs -y # Solo instala lo esencial
apt full-upgrade -y # Instala todo y el kernel
apt autoremove -y
cat /etc/debian_version # Confirmamos la versión
```

### 2.2 Servicio SSH
Se instaló y configuró el servicio SSH para permitir el acceso al usuario root mediante una clave privada/pública, disponible dentro del Home de root. Tambien agregamos la clave publica que aparece 'Material_Adicional_TPVMCA' en a `/root/.ssh/authorized_keys`

```bash
apt update && apt install openssh-server -y

# Generación de claves
ssh-keygen -t rsa -b 4096 -f /root/.ssh/id_rsa -N ""

# Asignación de permisos
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys

# Configuración del servicio
nano /etc/ssh/sshd_config
# Se modificaron las siguientes líneas:
# PermitRootLogin prohibit-password
# PubkeyAuthentication yes

# Se agregaron las claves publicas de la carpeta y las que se crearon
cat /root/id_rsa.pub >> /root/.ssh/authorized_keys
cat /root/Material_Adicional_TPVMCA/clave_publica.pub >> /root/.ssh/authorized_keys

systemctl restart ssh
```

### 2.3 Servidor Web (Apache y PHP)
Se instaló el servidor Apache con soporte para PHP. Se configuró para servir los archivos index.php y logo.png desde el Home de root.

```bash
apt install apache2 php libapache2-mod-php -y

# Configuración de Apache para apuntar al nuevo directorio
nano /etc/apache2/sites-available/000-default.conf
# Se modificó: DocumentRoot /root/www/html

# Permisos globales de Apache
nano /etc/apache2/apache2.conf
# Se añadió:
# <Directory /root/www/html>
#       Options Indexes FollowSymLinks
#       AllowOverride None
#       Require all granted
# </Directory>

# Descarga de la imagen requerida
wget -O /root/www/html/logo.png [https://www.pngfind.com/pngs/m/679-6791685_logo-tecnico-em-informatica-hd-png-download.png](https://www.pngfind.com/pngs/m/679-6791685_logo-tecnico-em-informatica-hd-png-download.png)

systemctl restart apache2
```

### 2.4 Base de Datos (MariaDB)
Se instaló y configuró MariaDB. Se cargó el script SQL db.sql disponible en el Home de root.

```bash
apt install mariadb-server -y

# Instalación de dependencia para evitar error MySQLi
apt install php-mysql -y

# Configuración rápida y segura de la BD
mysql_secure_installation
```

Nota: Las pruebas de conectividad y acceso al sitio web se realizaron desde una maquina fisica distinta en una misma red.

## 3. Configuración de red

Se configuró la interfaz de red con una IP estática, verificando previamente la IP de la máquina principal para asegurar que pertenezca al mismo rango de red. El archivo incluye ADDRESS, NETMASK y GATEWAY.

```bash
ip a
nano /etc/network/interfaces

# Ejemplo de configuración agregada:
# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet static
    address 192.168.0.200
    netmask 255.255.255.0
    gateway 192.168.0.1
    
systemctl restart networking
```

## 4. Almacenamiento

Se agregó un nuevo disco de 10 GB adicional a la máquina virtual y se crearon dos particiones estándar (tipo 83):
- /www_dir: 3 GB
- /backup_dir: 6 GB

```bash
lsblk
fdisk /dev/sdb # Creación de particiones

# Formateo de particiones
mkfs.ext4 /dev/sdb1
mkfs.ext4 /dev/sdb2

# Creación de directorios
mkdir /www_dir
mkdir /backup_dir

# Montaje automático en fstab
nano /etc/fstab
# Se agregaron las líneas:
# /dev/sdb1    /www_dir       ext4    defaults    0    2
# /dev/sdb2    /backup_dir    ext4    defaults    0    2

mount -a # Montar todo
df -h # Verificar

# Respaldo de particiones efímeras
cat /proc/partitions > /opt/particion
```

Migración Web: Se configuró /www_dir para alojar index.php y logo.png, actualizamos los archivos de configuración de Apache (000-default.conf y apache2.conf) apuntando a esta nueva ubicación.

```bash
nano /etc/apache2/apache2.conf
# Se añadió:
<Directory /www_dir/html>
       Options Indexes FollowSymLinks
       AllowOverride None
       Require all granted
</Directory>
# Y lo anterio que esta se comento
```

## 5. Backup

Se desarrolló un script de backup denominado backup_full.sh, guardado en /opt/scripts. 

Características del script:
 - Backupea directorios incluyendo la fecha en formato ANSI (ej: log_bkp_20240302.tar.gz).
 - Almacena los backups en la partición montada /backup_dir.
 - Acepta argumentos de origen (qué backupear) y destino (dónde backupear).
 - Incluye una opción de ayuda (--help y -h).
 - Valida que los sistemas de archivos de origen y destino estén disponibles antes de ejecutar el backup.

Automatización (Cronjobs):
Se incluyó el script en el calendario de tareas (cron) para correr automáticamente:
 - Todos los días a las 00:00 hs: Backupear /var/logs
 - Lunes, Miércoles, Viernes a las 23:00 hs: Backupear /www_dir

# 6. Entregables

En este repositorio de GitHub se encuentran subidos los siguientes recursos (Comprimido en .tar.gz):
 - /root 
 - /etc
 - /opt 
 - /www_dir
 - /backup_dir
 - /var (Dividido en 3 partes pequeñas).

