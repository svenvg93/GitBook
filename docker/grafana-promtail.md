# Grafana Promtail

This guide offers a detailed tutorial on setting up Promtail, including installation, configuration, and integration with Loki for efficient log collection and forwarding.

!!! failure Grafana Promtail is deprecated and replaced with Grafana Alloy.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Directories

Create a **promtail** folder

```shell
mkdir promtail
cd promtail
```

## Docker

Make the docker compose file containing all information to start the container.

```yaml
services:
  promtail:
    image: grafana/promtail:2.9.2
    container_name: promtail
    restart: unless-stopped
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
    volumes:
      - ./promtail-config.yaml:/etc/promtail/promtail-config.yaml:ro
      - /var/log/:/logs # location of the log files to import
    command: -config.file=/etc/promtail/promtail-config.yaml
    networks:
      - monitoring

networks:
  monitoring:
    name: monitoring
```

## Configuration

Create Promtail configuration file. This configuration will grab the systems auth & syslog logging. And sent it to Grafana Loki.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://<loki-ip>:3100/loki/api/v1/push

scrape_configs:
- job_name: authlog
  static_configs:
  - targets:
      - localhost
    labels:
      job: authlog
      __path__: /var/log/auth.log

- job_name: syslog
  static_configs:
  - targets:
      - localhost
    labels:
      job: syslog
      __path__: /var/log/syslog 

```

!!! info Replace `<loki-ip>` with the IP Address of your Loki server.

## Start Promtail

```shell
docker compose up -d
```
