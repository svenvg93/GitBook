# Prometheus

Prometheus is an open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach. This guide walks you through installing and setting up Prometheus

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Directories

Create a `prometheus` folder, which will hold the needed files.

```shell
mkdir prometheus
cd prometheus
```

## Docker

Make the docker compose file containing all information to start the container.

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

Create the Prometheus configuration file, `prometheus.yml`, to define how Prometheus should scrape and store metrics. This step is crucial for ensuring that Prometheus collects the right data and operates according.

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

## Start Prometheus

```shell
docker compose up -d
```

Go to `http://<Host IP>:9090`, to see the Prometheus WebGui.

## Grafana Datasource

You can to add Prometheus as a data soure to Grafana to see the data collected. This can be done via the Grafana WebGui or via Grafana Provisioning.

### WebGui

1. Click **Connections** in the left-side menu.
2. Search for **Prometheus**
3. Click **Add new Datasource**
4. Enter the name **prometheus**
5. Fill in the Prometheus server URL `http://<prometheus-ip>:9090`

> Replace `<prometheus-ip>` with the IP Address of your prometheus server.

### Provisioning

Add the following to your `datasource.yml` for Grafana.

```yaml
apiVersion: 1 # Optional is this is your first datasource
datasources: # Optional is this is your first datasource
  - name: prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://<prometheus-ip>:9090
    basicAuth: false
    isDefault: true
    editable: false
```

> Replace `<prometheus-ip>` with the IP Address of your prometheus server.

Restart Grafana

```
docker restart grafana
```
