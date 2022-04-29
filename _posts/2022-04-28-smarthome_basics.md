---
title: Smarthome auf RaspberryPi - Die Grundlagen
author: benoegen
tag: it
---
Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome in homeassistant mit weiteren features auf einem RaspberryPi mittels docker-compose aufsetzt.

Die einzelnen Abschnitte sind dabei:

  - Grundlagen und Portainer
  - DNS-Filter und DHCP mit Pihole
  - von außen über domain bei duckdns erreichbares Smarthome mit traefik, homeassistant, zigbee2mqtt, mariadb, node-red
  - Passwortmanager mit bitwarden

### Anfang

Grundlage bildet ein Raspberry Pi 4. Als Betriebssystem kommt ein 64-bit Ubuntu zum Einsatz, das Ganze ist auf einer SSD installiert.
Mittels `sudo apt-get update && sudo apt-get upgrade -y` wird das System auf den neuesten Stand gebracht.
<!--mehr-->
### docker installieren

Zunächst Script herunterladen `curl -fsSL https://get.docker.com -o get-docker.sh` und ausführen `sudo sh get-docker.sh`.
Der Hauptuser muss noch in die docker Gruppe eingefügt werden (*ubuntu* oder *pi*, je nach System) `sudo usermod -aG docker ubuntu`

### docker-compose

Docker-compose wird benötigt, um container komfortabel über ein composefile starten zu können. Installation über: `sudo curl -L "https://github.com/docker/compose/releases/download/v2.2.3/docker-compose-linux-aarch64" -o /usr/local/bin/docker-compose`
(Die Version könnte inzwischen nicht mehr aktuell sein)

`sudo chmod +x /usr/local/bin/docker-compose`

Danach prüfen, ob compose korrekt installiert wurde:
`docker-compose --version`

### Samba

Samba lässt sich wohl auch als container aufsetzen, habe ich jedoch nicht so gemacht.
Für die ersten Schritte ist Samba auch nicht notwendig, bei der Nutzung von Plex als Mediaserver ist Samba zum befüllen sehr praktisch.
Das Paket Samba installiert man mittels `sudo apt-get install samba`.

Die Konfigurationsdatei backuppen `sudo mv /etc/samba/smb.conf etc/samba/smb_old.conf` und mittels
`sudo nano /etc/samba/smb.conf` neu erstellen.

Beispiel:

```
[global]
workgroup = WORKGROUP
client min protocol = SMB2
client max protocol = SMB3

[Share1]
path = /opt/containers/share/
writeable = yes
```
Der Pfad lässt sich dabei natürlich beliebig variieren, evtl müssen Berechtigungen des Ordners noch geändert werden, um vom Windows PC Schreibrechte zu erhalten.

Sambapasswort für den User ubuntu vergeben: `sudo smbpasswd -a ubuntu`
Scharfschalten `testparm` und Samba neu starten `systemctl restart smbd.service`

Danach ist das grundlegende System eingerichtet und die ersten Container können installiert werden.

### Portainer

Als erster Container bietet sich Portainer an, eine grafische Oberfläche um laufende Container zu prüfen, killen oder neu zu starten.
Hierfür eine docker-compose Datei erstellen
```
cd /opt/containers
sudo nano docker-compose.yml
```

Hier folgenden Code einfügen:

```
version: "3.8"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    ports:
      - 9000:9000
      - 8000:8000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

volumes:
  portainer_data:
```

Datei speichern und `docker-compose up -d` (im Ordner, wo auch die compose-file liegt) ausführen. Unter `IP_des_PI:9000` ist nach kurzer Zeit die Portainer Oberfläche zu finden. Weitere Container können nun in der gleichen compose Datei angelegt werden.

