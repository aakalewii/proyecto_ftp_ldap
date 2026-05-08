Proyecto de Infraestructura de Red: DNS, LDAP y Apache

Este proyecto documenta el despliegue de una infraestructura de red profesional separando los servicios en distintas máquinas. La arquitectura se basa en la integración de un servidor de nombres (DNS), un servicio de directorio para la gestión centralizada de usuarios (LDAP) y un servidor web (Apache) con control de acceso basado en roles. Proyecto realizado por Álvaro Curbelo Rodríguez y Lenny Iborra Morán.

1. Preparación del Sistema Operativo y Redes

Para llevar a cabo este entorno, se utilizan dos máquinas virtuales ejecutando el sistema operativo Ubuntu 22.04 LTS. La arquitectura se divide de la siguiente manera:  


VM1 (srvinfra-alvaro-lenny): Actuará como servidor de infraestructura, alojando los servicios DNS y LDAP.  


VM2 (srvweb-alvaro-lenny): Actuará como servidor de servicios, alojando el servidor web Apache (y FTP, a configurar posteriormente).

Configuración de Adaptadores de Red

Ambas máquinas virtuales cuentan con dos tarjetas de red para garantizar la conectividad a Internet y la comunicación privada entre ellas:


Adaptador 1 (NAT): Proporciona salida a Internet para la descarga de paquetes y actualizaciones del sistema sin requerir configuración adicional.  


Adaptador 2 (Red Interna): Funciona como un enlace privado directo entre las dos máquinas virtuales.


Asignación de Direccionamiento IP (Netplan)

Se establecen IPs estáticas utilizando el gestor de red Netplan en el adaptador de la Red Interna (enp0s8):

Srvinfra (VM1): Se le asigna la dirección IP estática 192.168.100.10/24. Se aplican los cambios con el comando netplan apply.  

Srvweb (VM2): Se le asigna la dirección IP estática 192.168.100.20/24. Además, se configura para que su servidor DNS apunte a la IP de la VM1 (192.168.100.10).

2. Servidor DNS (BIND9)

El servidor DNS es fundamental para que las máquinas y servicios puedan localizarse mediante nombres de dominio en lugar de direcciones IP.

Instalación

En la máquina Srvinfra, se actualizan los repositorios y se instalan los paquetes necesarios para el servidor DNS: bind9, bind9utils, bind9-doc y dnsutils.

Configuración de la Zona

Se informa a BIND9 de la creación de una nueva zona local llamada centro.local. Para ello, se edita el archivo /etc/bind/named.conf.local añadiendo la zona como tipo "master" e indicando que su archivo de base de datos estará en /etc/bind/db.centro.local.

Creación de los Registros

Se utiliza la plantilla por defecto de Ubuntu (db.local) copiándola a nuestro nuevo archivo de base de datos (db.centro.local). En este archivo se borra el contenido original y se definen los siguientes registros tipo A para la resolución directa:

ns apunta a 192.168.100.10.  


ldap apunta a 192.168.100.10.  


intranet apunta a 192.168.100.20.  


ftp apunta a 192.168.100.20.

Finalmente, se reinicia y habilita el servicio BIND9 (systemctl restart bind9) para aplicar los cambios.

3. Servidor de Directorio LDAP

LDAP se utilizará para centralizar las credenciales de los usuarios de la red.

Instalación y Configuración Base

En la máquina Srvinfra, se instalan los paquetes slapd y ldap-utils. Durante la instalación, se establece 1234 como contraseña de administración. A continuación, se reconfigura el paquete slapd (dpkg-reconfigure slapd) para establecer el nombre de dominio DNS como centro.local.

Carga Inicial de Datos (Estructura Organizativa)

Para poblar el directorio, se crea un archivo LDIF (carga_inicial.ldif) que define la estructura jerárquica y los usuarios. El archivo contiene:

Unidades Organizativas (OU): Creación de las OUs usuarios y grupos.  

3.1. Usuarios (inetOrgPerson):

alumno1 (Alumno Uno) con contraseña 1234.  

alumno2 (Alumno Dos) con contraseña 1234.  

profe1 (Profesor Alvaro) con contraseña 1234.  

profe2 (Profesor Lenny) con contraseña 1234.

3.2. Grupos (groupOfNames):

Grupo alumnos: Contiene a los miembros alumno1 y alumno2.  

Grupo profesores: Contiene a los miembros profe1 y profe2.

Estos datos se inyectan en el servidor mediante el comando ldapadd autenticándose como administrador.

4. Servidor Web APACHE + Autenticación LDAP

En esta fase, convertimos la máquina Srvweb en un servidor web que valida los accesos consultando al servidor LDAP.

Instalación y Módulos

Se actualizan los repositorios y se instalan los paquetes apache2 y tree. Para que Apache pueda comunicarse con LDAP, se activan los módulos necesarios mediante el comando a2enmod ldap authnz_ldap y se reinicia el servicio.

Estructura Web

Se crea el árbol de directorios para la intranet en /var/www/intranet con tres zonas diferenciadas:

/public (Zona libre).  

/profesores (Zona exclusiva para profesores).  

/alumnos (Zona exclusiva para alumnos).

Se generan archivos index.html básicos en cada carpeta con títulos identificativos ("Zona Pública", "Zona Profesores", "Zona Alumnos") y se otorga la propiedad recursiva de la carpeta al usuario web www-data.


Configuración del VirtualHost

Se define la configuración del sitio web editando el archivo /etc/apache2/sites-available/intranet.conf. La configuración es la siguiente:

ServerName: intranet.centro.local.  

DocumentRoot: /var/www/intranet.  

Directorio /public: Se permite el acceso a todos (Require all granted).  

Directorio /profesores: Se configura autenticación básica (AuthType Basic) respaldada por LDAP (AuthBasicProvider ldap). La URL de LDAP apunta a la máquina Srvinfra (ldap://192.168.100.10/). Se restringe el acceso exigiendo pertenencia al grupo cn=profesores,ou=grupos,dc=centro,dc=local.  

Directorio /alumnos: Sigue la misma estructura de autenticación LDAP que la zona de profesores, pero exige pertenencia al grupo cn=alumnos,ou=grupos,dc=centro,dc=local.

Para finalizar, se activa el nuevo sitio (a2ensite intranet.conf) y se recarga la configuración de Apache (systemctl reload apache2).


5. Servidor FTP (VSFTPD) con Integración LDAP

Para permitir la transferencia de archivos de forma segura y controlada, se configura un servidor FTP en la máquina Srvweb. El objetivo es que los usuarios no se validen contra las cuentas locales del sistema operativo, sino contra la base de datos centralizada de LDAP que reside en la máquina de infraestructura.

Instalación del Servicio y Puente LDAP

Se instalan los paquetes necesarios para el servidor FTP y los módulos que hacen de puente de autenticación: vsftpd, libnss-ldap y libpam-ldapd.

Al tratarse de una arquitectura profesional separada, durante la instalación de los paquetes de autenticación, el sistema nos solicita configurar el cliente LDAP:  


URI del servidor LDAP: Se indica la dirección exacta donde reside el servidor de identidades, en este caso ldap://192.168.100.10.  


Base de Búsqueda (Base DN): Se define la raíz jerárquica del árbol LDAP introduciendo dc=centro,dc=local. Esto es vital para que el servidor web sepa en qué parte del directorio debe buscar las unidades organizativas de usuarios y grupos.

Configuración de la Ruta de Subida

El proyecto requiere una ruta específica para almacenar los archivos subidos vía FTP.
Se crea el directorio destino ejecutando el comando mkdir -p /srv/ftp/publicaciones.
A continuación, para asegurar que no haya bloqueos por permisos estrictos del sistema a la hora de recibir archivos a través del servicio, se otorgan permisos totales recursivos con chmod -R 777 /srv/ftp.

Configuración del Demonio VSFTPD

Para que el servidor FTP se comporte según los requisitos, se edita su archivo de configuración principal ubicado en /etc/vsftpd.conf. Se añaden o descomentan las siguientes directivas clave:  

anonymous_enable=NO: Desactiva el acceso público/anónimo al FTP, obligando a usar credenciales.  

local_enable=YES: Permite el inicio de sesión a los usuarios locales (que, gracias al puente instalado, ahora incluyen a los usuarios de LDAP).  

write_enable=YES: Habilita los comandos de escritura, permitiendo la subida y modificación de archivos.  

local_root=/srv/ftp/publicaciones: Fuerza a que el directorio de inicio al conectarse por FTP sea directamente la carpeta de publicaciones que creamos en el paso anterior.  

chroot_local_user=YES y allow_writeable_chroot=YES: Enjaula (hace chroot) a los usuarios en su directorio de inicio para que no puedan navegar hacia atrás y explorar la raíz del sistema operativo del servidor, mejorando drásticamente la seguridad. 

Finalmente, para que todos estos cambios en el archivo de configuración surtan efecto, se reinicia el servicio FTP ejecutando systemctl restart vsftpd. 