# laboratorio 1

## act 1. implementación del iot virtual

1. instalar los programitas
```bash
$ curl -fsSL https://get.docker.com | sudo sh
$ sudo apt install -y uidmap
$ dockerd-rootless-setuptool.sh install
$ sudo apt install -y python3 python3-pip python3-venv
```

2. **Verificación del entorno:** Comprueba que Docker, Docker Compose y Python estén disponibles:
```bash
$ docker --version
$ python3 --version
```

3. **Estructura del laboratorio:** Crea una carpeta independiente para conservar scripts, configuraciones y evidencias:
```bash
$ mkdir -p ~/lab_iot_t1/{config,sensores,gateway,edge_ai,evidencias}
$ cd ~/lab_iot_t1
```

4. **Bróker MQTT:** Crea el archivo config/mosquitto.conf con la siguiente configuración controlada:
```bash
$ listener 1883
$ allow_anonymous true
$ persistence false
$ log_type notice
```
**Advertencia:** el acceso anónimo se habilita solo para observar una condición insegura dentro del laboratorio. El puerto quedará publicado únicamente en la interfaz local.

5. **Contenedor del bróker:** Crea compose.yaml y ejecuta Mosquitto:
```bash
$ touch ~/lab_iot_t1/compose.yaml
```
y agregamos:
```docker
services:
  broker:
    image: eclipse-mosquitto:2
    container_name: broker-iot-t1
    ports:
      - "127.0.0.1:1883:1883"
    volumes:
      - ./config/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
```
y corremos en el directorio
```bash
$ docker compose up -d
```

6. **Entorno Python:** Crea un entorno virtual e instala la biblioteca MQTT:
```bash
$ python3 -m venv .venv
$ source .venv/bin/activate
$ python -m pip install --upgrade pip paho-mqtt
```

7. **Sensor virtual:** Crea `sensores/sensor_virtual.py`. El programa publicará doce mediciones en un tema raw:
```python
import argparse, json, random, time
from datetime import datetime, timezone
import paho.mqtt.client as mqtt

p = argparse.ArgumentParser()
p.add_argument("--id", required=True)
p.add_argument("--tipo", choices=["temperatura", "humedad"], required=True)
p.add_argument("--minimo", type=float, required=True)
p.add_argument("--maximo", type=float, required=True)
a = p.parse_args()

c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2, client_id=a.id)
c.connect("127.0.0.1", 1883, 60)
c.loop_start()
topic = f"frioandes/camara01/raw/{a.tipo}"

for _ in range(12):
    dato = {"sensor_id": a.id, "tipo": a.tipo,
            "valor": round(random.uniform(a.minimo, a.maximo), 2),
            "timestamp": datetime.now(timezone.utc).isoformat()}
    c.publish(topic, json.dumps(dato), qos=0).wait_for_publish()
    print(topic, dato)
    time.sleep(3)

c.loop_stop()
c.disconnect()
```

8. **Gateway lógico:** Crea `gateway/gateway_virtual.py`. El gateway valida los rangos, publica la telemetría aceptada y activa una alarma cuando la temperatura supera 7 °C:
```python
import json
from datetime import datetime, timezone
import paho.mqtt.client as mqtt

def recibir(c, datos_usuario, mensaje):
    try:
        dato = json.loads(mensaje.payload.decode("utf-8"))
        tipo, valor = dato["tipo"], float(dato["valor"])
        valido = (-20 <= valor <= 20) if tipo == "temperatura" else (0 <= valor <= 100)
        if not valido:
            raise ValueError("valor fuera del rango esperado")

        salida = mensaje.topic.replace("/raw/", "/telemetria/")
        c.publish(salida, json.dumps(dato), qos=0)
        print("ACEPTADO:", salida, dato)

        if tipo == "temperatura" and valor > 7:
            alarma = {"actuador_id": "alarma-01", "accion": "activar",
                      "motivo": "temperatura_alta",
                      "timestamp": datetime.now(timezone.utc).isoformat()}
            c.publish("frioandes/camara01/comandos/alarma", json.dumps(alarma), qos=0)
    except (KeyError, ValueError, TypeError, json.JSONDecodeError) as e:
        print("RECHAZADO:", mensaje.topic, e)

c = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2, client_id="gateway-camara01")
c.on_message = recibir
c.connect("127.0.0.1", 1883, 60)
c.subscribe("frioandes/camara01/raw/+")
print("Gateway virtual activo...")
c.loop_forever()
```

10. **Validación del actuador:** Si no se genera una alerta durante las mediciones, publica una temperatura controlada de 8.5 °C y comprueba el mensaje enviado a comandos/alarma:
```docker
docker exec broker-iot-t1 mosquitto_pub -h localhost   -t 'frioandes/camara01/raw/temperatura'   -m '{"sensor_id":"temp-01","tipo":"temperatura","valor":8.5}'
```

11. **Arquitectura:** Elabora un diagrama con las cinco capas y el flujo de datos: sensores y alarma (dispositivo), conexión local/Docker (red), gateway_virtual.py (gateway), Mosquitto (plataforma) y suscriptor de monitoreo (aplicación).

## Actividad 2. Inventario, superficie de ataque y análisis de riesgos

Instrucción: Identifica los activos del escenario, incorpora un archivo representativo de Edge AI, reconoce los puntos de exposición y propone controles iniciales sin intervenir sistemas externos.

1. **Activo Edge AI:** Crea un archivo de metadatos que represente el modelo utilizado para detectar vibraciones anómalas y registra su hash de referencia:
```bash
$ cat > edge_ai/modelo_anomalias_v1.json <<'EOF'
{"modelo":"deteccion_vibracion_compresor","version":"1.0","umbral":0.82}
EOF
sha256sum edge_ai/modelo_anomalias_v1.json | tee evidencias/hash_modelo.txt
```
2. **Comprobación de integridad:** Crea una copia, modifica el umbral y compara los hashes para demostrar que un cambio en los parámetros altera la integridad del activo:
```bash
$ cp edge_ai/modelo_anomalias_v1.json edge_ai/modelo_anomalias_modificado.json
$ sed -i 's/0.82/0.20/' edge_ai/modelo_anomalias_modificado.json
$ sha256sum edge_ai/modelo_anomalias*.json
```

3. **Inventario de activos:** Registra al menos diez activos con los campos ID, capa, activo, ubicación o archivo, responsable, versión, criticidad y evidencia. Incluye los sensores, la alarma, el gateway, el bróker, los temas MQTT, la telemetría, la imagen Docker, los scripts y el modelo Edge AI.

4. **Superficie de ataque:** Identifica los servicios en escucha y revisa únicamente los puertos locales autorizados:

```bash
$ ss -lntp | grep 1883
$ nmap -sV -p 1883,1880,8080 127.0.0.1
$ docker logs broker-iot-t1 --tail 20
```

5. **Análisis de exposición:** Documenta para cada punto: activo, interfaz o puerto, necesidad, autenticación, permisos, evidencia e impacto. Distingue entre exposición, vulnerabilidad y riesgo. El puerto 1883 abierto es una exposición; el acceso anónimo constituye una configuración insegura dentro del escenario.

6. **Riesgos y controles:** Elabora una matriz con al menos cinco hallazgos. Considera acceso anónimo, ausencia de cifrado, identidades no diferenciadas, modificación del modelo, permisos de archivos y registros insuficientes. Para cada hallazgo indica impacto, prioridad y control propuesto: credenciales únicas, ACL, TLS, permisos mínimos, hash/versionado, segmentación y monitoreo.

7. **Cierre técnico:** Responde en un párrafo por pregunta: ¿por qué un puerto abierto no es automáticamente una vulnerabilidad?, ¿qué activo es más crítico en este escenario?, ¿cómo amplía Edge AI la superficie de ataque? y ¿qué controles deberían implementarse primero?

## Entregable

El estudiante deberá presentar un Informe de Laboratorio 1 (PDF) que contenga:
1. Arquitectura: Diagrama de las cinco capas con sensores, actuador, gateway, bróker, temas MQTT, aplicación y dirección del flujo de datos.
2. Evidencia funcional: Capturas del contenedor activo, del gateway validando mensajes, de los dos sensores publicando y del suscriptor recibiendo telemetría y la orden de alarma.
3. Inventario: Tabla de activos con al menos diez registros, incluyendo firmware o scripts, credenciales previstas, datos, servicios y el modelo Edge AI con su hash.
4. Superficie de ataque: Salida de ss/nmap y tabla de puntos de exposición, diferenciando exposición, vulnerabilidad y riesgo.
5. Matriz de riesgos: Cinco hallazgos priorizados con impacto y controles iniciales de mitigación.
6. Análisis técnico: Respuestas a las cuatro preguntas de cierre y dos conclusiones sobre la protección integral del ecosistema.

Nota: conserva la carpeta ~/lab_iot_t1 y las evidencias obtenidas. Este escenario será reutilizado y endurecido progresivamente en las siguientes guías. Para detener el entorno ejecuta docker compose down.
