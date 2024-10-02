# Node Exporter

Node Exporter is a Prometheus exporter for collecting hardware and OS metrics exposed by \*NIX kernels. This guide provides step-by-step instructions for setting up Node Exporter, including installation, configuration, and integration with Prometheus for detailed monitoring of Linux system metrics.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Create Directories

Create a node-exporter folder to store the configuration files:

```shell
mkdir node-exporter
cd node-exporter
```

## Docker Compose Setup

Create a `docker-compose.yml` file in the node-exporter directory with the following content:

{% code title="docker-compose.yml" %}
```yaml
services:
  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
```
{% endcode %}

> We use the host network for Node Exporter; otherwise, it cannot access the network information from the host's network interfaces.

## Start Node Expoter

To collect real-time system metrics from Node Exporter, update your existing `prometheus.yml` configuration file.

Append the following configuration to your `prometheus.yml` file:

```shell
docker compose up -d
```

## Prometheus Scrape configuration

A Prometheus scrape config for Node Exporter is needed to collect real-time systems metrics, essential for monitoring resource usage and optimizing performance.

Add the below to your existing `prometheus.yml`

{% code title="prometheus.yml" %}
```yaml
scrape_configs: # Optional is this is your first config
   - job_name: 'nodeexporter'
    scrape_interval: 60s
    static_configs:
      - targets: ['<Server IP Address>:9100']
        labels:
          hostname: '<hostname>'
    metric_relabel_configs:
      - source_labels: [hostname]
        target_label: "instance"
        action: "replace"
```
{% endcode %}

> Replace with the IP address of your server and with the hostname of the monitored server.

Restart Prometheus to apply the changes.

## Grafana Dashboard

You can import an existing Node Exporter Grafana dashboard to quickly start visualizing your container metrics.

### Download the Dashboard

You can download the dashboard JSON file from this [GitHub repository](https://github.com/svenvg93/Grafana-Dashboard/tree/master/node\_expoter).

### Import the Dashboard into Grafana

1. Open your Grafana instance and go to Dashboards > Import.
2. Upload the downloaded JSON file.
3. Choose the correct Prometheus datasource.

Once imported, you can begin monitoring the real-time resource usage of your systems using Grafana.
