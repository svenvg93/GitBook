# Grafana Alloy

This guide offers a detailed tutorial on setting up and using Grafana Alloy, covering configuration, data integration, and visualization to enable powerful, unified insights and monitoring across your systems.

!!! note "Your mileage may vary" The step below might need adjustment to work in your environment!

??? tip "Prerequisites" \* Docker installed on your server

## Directories

Create a **alloy** folder

```shell
mkdir alloy
cd alloy
```

## Docker

Make the docker compose file containing all information to start the container.

```yaml
services:
  alloy:
    image: grafana/alloy:v1.2.0
    container_name: alloy
    restart: unless-stopped
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
    command: run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy
      - /var/log/:/logs # location of the log files to import
    networks:
      - monitoring

networks:
  monitoring:
    name: monitoring
```

## Configuration

Create Alloy configuration file. This configuration will grab the systems auth & syslog logging. And sent it to Grafana Loki.

```yaml
loki.write "default" {
  endpoint {
    url = "http://<loki-ip>:3100/loki/api/v1/push"
  }
}

loki.source.file "syslog" {
  targets = [
    {
      __path__ = "/logs/syslog"
      job      = "syslog"
    }
  ]
  forward_to = [loki.write.default.receiver]
}

loki.source.file "authlog" {
  targets = [
    {
      __path__ = "/logs/auth.log"
      job      = "authlog"
    }
  ]
  forward_to = [loki.write.default.receiver]
}

```

!!! info Replace `<loki-ip>` with the IP Address of your Loki server.

### Start Alloy

```shell
docker compose up -d
```

## Migrate from Promtail

Alloy replaces Grafana Promtail to allow for a smooth migration Alloy has a built in migration tool which will convert your promtail configuration to a Alloy accepted format.

### Docker Volumes

Add the following to your `docker-compose.yml`

```yaml
volumes:
  - /path/to/promtail-config.yaml:/promtail-config.yaml:r0
```

Restart the docker container

```shell
docker compose up -d --force-recreate
```

### Migration Command

Now we need to login to the container and run the command below to create a Alloy accepted config file based of your promtail configuration.

Container Login

```shell
docker exec -it alloy /bin/bash
```

```shell
alloy convert --source-format=promtail --output=config.alloy promtail-config.yaml
```

As we are inside the docker container we need to grab the content of the config file and copy it to the config file we created earlier.

```shell
cat config.alloy
```

Copy the content and past it in your existing `config.alloy` configuration file.

Leave the container by the **exit** command.

In the `docker-compose.yml` remove the volume for the `promtail-config.yaml` and recreate the container.

```shell
docker compose up -d --force-recreate
```
