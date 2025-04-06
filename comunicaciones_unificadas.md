# Guía para el curso de Comunicaciones Unificadas
* Guía hecha por Estrella Freundt.
* Este curso me fue dictado por el profesor Napoleón Cerna Rosas.
* Último edit: 06 de Abril de 2025.
* Obtener la última versión del archivo en: https://github.com/eeeeestr/network-administration/blob/main/comunicaciones_unificadas.md 

## Router

### Configurar una pool DHCP para la voz.
```
Router>enable
Router#configure terminal
Router(config)#ip dhcp pool VOICE
Router(config-dhcp)#option 150 ip 192.168.20.1
Router(config-dhcp)#default-router 192.168.20.1
Router(config-dhcp)#dns-server 192.168.20.2
Router(config-dhcp)#domain-name EMPRESA
Router(config-dhcp)#network 192.168.20.0 255.255.255.0
Router(config-dhcp)#exit
Router(config)#ip dhcp excluded-addresses 192.168.20.1 192.168.20.2
```

### Configurar servicio de telefonía
```
Router>enable
Router#configure terminal
Router(config)#license boot module c2900 technology-package uck9
Router(config)#reload
Router>enable
Router#configure terminal
Router(config)#telephony-service
Router(config-telephony)#auto assign (número mínimo de teléfonos ip) to (número máximo de teléfonos ip)
Router(config-telephony)#ip source-address 192.168.20.1 port 2000
Router(config-telephony)#keepalive 30
Router(config-telephony)#max-dn (número máximo de números en el directorio)
Router(config-telephony)#max-ephones (número máximo de teléfonos ip)
Router(config-telephony)#exit
Router(config)#ephone-dn 1
Router(config-ephone-dn)#number (número)
```

## Switch

### Configurar una vlan para uso de voz. Además, permitir la vlan a través del enlace troncal.
```
Switch>enable
Switch#config terminal
Switch(config)#vlan 20
Switch(config-vlan)#name VOICE
Switch(config-vlan)#exit
Switch(config)#interface <Enlace Troncal>
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk native vlan <vlan nativa>
Switch(config-if)#switchport voice vlan 20
Switch(config-if)#exit
```

### Configurar una vlan para uso de voz. Además, definiar los puertos por los que el servicio de voz será transmitida.
```
Switch>enable
Switch#config terminal
Switch(config)#vlan 20
Switch(config-vlan)#name VOICE
Switch(config-vlan)#exit
Switch(config)#interface range <interfaces donde viene la voz>
Switch(config-if-range)#switchport mode access
Switch(config-if-range)switchport voice vlan 20
```
