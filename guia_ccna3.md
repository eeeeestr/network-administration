# Guía para el curso de Redes Escalables (CCNA I(I).
* Guía hecha por Estrella Freundt.
* Este curso me fue dictado por el profesor Napoleón Cerna Rosas.
* Último edit: 4 de Junio de 2025
* Obtener la última versión del archivo en:
https://github.com/eeeeestr/network-administration/blob/main/guia_ccna3.md 

## OSPF 

### Configurar una red en OSPF.
```
Router>enable
Router#config terminal
Router(config)#router ospf <1-65535>
Router(config-router)#router-id <id de 32 bits, por ejemplo 1.1.1.1>
Router(config-router)#network <red a propagar. por ejemplo, 192.168.1.1> <wildcard, por ejemplo, 0.0.0.255> area <área de ospf, por ejemplo, 0>
Router(config-router)#passive-interface <interfaz pasiva>
Router(config-router)#
```

### Activar una interfaz en OSPF.
```
Router>enable
Router#config terminal
Router(config)#interface <interfaz a activar OSPF en>
Router(config-if)#ip ospf <1-65535> area <área>
```
