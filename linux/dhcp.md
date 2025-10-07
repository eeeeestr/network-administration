# DHCP

Este archivo te guiará por el proceso de instalación de un servidor DHCP en Red Hat 10 y derivados.

## Instalación

Instalamos `dnsmasq`.
```
# dnf install dnsmasq
```

Editamos el archivo de configuración de `dnsmasq`
```
# nano /etc/dnsmasq.conf
```

Añadimos lo siguiente:
```
port=0  # Desactiva el servidor DNS interno de dnsmasq
interface=ens33  # Cambiar por tu interfaz real
bind-interfaces
dhcp-range=192.168.38.100,192.168.38.150,12h # Rango de IPs, tiempo de préstamo de IP.
dhcp-option=3,192.168.38.1  # Gateway
dhcp-option=6,192.168.38.131 # DNS que se enviará a los clientes
```

Habilitamos el servicio `dnsmasq`
```
sudo systemctl enable --now dnsmasq
```

Permitimos el servicio de dhcp por `firewall.d`.
```
sudo firewall-cmd --permanent --add-service=dhcp
sudo firewall-cmd --reload
```

Ahora ya tenemos un servidor DHCP configurado correctamente.

## Modo Debug, para correr sin daemon.
```
sudo dnsmasq --no-daemon
```
