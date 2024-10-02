# cAdvisor

cAdvisor (Container Advisor) provides container users an understanding of the resource usage and performance characteristics of their running containers. Discover how to set up and use cAdvisor with this guide, featuring step-by-step instructions on installation, configuration, and monitoring container performance metrics for optimized Docker environments.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Directories

Create a **cadvisor** folder, which will hold the needed files.

```shell
mkdir cadvisor
cd cadvisor
```

## Docker

Make the docker compose file containing all information to start the container.

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

## Start cAdvisor

```shell
docker compose up -d
```

## Prometheus Scrape configuration

A Prometheus scrape config for cAdvisor is needed to collect real-time container metrics, essential for monitoring resource usage and optimizing performance in containerized environments.

Add the below to your existing `prometheus.yml`

```yaml
scrape_configs: # Optional is this is your first config
  - job_name: 'cadvisor'
    scrape_interval: 15s
    static_configs:
      - targets: ['<cadvisor>:8080']
```

> Replace `<cadvisor>` with the IP Address of your cAdvisor instance.

## Grafana Dashboard

You can import this [Dashboard](https://github.com/svenvg93/Grafana-Dashboard/tree/master/cadvisor) to get started.
