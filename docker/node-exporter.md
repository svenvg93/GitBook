# Node Exporter

Prometheus exporter for hardware and OS metrics exposed by \*NIX kernels. This guide provides step-by-step instructions on setting up Node Exporter, covering installation, configuration, and integration with Prometheus for detailed monitoring of Linux system metrics.

!!! note "Your mileage may vary" The step below might need adjustment to work in your environment!

??? tip "Prerequisites" \* Docker installed on your server

## Directories

Create a **node-exporter** folder, which will hold the needed files.

```shell
mkdir node-exporter
cd node-exporter
```

## Docker

Make the docker compose file containing all information to start the container.

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

We use the host network for Node Exporter; otherwise, it cannot access the network information from the host's network interfaces.

## Start Node Expoter

```shell
docker compose up -d
```

## Prometheus Scrape configuration

A Prometheus scrape config for Node Exporter is needed to collect real-time systems metrics, essential for monitoring resource usage and optimizing performance.

Add the below to your existing `prometheus.yml`

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

!!! info \* Replace `<Server IP Address>` with the IP Address of your server. \* Replace `<hostname>` with the hostname of the server monitored

## Grafana Dashboard

You can import this [Dashboard](https://github.com/svenvg93/Grafana-Dashboard/tree/master/node\_expoter) to get started.
