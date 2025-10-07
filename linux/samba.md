# Samba

## Conceptos de Samba

Del lado del servidor, tenemos:
- **nmbd**: Servicio responsable de las solicitudes de ficheros e impresión.
- **smbd**: Registra todos los nombres NetBIOS y responde a las consultas de nombres.
- **/etc/samba/smb.conf**: Fichero de configuración del servidor Samba.
- **testparm**: Comprueba la sintaxis del fichero smb.conf
- **/etc/samba/smbpasswd**: Base de Datos de cuentas de usuarios para las contraseñas Samba codificadas.

Del lado del cliente:
- **smbclient**: Permite acceder a los recursos compartidos NetBIOS, consultar la lista de recursos compartidos, asegurar los recursos compartidos NetBIOS e imprimir en impresoras NetBIOS.
- **nmblookup**: Herramienta de diagnóstico para la resolución de nombres NetBIOS.

## Instalación

Primero instalamos los paquetes necesarios.
```
# dnf install -y samba samba-client samba-common
```

## Configuración

Editamos el archivo de configuración /etc/samba/smb.conf
```
# nano /etc/samba/smb.conf
```

añadiremos lo siguiente:
```
[global]
workgroup = EMPRESA
security = user
netbios name = samba.empresa
passdb backend = tdbsam
server string = Servidor Samba de Mi Empresa
interfaces = enp0s3 192.168.38.133/24 # Reemplazamos enp0s3 por nuestra interfaz de red, y ponemos la IP del servidor
printing = cups
printcap name = cups
load printers = yes
cups options = raw
local master = no
domain master = no
domain logons = no
```

Habilitamos e iniciamos el servicio `smb`
```
# systemctl enable --now smb
```

Ahora, vamos a crear una carpeta como ejemplo, y vamos a configurar todo correctamente.

Para este ejemplo, crearemos una carpeta llamada Compartido y asignamos los permisos requeridos. En este ejemplo, lo crearé en una subcarpeta de carpeta de usuario.
```
$ mkdir ~/smb/Compartido
# chmod 770 ~/smb/Compartido
```

Ahora vamos a editar el archivo `/etc/samba/smb.conf` una vez más, y añadiremos lo siguiente:
```
[compartido]
comment = Directorio publico compartido.
path = /home/user/smb/Compartido
public = yes
browseable = yes
writeable = yes
force group = users
directory mask = 0777
create mask = 0777
```

Ahora reiniciamos el servicio `smb`
```
# systemctl restart smb
```

Y, ahora, para verificar los archivos, directorios e impresoras compartidas, usamos:
```
# smbclient -L localhost
Password:
```
Solicitará una contraseña, no ingresamos ninguna, ya que no hemos definido una.

### Crear recursos privados (homes)

Para este ejemplo, vamos a crear el usuario `napo`.
```
# useradd napo -d /home/napo -m -s /bin/false
```

Ahora crearemos el usuario en samba, y la asgnamos la contraseña `test1234`
```
# pdbedit -a napo
# smbpasswd -a napo
```

Ahora verificamos la existencia del usuario en la base de datos de samba:
```
# pdbedit -L
```

Ahora crearemos recursos privados (homes) para todos los usuarios, en `/etc/samba/smb.conf`:
```
[homes]
comment = Carpetas Home de Usuarios valid users = %S
browseable = no
read only = no
```

Para aplizar los cambios, reiniciamos el servicio
```
# systemctl restart smb
```

Por último, para acceder al recurso creado mediante clientes Linux:
```
smbclient //localhost/publico -U jperez
```

### Crear un recurso compartido para un grupo de usuarios

Vamos a crear una cuenta de usuario y grupo para compartir un recurso:
```
# groupadd sistemas
# useradd -md /home/operador1 operador1 -G sistemas
# useradd -md /home/operador1 operador2 -G sistemas
# smbpasswd -a operador1
# smbpasswd -a operador2
```

Ahora vamos a crear un directorio de trabajo y asignar los permisos necesarios:
```
# mkdir /home/sistemas
# chmod 770 /home/sistemas
# chown root:sistemas /home/sistemas
```

Ahora vamos a crear un recurso compartido en `/etc/samba/smb.conf`:
```
[sistemas]
comment = Directorio Compartido para Grupo Sistemas
path = /home/sistemas
valid users = @sistemas write list = @sistemas force group = sistemas browseable = yes directory mask = 0770 create mask = 0770
```

Para aplizar los cambios, reiniciamos el servicio
```
# systemctl restart smb
```

## Probar sintaxis del archivo `/etc/samba/smb.conf`

```
testparm -v
```

## Agrega equipos al dominio generado por el servicio Samba para clientes Windows

Crear una cuenta en el sistema Linux para un equipo Windows
```
# useradd lab01-pc05$ -s /bin/false
```

Agregamos el usuario a la base de datos de Samba
```
# pdbedit –am lab01-pc05
# smbpasswd -am lab01-pc05
```

Ahora debemos realizar cambios en el registro del Windows 7 a unir en el dominio. Navegaremos hasta la ruta del registro:
```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\LammanWorkstation\Parameters
```

Agregamos los siguientes valores DWORD:
```
DomainCompatibilityMode = 1
DNSNameResolutionRequired = 0
```

Ahora navegamos hasta la ruta:
```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\Netlogon\Parameters
```

Y agregamos o modificamos los valores DWORD:
```
RequireSignalOrSeal = 1
RequireStrongKey = 1
```

Finalmente modifica en el registro del sistema los siguientes valores:
```
HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\Tcpip\Parameters
```
Ir a Panel de control → Sistema y Seguridad → Herramientas Administrativas → Directiva de seguridad local → Directivas locales → Opciones de seguridad.```
Domain = "empresa.lab(se abre en una nueva pestaña)" ICSDomain = "empresa.lab(se abre en una nueva pestaña)"
```

Ahora, editamos la Configuración Avanzada de TCP/IP de la interfaz a utilizar y establecemos manualmente la dirección IP del Servidor Samba como servidor WINS. Habilitamos la casilla de **Activar NetBIOS sobre TCP/IP**.

Vamos a `Panel de control → Sistema y Seguridad → Herramientas Administrativas → Directiva de seguridad local → Directivas locales → Opciones de seguridad.`
- En Seguridad de red: nivel de autenticación de LAN Manager, establecer enviar respuestas LM y NTML.
- En Seguridad de red: seguridad de sesión mínima para clientes NTML basados en SSP y Seguridad de red: seguridad de sesión mínima para servidores NTML basados en SSP, deshabilitar Requerir cifrado de 128-bit.
