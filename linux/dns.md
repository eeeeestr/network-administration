# Configurar una Servidor DNS en RHEL o derivados.

## Fijarse la versión del sistema operativo

```
# cat /etc/os-release
```

## Actualizar el sistema e instalar las bind-utils

```
# dnf update -y
# dnf install bind bind-utils -y
```

## Configurar el archivo named.conf

```
# nano /etc/named.conf
```

Reemplazar `SERVER.IP.GOES.HERE` con la IP del servidor.
Reemplazar `NETWORKS.IP.GOES.HERE` con la IP de la puerta de enalce.
Reemplazar `miempre.sa` con el dominio de la empresa
Reemplazar `REVERSE.IP.HERE` con la IP del servidor en reversa. Es decir, si la IP del servidor es `192.168.0.5(/24)`, escribir `0.168.192.`

```
/
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        listen-on port 53 { SERVER.IP.GOES.HERE; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { NETWORKS.IP.GOES.HERE/24; };
      /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "miempre.sa" IN {
        type master;
        file "directa.miempre.sa";
        allow-update { none; };
};

zone "REVERSE.IP.HERE.in-addr.arpa" IN {
        type master;
        file "inversa.miempre.sa";
        allow-update { none; };
};
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

## Crear el archivo directo de zona

```
# nano /var/named/directa.miempre.sa
```

Reemplazar `SERVER.IP.GOES.HERE` con la IP del servidor.
Reemplazar `miempre.sa` con el dominio de la empresa
Reemplazar `WEBSERVER.IP.GOES.HERE` con la IP del webserver, en caso se tenga uno.

```
$TTL 86400
@   IN  SOA     ns1.miempre.sa.    admin.miempre.sa. (
                2025051301 ; Serial
                3600	   ; Refresh
                1800	   ; Retry
                604800     ; Expire
                86400	   ; Minimum TTL
)

; Servidores de nombre
@	IN  NS      ns1.miempre.sa.

; Registros A
ns1        IN  A       SERVER.IP.GOES.HERE
www     IN  A       WEBSERVER.IP.GOES.HERE
```

## Crear el archivo inverso de zona

```
# nano /var/named/inversa.miempre.sa
```

Reemplazar `miempre.sa` con el dominio de la empresa
Reemplazar `HOSTBYTES` con los bits de host de la IP que desea apuntar al dominio.
- Por ejemplo, para la IP del servidor `192.168.0.5/24`, debido a que la máscara de red es `255.255.255.0`, los bits de host sería todo después del tercer punto, en este caso `5`.
- Otro ejemplo, si la IP del servidor web es `192.168.0.56/24`, la parte de la IP correspondiente al host sería `56`.

```
$TTL 86400
@   IN  SOA     ns1.miempre.sa.      admin.miempre.sa. (
                2025051301 ; Serial
                3600	   ; Refresh
                1800	   ; Retry
                604800     ; Expire
                86400	   ; Minimum TTL
)

@	IN  NS      miempre.sa.
HOSTBYTES       IN  PTR     ns1.miempre.sa.
HOSTBYTES       IN  PTR     www.miempre.sa.
```


## Cambiar permisos del archivo de zona

```
# chown root:named /var/named/directa.miempre.sa
# chown root:named /var/named/inversa.miempre.sa
# chmod 640 /var/named/directa.miempre.sa
# chmod 640 /var/named/inversa.miempre.sa
```

## Abrir el puerto DNS en el firewall (UDP/TCP 53)

```
# firewall-cmd --permanent --add-port=53/udp
# firewall-cmd --permanent --add-port=53/tcp
# firewall-cmd --reload
```


## Habilitar y arrancar el servicio BIND

```
# systemctl enable named
# systemctl start named
```

## Verificar que esté corriendo

```
# systemctl status named
```

## Probar el servidor DNS

Reemplazar `SERVER.IP.GOES.HERE` con la IP del servidor.

```
dig @SERVER.IP.GOES.HERE ns1.miempre.sa
```
