Guía para el curso de Enrutamiento y Conmutación de Redes
(CCNA II).
Guía hecha por Estrella Freundt.
Este curso me fue dictado por el profesor Napoleón Cerna Rosas.
Último edit: 16 de Enero de 2025
Obtener la última versión del archivo en:
https://github.com/efrndt/network-administration/blob/main/network-routing-and-switching/guia_ccna2.txt 

~~~~~~~~~~~~~~~~~~~~~~~~~~
~~~~~~~~~~SWITCH~~~~~~~~~~
~~~~~~~~~~~~~~~~~~~~~~~~~~

## Básico: IPv4 e IPv6

### Para cambiar la ipv4 de una vlan:
```
Switch>enable
Switch#configure terminal
Switch(config)#interface <vlan>
Switch(config-if)#ip address <ip> <máscara>
Switch(config-if)#no shutdown
```

### Para poner gateway por defecto:
```
Switch>enable
Switch#configute terminal
Switch(config)#ip default-gateway <ip>
```

### Para cambiar la ipv6 de una vlan:
```
Switch>enable
Switch#config terminal
Switch(config)#sdm prefer dual-ipv4-and-ipv6 default
Swtich(config)#int <interface donde estará el router>
Switch(config-if)#switchport mode trunk
Switch(config-if)#exit
Switch(config)#int <vlan>
Switch(config-if)#ipv6 address <ip>/<prefijo>
Switch(config-if)#no shutdown
```
### Consultar estado de interfaces ipv4 e ipv6
```
Switch>enable
Switch>show ip interface brief
Switch>show ipv6 interface brief
```

## VLans

### Crear una vlan de datos (vlan 10) y renombrarla a "OPERACIONES"
```
Switch>enable
Switch#config terminal
Switch(config)#vlan 10
Switch(config-vlan)#name OPERACIONES
```

### Asignarle puertos a una vlan (vlan 20)
```
Switch>enable
# Mostar las vlans existentes:
Switch#show vlan
# Mostrar los puertos disponibles con ipv4
Switch#show ip interface brief
Switch#config terminal
Switch(config)#interface range fa0/2-8
Switch(config-if-range)switchport mode access 
Switch(config-if-ramnge)swtich access vlan 20
```

### Crear una vlan administrativa

```
Switch>enable
Switch#config terminal
Switch(config)#vlan 90
Switch(config-vlan)#name ADMINISTRATIVA
Switch(config-vlan)#exit
# Ahora le asignaremos una dirección IP a la vlan administrativa.
# Recordar que la única vlan del switch que debe tener una IP debe de ser la vlan administrativa, de otra forma, se expone a riesgos de malfuncionamiento y seguridad.
Switch(config)interface vlan 90
Switch(config-if)#ip address 192.168.90.5 255.255.255.0
Switch(config-if)#no shutdown
# La vlan administrativa no debe de tener ningún puerto asignado en una situación ideal. La única forma a entrar a esta vlan deberá de ser por SSH.
```

### Crear una interface troncal

```
# En este ejemplo, usaremos la interfaz GigabitEthernet0/1. Permitiremos que por la troncal pasen las vlans 10, 20, y 90.
Switch>enable
Switch#configure terminal
Switch(config)#interface GigabitEthernet0/1
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk allow vlan 10,20,90
# Además, haremos que la vlan 90 sea una vlan nativa.
Switch(config-if)#switchport trunk native vlan 90
# Para verificar si la interfaz se ha configurado correctamente:
Switch#show interface trunk
```

## Spanning tree

### Reconocer el switch raíz

```
Switch>enable
Switch#show spanning-tree
# Si todos los puertos son **designados**, entonces usted se encuentra en el switch raíz
```

### Definir switch como switch raiz de la vlan 1
```
Switch>enable
Switch#config terminal
Switch(config)#spanning tree vlan 1 root primary
```

### Definir prioridades de los switches para la vlan 1
```
Switch>enable
Switch#config terminal
# Las prioridades van de 4096 en 4096.
Switch(config)#spanning-tree vlan 1 priority 4096
```


## Router

### Para cambiar la ipv4 de una interfaz:
```
Router>enable
Router#configure terminal
Router(config)#interface <interfaz>
Router(config-if)#ip address <ip> <máscara>
Router(config-if)#no shutdown
```
### Para cambiar la ipv6 de una interfaz:
```
  Router>enable
  Router#configure terminal
  Router(config)#ipv6 unicast-routing
  Router(config)#interface <interfaz>
  Router(config-if)#ipv6 enable
  Router(config-if)$#ipv6 address <ip>/<prefijo>
  Router#(config-if)#no shutdown
```
### Establecer servidor SSH
```
  Router>enable
  Router#config terminal
# Poner el switch en un dominio
  Router(config)#ip domain-name <nombre.dominio>
# Generar la crypto key
  Router(config)#crypto key generate rsa 
	How many bits in the modulus [512]: 2048
# Por defecto, el swtich que utilizo biene con ssh versión 1.
# Este comando cambia a la ssh versión 2.
  Router(config)#ip ssh version 2
  Router(config)#line vty 0 4
  Router(config-if)#transport input shh
  Router(config-if)#login local
  Router(config-if)#exit
  Router(config)#username <nombre de usuario> password <contraseña>
# Darles privilegios de administrador al usuario
  Router(config)#username <nombre de usuario> privilege 15 
# Encriptar las contraseñas
  Router(config)#service password-encryption 
```
### Configurar una subinterfaz para una vlan de datos.
```
# Para este ejemplo, usaremos el puerto GigabitEthernet0/0, para
# crear la subinterfaz GigabitEthernet0/0/0.10.
  Router>enable
  Router#configure terminal
# Seleccionamos la interface GI0/0.10
  Router(config)#interface GigabitEthernet0/0.10
# Configuramos el encapsulamiento intra VLan
  Router(config-subif)#encapsulation dot1Q 10
# Definimos la subred
  Router(config-subif)#ip address 192.168.10.1 255.255.255.0
  Router(config-subif)#exit
# Haremos lo mismo con la interface GI0/0.20
  Router(config)#interface gi0/0.20
  Router(config-subif)#encapsulation dot1Q 20
  Router(config-subif)#ip address 192.168.20.1 255.255.255.0
  Router(config-subif)#exit
```

### Configurar una subinterface para una vlan nativa.
```
  Router>enable
  Router#config terminal
# Seleccionaremos la interface GI0/0.90
  Router(config)#interface gi0/0.90
# Configuramos el encapsulamiento inter VLan, especificando
# que la encapsulación sea para una vlan nativa.
  Router(config-subif)#encapsulation dot 1Q 90 native
  Router(config-subif)#ip address 192.168.90.1 255.255.255.0
  Router(config-subif)#exit
  Router(config)#interface GigabitEthernet0/0
  Router(config-if)#no shutdown
```
