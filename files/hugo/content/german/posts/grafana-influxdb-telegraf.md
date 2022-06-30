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

```
- name: 'Create Grafana Server'
  docker_compose:
    pull: true
    project_name: 'monitoring'
    state: 'present'
    definition:
      version: '3'
      networks:
        monitoring_net:
          external: false
      services:
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
          user: '104'
```
