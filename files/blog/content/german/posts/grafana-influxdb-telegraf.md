---
title: 'Statistiken mit Grafana, Influxdb und Telegraf erstellen und Anzeigen'
date: 2022-06-21T36:36:07+02:00
tags:
  - 'grafana'
  - 'influxdb'
  - 'influxdb2'
  - 'telegraf'
  - 'monitoring'
  - 'ansible'
  - 'flux'
  - 'docker'
  - 'jinja2'
categories:
  - 'influxdb'
  - 'monitoring'
  - 'ansible'
  - 'docker'
  - 'jinja2'
description: 'Wie stelle ich Server-Statistiken durch Grafana und Influxdb dar'
draft: true
showToc: true
---

# Vorwort

Lange Zeit habe ich meine Serverstatistiken, dazu gehöoren CPU Usage, Memory Usage, Disk Usage und weitere Daten, über Grafana durch selbstgebaute Skripte dargestellt.
Diese Skripte haben die Informationen im Interval von einer Minute an eine mariadb eingeliefert.

Das führte aber dazu, dass ich maximal Daten für knapp 30 Tage vorhalten konnte, ohne das die Ladezeit der Grafana-Dashboards exorbitant angestiegen ist.

Nun habe ich mir zur Aufgabe gemacht eine Umgebung durch ansible zuerstellen, die mit einem Lauf einen Server (Grafana, Influxdb und Telegraf Receiver) und/oder einen Client (Telegraf) erzeugt.
Dabei lag mein Augenmerk vorallem darauf, dass man nicht unbedingt alles über diese Technologien wissen muss, um eine funktionierende Umgebung zu erzeugen.

Die Ansible-Rolle die hier genutzt wird ist auf meinem [Github](https://github.com/dhoessl/monitoring) zu finden.

# Aufbau
## Grafana

Begonnen habe ich mit der Erstellung einer aktuellen Grafana Instanz. Diese kann man mit dem [Ansible docker_compose Modul](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_module.html) erzeugen.

```yaml
grafana:
  image: 'grafana/grafana:latest'
  environment:
    GF_SERVER_PROTOCOL: '{{ monitoring_proto }}'
    GF_SERVER_DOMAIN: '{{ monitoring_domain }}'
    GF_SERVER_ROOT_URL: '{{ monitoring_proto }}://{{ monitoring_domain }}'
    GF_SECURITY_ADMIN_USER: '{{ monitoring_grafana_username }}'
    GF_SECURITY_ADMIN_PASSWORD: '{{ monitoring_grafana_password }}'
  restart: 'unless-stopped'
  volumes:
    - '/var/lib/grafana/:/var/lib/grafana'
    - '/etc/grafana:/etc/grafana'
  ports:
    - '3000:3000'
  user: '109'
```

Die Umgebungsvariablen werden durch die Ansible-Variablen gesetzt, welche schon defaults gesetzt haben. Password-Variablen haben keinen default gesetzt und müssen durch den Nutzer befüllt werden. Sollte das nicht der Fall sein, wird der Lauf mit entsprechender Meldung abgebrochen.

Das Protokoll kann grundsätzlich auf dem default "http" gelassen werden, wenn man die Seite über einen Reverse-Proxy auflöst.

Wichtig zu wissen ist, dass Grafana mit einem unpriviligierten Nutzer gestartet werden muss (in diesem Fall 109). Laut Dokumentation ist das aufgrund der Sicherheitspolicy.

Im Verzeichnis /var/lib/grafana sind Daten zum Alerting, Plugins und weitere Einstellungen gespeichert.

Im Verzeichnis /etc/grafana liegt die grafana.ini, die auf jedenfall vorhanden sein muss. Im Playbook wird diese Datei später automatisch mit angelegt

## Grafana Image Renderer

Für ein Alerting über z.B. Telegram kann man einen Image Renderer mit aufbauen, damit Bilder bei der Benachrichtigung angezeigt werden können.

Der Container kann einfach aufgebaut werden und sollte mit ins interne Netz aufgenommen werden.
```yaml
ir:
  image: 'grafana/grafana-image-renderer:latest'
  restart: 'unless-stopped'
  networks:
    - 'monitoring_net'
```

Damit Grafana mit diesem Container kommunizieren kann, muss Grafana auch mit dem gleichen Netz verbunden sein.

Die Containerdefinition von Grafana muss durch die folgenden Umgebungsvariablen ergänzt werden:
```yaml
GF_RENDERING_SERVER_URL: 'http://ir:8081/render'
GF_RENDERING_CALLBACK_URL: 'http://{{ monitoring_domain }}'
```
Da die Container durch ein Docker Netzwerk verknüpft sind, kann man bei der Server-URL einfach den Containernamen angeben (ir).

## Influxdb

Wie bereits im [Vorwort](#vorwort) erwähnt, habe ich meine Statistiken selbst per Python3 Skripte gesammelt und an eine mariadb geschickt.
Alleine bei Statistiken von drei Server hatte das Grafana eine Ladezeit von über 5 Minuten, wenn ich die Daten mehr als 30 Tage vorgehalten habe.
Also habe ich automatisiert die Daten aus der Datenbank gelöscht.

Meine selbstgebauten Pythonskripte haben ausgedient. Außerdem möchte ich die Vorteile einer Time Series Database (TSDB) im Gegensatz zu einer Key-Value Datenbank nutzen.
Vorallem liegt mein Augenmerk hier auf der schnellen und effizienten Auswertung der Daten.

Zur Erstellung der Datenbank habe ich folgende Parameter gesetzt:
```yaml
influxdb:
  image: 'influxdb:latest'
  environment:
    DOCKER_INFLUXDB_INIT_MODE: 'setup'
    DOCKER_INFLUXDB_INIT_USERNAME: '{{ monitoring_influxdb_username }}'
    DOCKER_INFLUXDB_INIT_PASSWORD: '{{ monitoring_influxdb_password }}'
    DOCKER_INFLUXDB_INIT_ORG: '{{ monitoring_influxdb_org }}'
    DOCKER_INFLUXDB_INIT_BUCKET: '{{ monitoring_influxdb_bucket }}'
    DOCKER_INFLUXDB_INIT_RETENTION: '{{ monitoring_influxdb_retention }}'
    DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: '{{ monitoring_influxdb_token }}'
  restart: 'unless-stopped'
  volumes:
    - '/var/lib/influxdb2:/var/lib/influxdb2'
  networks:
    - 'monitoring_net'
  ports:
    - '8086:8086'
```

Da influxdb die erste neue Technologie in diesem Projekt ist, hat es mit der Erstellung und Einrichtung etwas länger gedauert. Es mussten einige Verständnisprobleme aus dem Weg geräumt werden.
Zuerst musste ich klar werden, dass es Influxdb V1 und Influxdb V2 gibt. Als nächstes bin ich darauf gestoßen, dass Influxdb eine eigene Querysprache ([Flux](https://docs.influxdata.com/influxdb/v1.8/flux/)) mitbringt.

Das Abrufen der Daten ist zum einen über Basic Auth möglich, als auch mit dem Admin Token. Dieser Token kann auch bei der Erstellung des Containers durch die Umgebungsvariable `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN` gesetzt werden.
Man kann im Nachgang natürlich auch User mit Tokens anlegen und dort Rechte einschränken. Für mein Projekt soll es reichen den Admin Token zu nutzen.

Die Rolle befüllt die Variablen `DOCKER_INFLUXDB_INIT_ORG`,  `DOCKER_INFLUXDB_INIT_BUCKET`, `DOCKER_INFLUXDB_INIT_RETENTION` mit Standardwerten, diese können aber auch angepasst werden.

Die Variable `DOCKER_INFLUXDB_INIT_USERNAME` wird mit dem Wert `admin` gefüllt, kann und sollte aber angepasst werden.

Die Variablen `DOCKER_INFLUXDB_INIT_PASSWORD` und `DOCKER_INFLUXDB_INIT_ADMIN_TOKEN` werden nicht mit einem Wert befüllt, da diese vom Nutzer in einem Vault-File gesetzt werden sollen.
Sollten die Variablen nicht gesetzt sein und auf einem Host die Influxdb ausgerollt werden, bricht das Playbook den Lauf mit einer entsprechenden Fehlermeldung ab, dass man diese Werte noch setzen muss.

Wie man die Influxdb nutzt wird später noch behandelt.

## Telegraf

Telegraf Container lassen sich in dieser Rolle auf zwei Arten nutzen. Zum einem ein Telegraf Container auf dem Monitoring Server, auf dem [StatD]() und ein [Nginx Scan]() läuft.
[StatD]() ermöglicht es Daten im bestimmten Format anzunehmen und in die Influxdb zu schreiben.
Zum anderen kann man auf Hosts die man monitoren möchte eine Telegraf "Client" ausrollen, der Daten wie CPU Nutzung, Memory Nutzung und Disk Nutzung ausliest und an die Influxdb schickt.

### Telegraf als Datenempfänger

### Telegraf als Client
