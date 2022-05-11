---
title: Smarthome auf RaspberryPi - Traefik2
author: benoegen
tag: it
---
Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome in homeassistant mit weiteren features auf einem RaspberryPi mittels docker-compose aufsetzt.

Die einzelnen Abschnitte sind dabei:

  - Grundlagen und Portainer
  - DNS-Filter und DHCP mit Pihole
  - von außen über domain bei duckdns erreichbares Smarthome mit traefik, homeassistant, zigbee2mqtt, mariadb, node-red
  - Passwortmanager mit bitwarden

### Einsatzzweck von Traefik