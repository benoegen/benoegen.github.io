---
title: Smarthome auf RaspberryPi - PiHole
author: benoegen
tag: it
---
Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome in homeassistant mit weiteren features auf einem RaspberryPi mittels docker-compose aufsetzt.

Die einzelnen Abschnitte sind dabei:

  - Grundlagen und Portainer
  - DNS-Filter und DHCP mit Pihole
  - von außen über domain bei duckdns erreichbares Smarthome mit traefik, homeassistant, zigbee2mqtt, mariadb, node-red
  - Passwortmanager mit bitwarden

### Einsatzzweck von PiHole

[![Käfer](/assets/screenshots/2022-05-04 09_13_19-Pi-hole.png){:class="img-responsive"}](/assets/screenshots/2022-05-04 09_13_19-Pi-hole.png)

PiHole blockiert Werbung/Tracking bereits auf DNS Ebene, so dass diese gar nicht erst geladen werden, das spart Bandbreite gegenüber Ad-Blockern im Browser o.ä. Dabei wird das ganze Netzwerk gefiltert, sei es Handy, Computer, Smart-TV oder oder. Dabei muss der Pi lediglich im Router als DNS Server angegeben werden.

Mittels PiHole kann der RaspberryPi auch die Aufgabe eines DHCP übernehmen, die Oberfläche ist komfortabler und zugänglicher als bspw. bei einer FritzBox.

<!--mehr-->

### Besonderheiten bei der Installation auf Ubuntu

Auf Ubuntu (17.10+) blockiert systemd-resolved port 53 und blockiert damit die Funktion von pihole. Deaktiviert wird es mittels: `sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf`

und

`sudo sh -c 'rm /etc/resolv.conf && ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf'` 

Anschließend systemd-resolved neu starten `systemctl restart systemd-resolved`

#### dhcp in pihole einrichten

Die Vorarbeit, um Pi-hole als DNS Filter einzusetzen ist damit beendet, um auch als DHCP zu fungieren, sind weitere Vorarbeiten notwendig.

#### Cloud init deaktivieren

`sudo nano /etc/cloud/cloud.cfg.d/99-custom-networking.cfg`

Inhalt der neuen Datei

```
network: {config: disabled}
```

#### Statische IP in netplan vergeben
Die Netplan config muss bearbeitet werden, normalerweise sollte der Name folgender sein:
`sudo nano /etc/netplan/10-rpi-ethernet-eth0.yaml`

Dateiinhalt soll so aussehen:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      # Rename the built-in ethernet device to "eth0"
      dhcp4: no
      addresses: [192.168.1.101/24] #feste IP
      routes:
        - to: default
          via: 192.168.1.1 #Router
      nameservers:
        addresses: [127.0.0.1,8.8.8.8]
```

Danach 

`sudo netplan generate`

`sudo netplan try -timeout 180 `

Falls keine Fehler entstehen:

`sudo netplan apply`

### docker-compose
Wen die Vorarbeiten erledigt sind, kann das docker-compose.yml erstellt, bzw. erweitert werden `sudo nano docker-compose.yml`

```
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    network_mode: host
    environment:
      TZ: 'Europe/Berlin'
      WEBPASSWORD: 'supersicherespasswort'
      WEB_PORT: 5089
    volumes:
      - '/opt/containers/pihole/etc-pihole:/etc/pihole'
      - '/opt/containers/pihole/etc-dnsmasq.d:/etc/dnsmasq.d'      
    cap_add:
      - NET_ADMIN # Recommended but not required (DHCP needs NET_ADMIN)      
    restart: unless-stopped
```

Datei speichern und `docker-compose up -d` (im Ordner, wo auch die compose-file liegt) ausführen. Unter `IP_des_PI:5089/admin` ist nach kurzer Zeit die PiHole Oberfläche zu finden.