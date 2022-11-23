---
title: Smarthome auf RaspberryPi - Homeassistant
author: benoegen
tag: it
---
Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome in homeassistant mit weiteren features auf einem RaspberryPi mittels docker-compose aufsetzt.

Die einzelnen Abschnitte sind dabei:

  - Grundlagen und Portainer
  - DNS-Filter und DHCP mit Pihole
  - von außen über domain bei duckdns erreichbares Smarthome mit traefik, homeassistant, zigbee2mqtt, mariadb, node-red
  - Passwortmanager mit bitwarden

### Homeassistant

Grundsätzlich muss zunächst folgender Eintrag in die docker-compose eingefügt werden, das image gepullt und der Container gestartet werden:

```
  homeassistant:
    image: homeassistant/raspberrypi4-64-homeassistant:stable
    restart: unless-stopped
    container_name: homeassistant_container
    volumes:
      - /opt/containers/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
    network_mode: host
    environment:
      - TZ=Europe/Berlin
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hassio.rule=Host(`XXX.XXX.duckdns.org`)"
      - "traefik.http.routers.hassio.entrypoints=websecure"
      - "traefik.http.services.hassio.loadbalancer.server.port=8123"
      - "traefik.http.routers.hassio.service=hassio"
      - "traefik.http.routers.hassio.tls.certresolver=myresolver"
```

`docker-compose pull`

`docker-compose up -d --remove-orphans --build `


Die eingetragenen Labels sind dabei zur Kommunikation mit Traefik eingetragen und XXX muss an die eigene Domain angepasst werden. Damit Traefik korrekt funktioniert und HA außerhalb des eigenen Netzwerks erreichbar ist, muss der reverse proxy noch in der home assistant configuration.yaml eingetragen werden:

```
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.18.0.2      # Add the IP address of the proxy server
    - 172.18.0.0/16
```

Damit sollte Homeassistant schon laufen, es macht dennoch Sinn sowohl mosquitto als mqtt Server, als auch zigbee2mqtt zu installieren, so können zigbee Geräte über HA gesteuert werden.

<!--mehr-->

### Mosquitto

Mosquitto ist ein mqtt-server der auf der einen Seite mit HA kommunizieren kann, auf der anderen Seite mit zigbee2mqtt oder valetudo auf Saugrobotern oder Tasmota Geräten, also eine sehr universelle Schnittstelle zur Gerätekommunikation.

docker-compose:

```
  mosquitto:
    image: eclipse-mosquitto
    restart: unless-stopped
    container_name: mosquitto
    #network_mode: host
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - /opt/containers/mosquitto/conf:/mosquitto/config
      - /opt/containers/mosquitto/data:/mosquitto/data
      - /opt/containers/mosquitto/log:/mosquitto/log
```

Danach die `/opt/containers/conf/mosquitto.conf` erstellen:

`sudo nano /opt/containers/conf/mosquitto.conf` 
 
```
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log

#certfile fullchain.pem
#keyfile privkey.pem
require_certificate false
listener 1883
## Authentication ##
allow_anonymous false
password_file /mosquitto/config/mosquitto.passwd
```
User erstellen: `docker-compose exec mosquitto mosquitto_passwd -c /mosquitto/conf/mosquitto.passwd usermqtt`


### Zigbee2mqtt

Zigbee2mqtt kommunziert mittels mqtt mit Homeassistant, HA muss dabei korrekt mit dem Mqtt-Server (mosquitto) verbunden sein, die Zigbee Geräte tauchen dann dort automatisch auf.

[![1](/assets/screenshots/p0zEi9Hk38.png){:width="250px"}](/assets/screenshots/p0zEi9Hk38.png)

Der Eintrag für die docker-compose:

```
  zigbee2mqtt:
    container_name: zigbee2mqtt
    restart: unless-stopped
    image: koenkk/zigbee2mqtt
    volumes:
      - /opt/containers/zigbee2mqtt/data:/app/data
      - /run/udev:/run/udev:ro
    ports:
      - 8080:8080
    environment:
      - TZ=Europe/Berlin
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
```

Die Zigbee Config sollte so ähnlich aussehen, je nach Zigbee-Adapter unterscheidet sich der Port.

```
data_path: /config/zigbee2mqtt
external_converters: []
devices: devices.yaml
groups: groups.yaml
homeassistant: true
permit_join: true
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://XXX #IP des PI im Netzwerk, virtuelle docker-IP hat bei mir nicht funktioniert
  user: usermqtt
  password: XXX #(wie vorher festgelegt)
serial:
  port: /dev/ttyUSB0
advanced:
  log_level: debug
  pan_id: 6757
  channel: 11
  network_key:
    - 1
    - 3
    - 5
    - 7
    - 9
    - 11
    - 13
    - 15
    - 0
    - 2
    - 4
    - 6
    - 8
    - 10
    - 12
    - 13
  availability_blocklist: []
  availability_passlist: []
device_options: {}
blocklist: []
passlist: []
queue: {}
frontend:
  port: 8080
experimental: {}
availability: false
socat:
  enabled: false
  master: pty,raw,echo=0,link=/tmp/ttyZ2M,mode=777
  slave: tcp-listen:8485,keepalive,nodelay,reuseaddr,keepidle=1,keepintvl=1,keepcnt=5
  options: '-d -d'
  log: false

```