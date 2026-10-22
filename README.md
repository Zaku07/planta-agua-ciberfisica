# Planta de Tratamiento de Agua Potable Virtual — Gemelo Digital Ciberfísico

**PTSC08 Sistemas Ciberfísicos · Evaluación Sumativa 2/3 · INACAP Sede Temuco**

Emulación completa de una planta de agua potable sin hardware físico: gemelo digital por API REST,
soft-sensors y soft-actuators, PLC Modbus TCP, HMI local, telemetría desacoplada por MQTT, sonda IDS
pasiva sobre puerto SPAN, segmentación Purdue con FortiGate en GNS3 y analítica en AWS Academy
(EC2 + InfluxDB + Grafana).

## Los 9 contenedores

|#|Contenedor|Nivel Purdue|Script|Función|
|-|-|-|-|-|
|1|`estanque-agua`|0|`estanque\\\\\\\\\\\\\\\_agua.py`|Gemelo digital: nivel (Delta Time) y cloro (ppm) por API REST|
|2|`plc-central`|2|`plc\\\\\\\\\\\\\\\_central.py`|Servidor Modbus TCP, lazo de control con histéresis 20/90 %, alarmas, watchdog|
|3|`sensor-nivel-estanque`|1|`sensor\\\\\\\\\\\\\\\_nivel.py`|GET `/nivel` y `/cloro` → escribe HR 40001-40002 (FC16)|
|4|`actuador-bomba-agua`|1|`actuador\\\\\\\\\\\\\\\_bomba.py`|Lee coil 00001 (FC01) → POST `/bomba`|
|5|`panel-hmi-local`|3|`panel\\\\\\\\\\\\\\\_hmi.py`|HMI web de solo lectura (puerto 8080)|
|6|`gateway-borde`|3|`gateway\\\\\\\\\\\\\\\_borde.py`|Polling Modbus 1 s → publica MQTT (telemetría y eventos de alarma)|
|7|`broker-mqtt`|3|`mosquitto.conf`|Eclipse Mosquitto 2.0 con usuarios, claves y ACL por tópico|
|8|`sonda-ids`|3|`sonda\\\\\\\\\\\\\\\_ids.py`|IDS pasivo Scapy/DPI Modbus sobre puerto SPAN|
|9|`puente-aws`|3.5 (DMZ)|`puente\\\\\\\\\\\\\\\_aws.py`|MQTT → HTTPS/TLS con certificate pinning y store-and-forward hacia EC2|

Herramienta de prueba (no es parte de los 9): `herramientas/atacante-modbus` genera tramas no autorizadas
para las evidencias de FortiGate e IDS.

## Mapa de memoria del PLC

|Dirección|Tipo|Variable|Escribe|Lee|
|-|-|-|-|-|
|40001 (HR 0)|Holding Register|Nivel del estanque (%, 0-100)|sensor|gateway, HMI|
|40002 (HR 1)|Holding Register|Cloro libre residual (ppm × 100)|sensor|gateway, HMI|
|00001 (Coil 0)|Coil|Bomba de impulsión (1 = activa)|PLC|actuador, gateway, HMI|
|00002 (Coil 1)|Coil|Alarma nivel alto (≥ 90 %)|PLC|gateway, HMI|
|00003 (Coil 2)|Coil|Alarma nivel bajo (≤ 20 %)|PLC|gateway, HMI|
|00004 (Coil 3)|Coil|Alarma cloro bajo (< 0.2 ppm)|PLC|gateway, HMI|
|00005 (Coil 4)|Coil|Alarma cloro alto (> 4.0 ppm)|PLC|gateway, HMI|
|00006 (Coil 5)|Coil|Falla del sensor (watchdog 5 s)|PLC|gateway, HMI|

## Modelo físico

```
Nivel(t+Δt) = Nivel(t) + Δt · (TASA\\\\\\\\\\\\\\\_ENTRADA − TASA\\\\\\\\\\\\\\\_EXTRACCION · Estado\\\\\\\\\\\\\\\_Bomba)
TASA\\\\\\\\\\\\\\\_ENTRADA = 3.0 %/s   TASA\\\\\\\\\\\\\\\_EXTRACCION = 6.0 %/s   →  ±3.0 %/s netos
Ciclo 20 % → 90 % → 20 %:  2 × 70/3 = 46.7 s teóricos (≈ 48 s medidos)

C(t+Δt) = C(t) + Δt · ( (TASA\\\\\\\\\\\\\\\_ENTRADA / Nivel) · (C\\\\\\\\\\\\\\\_dosis − C(t)) − k · C(t) )
C\\\\\\\\\\\\\\\_dosis normal = 1.2 ppm,  k = 0.002 s⁻¹  →  cloro estable ≈ 1.1–1.2 ppm
```

Fallas de cloro para la demo: `POST /dosificador {"modo": "bajo" | "alto" | "normal"}` o variable
`DEMO\\\\\\\\\\\\\\\_FALLAS\\\\\\\\\\\\\\\_CLORO\\\\\\\\\\\\\\\_CADA=120` (alterna fallas automáticamente cada 120 s).

## 1\. Prueba local (Docker Compose)

```bash
cp .env.example .env          # completar claves
docker compose up --build -d
docker logs -f plc-central    # evidencia Módulo 1: conmutaciones 90 % / 20 % y periodo del ciclo
# Abrir http://localhost:8080 en el navegador (HMI local)
```

Prueba de intrusión (evidencia Módulo 3):

```bash
docker compose --profile ataque run --rm atacante-modbus python atacante\\\\\\\\\\\\\\\_modbus.py --plc 192.168.10.20 escribir-coil
docker logs sonda-ids         # \\\\\\\\\\\\\\\[!!! ALERTA DE INTRUSIÓN IDS !!!]
```

Inyección de falla de cloro (desde un contenedor de la red de campo):

```bash
docker exec sensor-nivel-estanque python -c "import requests;print(requests.post('http://estanque-agua:5000/dosificador',json={'modo':'bajo'}).text)"
# volver a la normalidad: {'modo':'normal'}
```

## 2\. Topología GNS3 + FortiGate

Ver [`red/gns3/README.md`](red/gns3/README.md) (plan IP, nodos, variables) y las configuraciones:

* `red/switches/sw-campo.txt` — IOSvL2 de Niveles 0-2, VLAN 10, puerto SPAN hacia la sonda.
* `red/switches/sw-operaciones.txt` — IOSvL2 de Nivel 3 / DMZ, trunk hacia el FortiGate.
* `red/fortigate/fortigate.conf` — VLANs, objetos, servicios industriales y 5 políticas con log.
* `red/fortigate/fortigate-compacto.conf` — variante para la licencia de evaluación limitada.

## 3\. Nube AWS Academy

En la EC2 (Ubuntu 22.04, t3.small o superior; Security Group con 22 y 443 abiertos):

```bash
git clone <repo> \\\\\\\\\\\\\\\&\\\\\\\\\\\\\\\& cd planta-agua-ciberfisica/nube-aws
./preparar\\\\\\\\\\\\\\\_ec2.sh                       # instala Docker (cerrar y reabrir SSH)
cp .env.example .env \\\\\\\\\\\\\\\&\\\\\\\\\\\\\\\& nano .env       # claves y token admin (openssl rand -hex 32)
./generar\\\\\\\\\\\\\\\_certificado.sh                # muestra la huella SHA-256 -> AWS\\\\\\\\\\\\\\\_HUELLA\\\\\\\\\\\\\\\_CERT
docker compose up -d
./crear\\\\\\\\\\\\\\\_token\\\\\\\\\\\\\\\_puente.sh                 # token de SOLO escritura -> INFLUX\\\\\\\\\\\\\\\_TOKEN\\\\\\\\\\\\\\\_PUENTE
```

Dashboard: `https://<IP-EC2>/` (usuario `admin`). El certificado es autofirmado: el navegador mostrará
una advertencia, pero el puente-aws valida la huella exacta. Paneles incluidos:

* Nivel del estanque en tiempo real (%) con bandas de umbral 20 % / 90 %.
* Estado operativo de la bomba (ACTIVA verde / INACTIVA rojo) e historial.
* Cloro libre residual (ppm) con zonas críticas < 0.2 y > 4.0, y valor actual.
* Registro de alertas de seguridad (Safety Alarms) con fecha, tipo, severidad y estado.
* Alertas de intrusión OT de la sonda-ids.

## Medidas de ciberseguridad implementadas

|Capa|Medida|
|-|-|
|Red|Segmentación Purdue en 3 VLAN + IT + WAN; el switch es solo L2 y todo el tráfico entre zonas pasa por el FortiGate|
|Firewall|Denegación por defecto; conduits mínimos (Modbus solo HMI/gateway→PLC, MQTT solo puente→broker, HTTPS solo puente→EC2); todo registrado|
|Switch|Puertos no usados apagados en VLAN 999, VLAN nativa no usada, `nonegotiate`, BPDU Guard, SPAN para monitoreo|
|Protocolo|IDS con lista blanca IP→códigos de función Modbus, detección de escrituras, reconocimiento (FC43/FC08) e inundación|
|MQTT|Sin anónimos, claves hasheadas (PBKDF2), ACL por tópico: el puente solo lee, no puede inyectar datos|
|Nube|Solo conexión saliente, TLS 1.2+, certificate pinning, token de solo escritura, nginx expone únicamente `/api/v2/write`|
|Contenedores|Versiones fijadas, usuario sin privilegios (salvo PLC: puerto 502; sonda: captura raw), sin secretos en la imagen|
|Disponibilidad|Watchdog de sensor en el PLC, reintentos y resincronización del actuador, store-and-forward en el puente|

## Estructura

```
contenedores/<9 contenedores>/   Dockerfile + script Python (+ requirements.txt)
herramientas/atacante-modbus/    Generador de tramas de prueba
red/                             IOSvL2, FortiGate y guía GNS3
nube-aws/                        Stack EC2: nginx TLS + InfluxDB + Grafana (dashboard aprovisionado)
docker-compose.yml               Despliegue local con el mismo plan IP de GNS3
```

