# Grafana Alloy

This guide offers a detailed tutorial on setting up and using Grafana Alloy, covering configuration, data integration, and visualization to enable powerful, unified insights and monitoring across your systems.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Create Directories

Create a alloy folder to store the configuration files:

```shell
mkdir alloy
cd alloy
```

## Docker Compose Setup

Make the docker compose file containing all information to start the container.

{% code title="docker-compose.yml" %}
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
{% endcode %}

## Create the Alloy Configuration File

Create a configuration file named `config.alloy` with the following content. This configuration will capture the system’s auth and syslog logging, sending it to Grafana Loki.

{% code title="config.alloy" %}
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
{% endcode %}

> Replace `<loki-ip>` with the IP Address of your Loki server.

### Start Alloy

Start the Alloy container using Docker Compose:

```shell
docker compose up -d
```

## Migrate from Promtail

Alloy replaces Grafana Promtail, providing a smooth migration path. Alloy includes a built-in migration tool to convert your Promtail configuration to a format accepted by Alloy.

### Add Docker Volumes

Update your `docker-compose.yml` to include the following volume for your Promtail configuration:

```yaml
volumes:
  - /path/to/promtail-config.yaml:/promtail-config.yaml:r0
```

### Restart the Docker Container

Restart the container with the updated configuration:

```shell
docker compose up -d --force-recreate
```

### Run the Migration Command

Log in to the Alloy container and run the migration command to create an Alloy-accepted configuration file based on your Promtail configuration.

#### Container Login

```shell
docker exec -it alloy /bin/bash
```

#### Run Migration Command

```shell
alloy convert --source-format=promtail --output=config.alloy promtail-config.yaml
```

#### Copy the Configuration

While inside the Docker container, display the content of the new config file:

```shell
cat config.alloy
```

Copy the content and paste it into your existing `config.alloy` configuration file. Exit the container by typing exit.

Clean Up and Recreate the Container

Remove the volume for `promtail-config.yaml` from `docker-compose.yml` and recreate the container:

```shell
docker compose up -d --force-recreate
```
