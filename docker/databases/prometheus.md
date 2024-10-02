# Prometheus

Prometheus is an open-source monitoring system with a flexible query language, an efficient time series database, and a modern alerting approach. This guide walks you through installing and setting up Prometheus using Docker.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Create Directories

First, create a prometheus folder to store the necessary configuration files:

```shell
mkdir prometheus
cd prometheus
```

## Docker Compose Setup

Create a `docker-compose.yml` file containing the configuration needed to start the Prometheus container:

{% code title="docker-compose.yml" %}
```yaml
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
    restart: unless-stopped
    network_mode: bridge
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.size=100GB'# Change the retention size to your liking.
      - '--web.enable-lifecycle'

volumes:
    prometheus:
      name: prometheus
```
{% endcode %}

## Create Prometheus Configuration File

Create a file named prometheus.yml in the prometheus directory. This file will define how Prometheus scrapes and stores metrics:

{% code title="prometheus.yml" %}
```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 60s
    static_configs:
      - targets: ['localhost:9090']
```
{% endcode %}

## Start Prometheus

Use the following command to start the Prometheus container:

```shell
docker compose up -d
```

Once the container is running, access the Prometheus web UI at:

`http://<Host IP>:9090`

## Adding Prometheus as a Grafana Datasource

Prometheus can be integrated into Grafana to visualize collected data. You can set up the Prometheus datasource through the Grafana web interface or through provisioning.

### Using the Grafana Web Interface

1. Go to **Connections** in the left-side menu.
2. Search for **Prometheus**.
3. Click **Add new Datasource.**
4. Enter **prometheus** as the name.
5. Set the URL to **http://\<prometheus-ip>:9090**.

> Replace `<prometheus-ip>` with the IP Address of your prometheus server.

### Using Grafana Provisioning

To automate the process, add the following configuration to your `datasource.yml` file in Grafana’s provisioning directory:

{% code title="datasource.yml" %}
```yaml
apiVersion: 1
datasources:
  - name: prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://<prometheus-ip>:9090
    basicAuth: false
    isDefault: true
    editable: false
```
{% endcode %}

> Replace `<prometheus-ip>` with the IP Address of your prometheus server.

### Restart Grafana

If you used provisioning to set up the datasource, restart the Grafana container:
