---
title: Smarthome auf RaspberryPi - Traefik2
author: benoegen
tag: it
---
Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome in homeassistant mit weiteren features auf einem RaspberryPi mittels docker-compose aufsetzt.

Die einzelnen Abschnitte sind dabei:

  - Grundlagen und Portainer
  - DNS-Filter und DHCP mit Pihole
  - von außen über domain bei duckdns erreichbar mit traefik,
  - Homeassistant, zigbee2mqtt, mariadb, node-red
  - Passwortmanager mit bitwarden

### Einsatzzweck von Traefik

Als Vorarbeit für Homeassistant und auch bitwarden macht eine Erreichbarkeit von außerhalb des lokalen Netzwerkes Sinn. Das lässt sich über Port Freigaben und besser noch über einen Reverse proxy realisieren. Bei mir kommt dafür traefik in der Version 2 zum Einsatz, die Domain von extern wird von duckdns (kostenlos) bereit gestellt. Dabei muss der Router so konfiguriert sein, dass er seine IP an duckdns liefert (dyndns). Die Ports 80 und 443 müssen auf die IP vom Raspberry Pi verweisen.


[![traefik](/assets/screenshots/traefik.png){:class="img-responsive"}](/assets/screenshots/traefik.png)

Zuerst müssen Ordner und Dateien erstellt werden

```
sudo mkdir -p /opt/containers/traefik
sudo mkdir -p /opt/containers/traefik/data
sudo touch /opt/containers/traefik/data/acme.json
sudo chmod 600 /opt/containers/traefik/data/acme.json
```

htpasswd installieren, dieses Tool wird zur Erstellung der Zugangsdaten benötigt:

```
sudo apt-get update
sudo apt-get install apache2-utils
```
<!--mehr-->

Die docker-compose file (muss bei XXX entsprechend abgeändert werden):

```
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=traefik
      - --entrypoints.web.address=:80
      - --entrypoints.web.http.redirections.entryPoint.to=websecure
      - --entrypoints.web.http.redirections.entryPoint.scheme=https
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.myresolver.acme.dnschallenge.provider=duckdns
      - --certificatesresolvers.myresolver.acme.dnschallenge.resolvers=1.1.1.1:53,8.8.8.8:53
      - --certificatesresolvers.myresolver.acme.email=XXX@XXX.xx
      - --certificatesresolvers.myresolver.acme.storage=/acme.json
      - --metrics.prometheus=true
      - --api.dashboard=true
      - --entrypoints.websecure.http.tls.certResolver=myresolver
      - --entrypoints.websecure.http.tls.domains[0].main=XXX.duckdns.org
      - --entrypoints.websecure.http.tls.domains[0].sans=*.XXX.duckdns.org
      - --providers.docker.defaultrule= Host(`{{ index .Labels "com.docker.compose.service" }}.XXX.duckdns.org`)
      - --providers.file.filename=/dynamic_conf.yml
    ports:
      - "80:80"
      - "443:443"
    environment:
      - DUCKDNS_TOKEN=XXX
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/containers/traefik/data/acme.json:/acme.json
      - /opt/containers/traefik/data/dynamic_conf.yml:/dynamic_conf.yml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.XXX.duckdns.org`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=USER:PASSWORT_HT"
      - "providers.file.filename=/dynamic_conf.yml"
    extra_hosts:
      - host.docker.internal:172.17.0.1
```

Nutzer und Passwort anlegen für die compose Datei:

`echo $(htpasswd -nb USER PASSWORT) | sed -e s/\\$/\\$\\$/g`

Ausgabe in compose (USER:HTPSSWORT_HT) einfügen.

Das externe Docker Netzwerk anlegen:

`docker network create proxy`

Eine weitere Datei muss erstellt werden, die die Parameter für https/Verschlüsselung anpasst:

`nano /opt/containers/traefik/data/dynamic_conf.yml`

```
tls:
  options:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_AES_128_GCM_SHA256
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256
      curvePreferences:
        - CurveP521
        - CurveP384
      sniStrict: true
http:
  middlewares:
    secHeaders:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        frameDeny: true
        sslRedirect: true
        #HSTS Configuration
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"

```
Wenn das alles gepasst hat, dann sollte nach ausführen von `docker-compose up -d` die Weboberfläche von Traefik erreichbar sein.

Quellen:

https://community.home-assistant.io/t/add-ha-supervised-on-server-with-traefik-letsencryt-and-multiple-existing-services/280392

https://www.reddit.com/r/docker/comments/d0z61s/traefikletsencrypt_with_duckdns/

https://www.alexhyett.com/traefik-vs-nginx-docker-raspberry-pi#docker-networks

https://goneuland.de/traefik-v2-https-verschluesselung-sicherheit-verbessern/

https://teddit.net/r/docker/comments/p82vgr/traefik_with_lets_encrypt_using_duckdns_domain/

https://doc.traefik.io/traefik/https/acme/#configuration-examples
