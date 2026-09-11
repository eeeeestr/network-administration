# Laboratorio 2

## Actividad 1. Identidad, autenticación y autorización en MQTT
Instrucción: Analiza la configuración MQTT del escenario implementado en el Laboratorio 1 y aplica controles básicos para diferenciar dispositivos, rechazar accesos anónimos y limitar las operaciones permitidas.

1. **Preparación:** Copia la carpeta del laboratorio anterior para conservar una línea base y trabajar sobre una nueva versión:
```bash
$ cp -a ~/lab_iot_t1 ~/lab_iot_t2
$ cd ~/lab_iot_t2
$ docker compose down
```

2. Revisión inicial: Comprueba si el bróker permite conexiones sin credenciales y registra la evidencia:
```bsah
cat config/mosquitto.conf
docker compose up -d
docker exec -it broker-iot-t1 mosquitto_sub -h localhost -t 'frioandes/camara01/#' -v
```

Verifica si aparecen las directivas `listener 1883` y `allow_anonymous true`. Esta condición se utiliza únicamente como punto de partida del análisis.

4. **Matriz de identidades:** Registra las cuentas `sensor-temp-01`, `sensor-hum-01`, `gateway-camara01` y `monitor-frioandes`. Para cada una indica función, propietario y operaciones permitidas.

5. **Creación de credenciales:** Detén el contenedor y crea una contraseña diferente para cada identidad:
```bash
docker compose down
docker run --rm -it -v "$PWD/config:/work" eclipse-mosquitto:2 mosquitto_passwd -c
/work/passwd sensor-temp-01
docker run --rm -it -v "$PWD/config:/work" eclipse-mosquitto:2 mosquitto_passwd /work/passwd
sensor-hum-01
docker run --rm -it -v "$PWD/config:/work" eclipse-mosquitto:2 mosquitto_passwd /work/passwd
gateway-camara01
docker run --rm -it -v "$PWD/config:/work" eclipse-mosquitto:2 mosquitto_passwd /work/passwd
monitor-frioandes
```

No incluyas contraseñas reales ni muestres los valores completos en el informe.

6. **Autorización básica:** Crea `config/acl` para limitar los temas según la función del cliente:
```
user sensor-temp-01
topic write frioandes/camara01/raw/temperatura
user sensor-hum-01
topic write frioandes/camara01/raw/humedad
user gateway-camara01
topic read frioandes/camara01/raw/+
topic write frioandes/camara01/telemetria/+
topic write frioandes/camara01/comandos/alarma
user monitor-frioandes
topic read frioandes/camara01/telemetria/+
topic read frioandes/camara01/comandos/#
```

7. **Configuración segura inicial:** Actualiza `config/mosquitto.conf` con autenticación y permisos:
```
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl
persistence false
log_type error
log_type warning
log_type notice
```

Mantén el puerto publicado solo en 127.0.0.1 dentro de compose.yaml. El cifrado TLS se abordará en un laboratorio posterior.

8. **Validación:** Inicia el bróker y ejecuta las siguientes comprobaciones: acceso anónimo rechazado, publicación válida aceptada y publicación en un tema no autorizado rechazada.
```bash
$ docker compose up -d
# Acceso anónimo: debe ser rechazado
$ docker exec broker-iot-t1 mosquitto_pub -h localhost -t 'frioandes/camara01/raw/temperatura' -m '{"valor":8.5}'
# Acceso válido: debe ser aceptado
$ docker exec -it broker-iot-t1 mosquitto_pub -h localhost -u 'sensor-temp-01' -P 'CLAVE_TEMPORAL' -t 'frioandes/camara01/raw/temperatura' -m '{"sensor_id":"temp-01","valor":5.5}'
# Tema no autorizado: debe ser rechazado
$ docker exec -it broker-iot-t1 mosquitto_pub -h localhost -u 'sensor-temp-01' -P 'CLAVE_TEMPORAL' -t 'frioandes/camara01/raw/humedad' -m '{"sensor_id":"temp-01","valor":80}'
```

9. **Registro de resultados:** Completa una tabla con identidad, operación, resultado esperado, resultado obtenido y evidencia. Explica brevemente la diferencia entre autenticación y autorización.

## Actividad 2. Integridad del firmware y hardening básico del dispositivo virtual
**Instrucción:** Verifica la integridad del software que representa al gateway virtual, registra sus componentes y aplica medidas básicas de hardening sin afectar la funcionalidad del escenario.

1. **Versión de firmware simulada:** Copia el script del gateway como versión aprobada y calcula su hash de referencia:
```bash
$ mkdir -p firmware evidencias_t2
$ cp gateway/gateway_virtual.py firmware/firmware_gateway_v1.0.0.py
$ sha256sum firmware/firmware_gateway_v1.0.0.py | tee firmware/manifest_v1.0.0.sha256
$ sha256sum -c firmware/manifest_v1.0.0.sha256
```

3. **Comprobación de modificación:** Crea una copia alterada, agrega un comentario y compara los hashes. No ejecutes el archivo modificado:
```bash
$ cp firmware/firmware_gateway_v1.0.0.py firmware/firmware_gateway_alterado.py
$ echo '# cambio no autorizado de laboratorio' >> firmware/firmware_gateway_alterado.py
$ sha256sum firmware/firmware_gateway_v1.0.0.py firmware/firmware_gateway_alterado.py
```

4. **Registro de componentes:** Documenta las versiones del sistema y las dependencias utilizadas:
```
$ python3 --version | tee evidencias_t2/version_python.txt
$ docker --version | tee evidencias_t2/version_docker.txt
$ source .venv/bin/activate
$ python -m pip freeze | tee evidencias_t2/dependencias_python.txt
```

5. **Protección de credenciales:** Modifica los clientes virtuales para leer el usuario y la contraseña desde variables de entorno, en lugar de escribirlos en el código:
```python
import os
c.username_pw_set(os.environ['MQTT_USER'], os.environ['MQTT_PASS'])
```
Antes de ejecutar cada cliente, define MQTT_USER y MQTT_PASS con la identidad correspondiente. Al finalizar, elimina ambas variables con unset MQTT_USER MQTT_PASS.

7. **Hardening básico:** Aplica permisos restrictivos a scripts, firmware y archivos de credenciales; confirma que el bróker escuche únicamente en la interfaz local:
```bash
# chown 1883:1883 config/passwd config/acl
# chmod 640 config/passwd config/acl
# chmod 750 sensores gateway firmware
$ find sensores gateway firmware -type f -exec chmod 640 {} \;
$ ss -lntp | grep 1883
```

8. **Registros y funcionamiento:** Revisa los eventos del bróker y comprueba que los sensores, gateway y monitor sigan operativos con identidades diferenciadas:
```bash 
$ docker logs broker-iot-t1 --tail 30 | tee evidencias_t2/log_mosquitto.txt
```
10. **Lista de verificación:** Confirma que el acceso anónimo esté deshabilitado, cada cliente posea identidad propia, las contraseñas no estén en el código, los permisos se encuentren limitados, el firmware tenga versión y hash, y los errores de autenticación queden registrados.

11. **Cierre técnico:** Explica por qué una identidad única mejora la trazabilidad, qué riesgo presenta una credencial por defecto, por qué debe validarse una actualización y qué medidas de hardening redujeron la superficie de ataque.

## Entregable

El estudiante deberá presentar un Informe de Laboratorio 2 (PDF) que contenga:
1. Configuración inicial: evidencia del acceso anónimo y explicación del riesgo.
2. Matriz de identidades: cuenta, componente, función y operaciones autorizadas.
3. Autenticación y autorización: capturas de una conexión rechazada, una operación permitida y una operación denegada.
4. Firmware: versión, hash aprobado, validación de integridad y comparación con el archivo alterado.
5. Componentes: versiones de las herramientas y listado de dependencias utilizadas.
6. Hardening: lista de verificación con el estado de cada control aplicado.
7. Evidencia funcional: sensores, gateway, bróker y monitor operando con identidades diferenciadas.
8. Análisis técnico: respuestas a las preguntas de cierre y dos conclusiones sobre identidad, firmware y configuración segura.

**Nota:** Conserva las carpetas `~/lab_iot_t1` y `~/lab_iot_t2`. La primera representa el escenario inicial y la segunda, el escenario con controles básicos. Ambas serán reutilizadas en las siguientes guías.
