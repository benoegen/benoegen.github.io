---
title: Smarthome auf RaspberryPi Pihole
author: benoegen
tag: it
---
### Ziel

Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome auf homeassistant mit weiteren Extras (proxy, pihole etc) auf einem RaspberryPi mittels docker-compose aufsetzt.

Die einzelnen Abschnitte sind dabei:

  - Grundlagen und Portainer
  - DNS-Filter und DHCP mit Pihole
  - von außen über domain bei duckdns erreichbares Smarthome mit traefik, homeassistant, zigbee2mqtt, mariadb, node-red

### Struktur

Alle Container und die dazugehörigen Daten werden unter `/opt/containers/` abgelegt, für abweichende Strukturen müssen die einzelnen Dateien und Befehle abgeändert werden.

