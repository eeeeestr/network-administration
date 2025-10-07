# Servidor LDAP

Este archivo te guiará en la instalación y configuración básica de un servidor LDAP en Red Hat 10 o derivados.

## Instalación

Instalamos el paquete `389-ds-base`
```
# dnf install 389-ds-base -y
```

Creamos una instancia LDAP
```
# dscreate interactive
Install Directory Server (interactive mode)
===========================================

Enter system's hostname [localhost.localdomain]: ejemplo.lab

Enter the instance name [cibertec]: servidorldap

Enter port number [389]:

Create self-signed certificate database [yes]:

Enter secure port number [636]:

Enter Directory Manager DN [cn=Directory Manager]:

Enter the Directory Manager password:
Confirm the Directory Manager Password:

Choose whether mdb or bdb is used. [mdb]:

Enter the lmdb database size [9.2 GB]:

Enter the database suffix (or enter "none" to skip) [dc=ejemplo,dc=lab]:

Create sample entries in the suffix [no]:

Create just the top suffix entry [no]:

Do you want to start the instance after the installation? [yes]:

Are you ready to install? [no]: yes
Starting installation ...
Validate installation settings ...
Create file system structures ...
Create self-signed certificate database ...
Perform SELinux labeling ...
Create database backend: dc=cibertec,dc=edu ...
Perform post-installation tasks ...
Completed installation for instance: slapd-servidorldap
```

Verificamos el estado del servicio
```
# systemctl status dirsrv@servidorldap
● dirsrv@servidorldap.service - 389 Directory Server servidorldap.
     Loaded: loaded (/usr/lib/systemd/system/dirsrv@.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/dirsrv@.service.d
             └─custom.conf
     Active: active (running) since Fri 2025-10-03 19:40:16 -05; 3min 54s ago
 Invocation: 4bc2cc85aff0428388b84ed4a31d9d13
    Process: 9940 ExecStartPre=/usr/libexec/dirsrv/ds_systemd_ask_password_acl /etc/dirsrv/slapd-servidorldap/dse.ldif (code=exited, status=0/SUCCESS)
    Process: 9945 ExecStartPre=/usr/libexec/dirsrv/ds_selinux_restorecon.sh /etc/dirsrv/slapd-servidorldap/dse.ldif (code=exited, status=0/SUCCESS)
   Main PID: 9951 (ns-slapd)
     Status: "slapd started: Ready to process requests"
      Tasks: 24 (limit: 22920)
     Memory: 64.1M (peak: 64.5M)
        CPU: 1.176s
     CGroup: /system.slice/system-dirsrv.slice/dirsrv@servidorldap.service
             └─9951 /usr/sbin/ns-slapd -D /etc/dirsrv/slapd-servidorldap -i /run/dirsrv/slapd-servidorldap.pid

Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.639893403 -0500] - NOTICE - attrcrypt_cipher_init - No symmetric key found for cipher AES in backend userroot, attempting to create >
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.645957086 -0500] - INFO - attrcrypt_cipher_init - Key for cipher AES successfully generated and stored
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.649734412 -0500] - NOTICE - attrcrypt_cipher_init - No symmetric key found for cipher 3DES in backend userroot, attempting to create>
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.655466215 -0500] - INFO - attrcrypt_cipher_init - Key for cipher 3DES successfully generated and stored
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.660808636 -0500] - INFO - slapd_daemon - New referral entries are detected under dc=cibertec,dc=edu (returned to SRCH req)
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.722927207 -0500] - INFO - connection_table_new - Number of connection sub-tables 1, each containing 63937 slots.
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.758115096 -0500] - INFO - slapd_daemon - slapd started.  Listening on All Interfaces port 389 for LDAP requests
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.761926392 -0500] - INFO - slapd_daemon - Listening on All Interfaces port 636 for LDAPS requests
Oct 03 19:40:16 localhost.localdomain ns-slapd[9951]: [03/Oct/2025:19:40:16.765405685 -0500] - INFO - slapd_daemon - Listening on /run/slapd-servidorldap.socket for LDAPI requests
Oct 03 19:40:16 localhost.localdomain systemd[1]: Started dirsrv@servidorldap.service - 389 Directory Server servidorldap..
```

Verificamos que los puertos estén escuchando
```
$ ss -tulpn | grep 389
tcp   LISTEN 0      128                                    *:389              *:*
$ ss -tulpn | grep 636
tcp   LISTEN 0      128                                    *:636              *:*
```

Ahora creamos la entrada raíz
```
$ mkdir ldap
$ cd ldap
$ nano root.ldif
```
e insertamos lo siguiente:
```
dn: dc=ejemplo,dc=lab
objectClass: top
objectClass: domain
dc: cibertec
description: Dominio raíz de ejemplo.lab
```

Ahora añadimos la entrada
```
# ldapadd -x -D "cn=Directory Manager" -W -f root.ldif
Enter LDAP Password:
adding new entry "dc=ejemplo,dc=lab"
```

Vamos a probar la conección
```
$ ldapsearch -x -b "dc=ejemplo,dc=lab"
# extended LDIF
#
# LDAPv3
# base <dc=cibertec,dc=edu> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 0 Success

# numResponses: 1
```

## Entradas

Ahora vamos a añadir una entrada. En este ejemplo, usaremos el dos objetos:
- La unidad organizacional `Personas`
- El usuario de Napoleón Cerna, con userID `napo` y la contraseña ejemplo1234

Primero, para generar la contraseña "ejemplo1234" encriptada:
```
$ python3 -c 'import hashlib, base64, os; pw = b"ejemplo1234"; salt = os.urandom(4); digest = hashlib.sha1(pw + salt).digest() + salt; print("{SSHA}" + base64.b64encode(digest).decode())'
{SSHA}KF2OahWXSdX1wmdp45iNTLo/uvrXIaQb
```

Ahora que tenemos el hash de la contraseña, vamos a crear un archivo en donde guardar estas entradas. En este caso, será el archivo `base.ldif`.
```
$ nano base.ldif
```

y entramos lo siguiente:
```
dn: ou=People,dc=ejemplo,dc=lab
objectClass: organizationalUnit
ou: Personas

dn: uid=napo,ou=Personas,dc=ejemplo,dc=lab
objectClass: inetOrgPerson
uid: napo
sn: Cerna
givenName: Napoleon
cn: Napoleon Cerna
mail: napo@ejemplo.lab
userPassword: {SSHA}KF2OahWXSdX1wmdp45iNTLo/uvrXIaQb
```

Ahora vamos a cargar el archivo `base.ldif`
```
$ ldapadd -x -D "cn=Directory Manager" -W -f base.ldif
Enter LDAP Password:
adding new entry "ou=Personas,dc=ejemplo,dc=lab"

adding new entry "uid=napo,ou=Personas,dc=ejemplo,dc=lab"
```

Ahora vamos a verificar que el archivo haya sido importado correctamente
```
$ ldapsearch -x -b "dc=cibertec,dc=edu" "(uid=napo)"
# extended LDIF
#
# LDAPv3
# base <dc=cibertec,dc=edu> with scope subtree
# filter: (uid=napo)
# requesting: ALL
#

# search result
search: 2
result: 0 Success

# numResponses: 1
```

