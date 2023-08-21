---
title: "Grafana Loki 日志收集"
date: 2023-08-21T08:58:27+08:00
tags: [docker, grafana, loki]
---

[Loki](https://grafana.com/oss/loki/) 是一个轻量级的日志收集解决方案,
可以轻松收集 `Docker` 日志, 参考官方教程
[Install Grafana Loki with Docker or Docker Compose](https://grafana.com/docs/loki/latest/installation/docker/?pg=oss-loki&plcmt=quick-links)

<!--more-->

### docker-compose.yaml

```yaml
version: "3"

networks:
  loki:

services:
  loki:
    image: grafana/loki:2.8.0
    restart: always
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki

  promtail:
    image: grafana/promtail:2.8.0
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config/promtail-config.yaml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml
    networks:
      - loki

  grafana:
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=password
    volumes:
        - ./config/grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yaml
    image: grafana/grafana:latest
    restart: always
    ports:
      - "3200:3000"
    networks:
      - loki
```

### ./config/promtail-config.yaml

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: "container"
```

### ./config/grafana-datasources.yml

```yaml
apiVersion: 1

datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    version: 1
    editable: false
    isDefault: true
```
