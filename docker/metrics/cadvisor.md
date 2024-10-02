# cAdvisor

cAdvisor (Container Advisor) helps container users gain insight into resource usage and performance characteristics of their running containers. This guide provides step-by-step instructions for installing, configuring, and monitoring container performance metrics using cAdvisor, ideal for optimizing Docker environments.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Create Directories

Create a `cadvisor` folder to store the configuration files:

```shell
mkdir cadvisor
cd cadvisor
```

## Docker Compose Setup

Create a `docker-compose.yml` file in the cadvisor directory with the following content:

{% code title="docker-compose.yml" %}
```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    container_name: cadvisor
    restart: unless-stopped
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
    networks_mode: bridge
    ports:
      - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /cgroup:/cgroup:ro #doesn't work on MacOS only for Linux
    command: 
      - '--housekeeping_interval=15s'
      - '--docker_only=true'
    devices:
      - /dev/kmsg:/dev/kmsg
    privileged: true
```
{% endcode %}

## Start cAdvisor

Start the cAdvisor container using Docker Compose:

```shell
docker compose up -d
```

## Prometheus Scrape configuration

To monitor real-time container metrics with Prometheus, update your existing `prometheus.yml` configuration file to include cAdvisor.

Append the following configuration to your `prometheus.yml` file:

{% code title="prometheus.yml" %}
```yaml
scrape_configs: # Optional is this is your first config
  - job_name: 'cadvisor'
    scrape_interval: 15s
    static_configs:
      - targets: ['<cadvisor>:8080']
```
{% endcode %}

> Replace `<cadvisor>` with the IP Address of your cAdvisor instance.

## Grafana Dashboard

You can import an existing cAdvisor Grafana dashboard to quickly start visualizing your container metrics.

### Download the Dashboard

You can download the dashboard JSON file from this [GitHub repository](https://github.com/svenvg93/Grafana-Dashboard/tree/master/cadvisor).

### Import the Dashboard into Grafana

1. Open your Grafana instance and go to Dashboards > Import.
2. Upload the downloaded JSON file.
3. Choose the correct Prometheus datasource.

Once imported, you can begin monitoring the real-time resource usage of your containers using Grafana.
