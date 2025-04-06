# Guía para el curso de Comunicaciones Unificadas
* Guía hecha por Estrella Freundt.
* Este curso me fue dictado por el profesor Napoleón Cerna Rosas.
* Último edit: 06 de Abril de 2025
Obtener la última versión del archivo en:
https://github.com/eeeeestr/network-administration/blob/main/comunicaciones_unificadas.md 

## Router

### Configurar una pool DHCP para la voz.
```
Switch>enable
Switch#configure terminal
Switch(config)#ip dhcp pool VOICE
Switch(config-dhcp)#option 150 ip 192.168.20.1
Switch(config-dhcp)#default-router 192.168.20.1
Switch(config-dhcp)#dns-server 192.168.20.2
Switch(config-dhcp)#domain-name EMPRESA
Switch(config-dhcp)#network 192.168.20.0 255.255.255.0
Switch(config-dhcp)#exit
Switch(config)#ip dhcp excluded-addresses 192.168.20.1 192.168.20.2
```

### Configurar servicio de telefonía
```
Switch>enable
Switch#configure terminal
Switch(config)#license boot module c2900 technology-package uck9
Switch(config)#reload
Switch>enable
Switch#configure terminal
Switch(config)#telephony-service
Switch(config-telephony)#auto assign (número mínimo de teléfonos ip) to (número máximo de teléfonos ip)
Switch(config-telephony)#ip source-address 192.168.20.1 port 2000
Switch(config-telephony)#keepalive 30
Switch(config-telephony)#max-dn (número máximo de números en el directorio)
Switch(config-telephony)#max-ephones (número máximo de teléfonos ip)
Switch(config-telephony)#exit
Switch(config)#ephone-dn 1
Switch(config-ephone-dn)#number (número)
```
