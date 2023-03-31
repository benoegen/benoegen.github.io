---
title: Smarthome auf RaspberryPi - Traefik2 Update
author: benoegen
tag: it
---
Als Hilfe für andere und auch als Gedankenstütze für mich selber möchte ich dokumentieren, wie man ein Smarthome in homeassistant mit weiteren features auf einem RaspberryPi (inzwischen Futro 740) mittels docker-compose aufsetzt.

### Mehr Sicherheit mit crowdsec

Update zu meinem [Traefik setup]({% post_url 2022-06-19-smarthome_traefik2 %}) 
Um weiterhin Bitwarden guten Gewissens verwenden zu können, soll mein Server abgesicherte werden. Dazu gibt es die Möglichkeit, Logins von verdächtigen Ips zu unterbinden, mittels crowdsec und einem sogenannten Bouncer. Ich möchte dafür dem [Tutorial von Goneuland](https://goneuland.de/traefik-v2-reverse-proxy-mit-crowdsec-einrichten/) folgen. Dafür muss aber die Konfiguration von Traefik etwas umgebaut werden, die Einstellungen sollen in Zukunft nicht mehr in der docker-compose Datei, sondern in einer traefik.yml vorgenommen werden (es geht nur entweder oder) , die aber weiterhin den use case mit duckdns als Domain-Provider abbildet. 
Durch einen Serverumzug liegen meine Container nicht mehr unter /opt/containers, sondern unter /home/containers ab.

<!--mehr-->

Die neue docker-compose sieht so aus:
 
```
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      proxy
    ports:
      - "80:80"
      - "443:443"
    environment:
      - DUCKDNS_TOKEN=XXXXXX
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /home/containers/traefik/data/acme.json:/acme.json
      - /home/containers/traefik/data/traefik.yml:/traefik.yml:ro
      - /home/containers/traefik/data/dynamic_conf.yml:/dynamic_conf.yml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`xxx.xxx.duckdns.org`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.middlewares=traefikAuth@file,default@file"
      - "traefik.http.services.dashboard.loadbalancer.server.port=80"
      - "traefik.http.services.dashboard.loadbalancer.sticky.cookie.httpOnly=true"
      - "traefik.http.services.dashboard.loadbalancer.sticky.cookie.secure=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
    extra_hosts:
      - host.docker.internal:172.17.0.1
```

Die traefik.yml sieht so aus und ersetzt damit alle Konfugurationsflags, die vorher in der deocker-compose standen:

```
api:
  dashboard: true
  
global:
  checknewversion: true
  sendanonymoususage: false

certificatesResolvers:
  myresolver:
    acme:
      email: "XXX@XXX.de"
      storage: "acme.json"
      dnsChallenge:
        provider: "duckdns"
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: "websecure"
          scheme: "https"
          
  websecure:
    address: ":443"
    http:
#      middlewares:
#        - "crowdsec-bouncer@file"
      tls:
        certResolver: myresolver
        domains:
          main: XXX.duckdns.org
          sans:
            - "*.XXX.duckdns.org"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: "proxy"
  file:
    filename: "./dynamic_conf.yml"
    watch: true
  providersThrottleDuration: 10

log:
  level: "INFO"
  filePath: "/var/log/traefik/traefik.log"

accessLog:
  filePath: "/var/log/traefik/access.log"
  bufferingSize: 100

```

Die dynamic folgt den Empfehlungen von goneuland:

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
    traefikAuth:
      basicAuth:
        users:
          - "user:passwort"

    default:
      chain:
        middlewares:
          - default-security-headers
          - gzip

    secHeaders:
      chain:
        middlewares:
          - default-security-headers
          - gzip

    # Standard Header
    default-security-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        frameDeny: true
#       Deprecated
#       sslRedirect: true
        #HSTS Configuration
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"
    # Gzip Kompression
    gzip:
      compress: {}
```

Mit diesen Anpassungen ist es nun möglich, das [goneulandTutorial](https://goneuland.de/traefik-v2-reverse-proxy-mit-crowdsec-einrichten/) nahezu 1:1 zu befolgen. 

Was dort nicht beschrieben ist, dass man den bouncer am Ende testen kann, indem man seine IP manuell auf die Blacklist setzt. Die IP wird zunächst über whatismyipaddress.com oder eine ähnliche Website ermittelt. 

Anschließend setzt man sich mit dem Befehl

```
docker exec crowdsec-example cscli decisions add --ip 192.168.128.1
```

auf die Blockliste. Beim aufrufen der Dienste über traefik müsste nun bei korrekter Konfiguration *forbidden* angezeigt werden. 

Anschließend nimmt man sich selber mit

```
docker exec crowdsec-example cscli decisions delete --ip 192.168.128.1
```

wieder von der Blockliste.
