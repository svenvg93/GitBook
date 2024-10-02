# Grafana Promtail

Warning: Grafana Promtail is deprecated and has been replaced with Grafana Alloy. Consider migrating to Alloy for future-proofing your setup.

This guide provides a comprehensive tutorial on setting up Promtail, covering installation, configuration, and integration with Loki for efficient log collection and forwarding.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Docker Compose Setup

Create a `promtail` folder to store the configuration files:

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

## Create the Promtail Configuration File

Create a configuration file named `promtail-config.yaml` with the following content. This configuration will capture the system’s auth and syslog logs and send them to Grafana Loki.

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

> Replace `<loki-ip>` with the IP Address of your Loki server.

## Start Promtail

Start the Promtail container using Docker Compose:

```shell
docker compose up -d
```
