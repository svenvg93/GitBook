# Cloudflare Tunnels

Cloudflare Tunnel is a tunneling software that secures and encrypts application traffic, hiding server IP addresses and blocking direct attacks. This guide will help you configure and securely deploy Cloudflare Tunnel to shield web server IP addresses and create robust, attack-resistant web services.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server
* Cloudflare Account
* Cloudflare registered/managed Domain Name

</details>

## Create Cloudflare Tunnel

Now we need to create a tunnel configuration with Cloudflare.

### Cloudflare tunnel token

* Go to [https://dash.cloudflare.com](https://dash.cloudflare.com/)
* Go to **Zero Trust**
* Click **Networks** -> **Tunnels**
* Click on **Add a Tunnel**
* Select **Cloudflared** and click **Next**
* Give your tunnel name
* Click **Save Tunnel**
* Choose for **Docker** as the installation
* Copy the command with the copy button
* Paste this in a notepad of your liking

## Create Directories

Create a `cloudflared` folder to store the configuration files:

```shell
mkdir cloudflared
cd cloudflared
```

## Docker Compose Setup

Make the docker compose file containing all information to start the container.

{% code title="docker-compose.yml" %}
```yaml
services:
  cloudflared:
    image: cloudflare/cloudflared
    container_name: cloudflared
    restart: unless-stopped
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
      TUNNEL_TOKEN: ${TOKEN}
   network_mode: bridge
   command: tunnel --no-autoupdate run 
```
{% endcode %}

### Environment File

To keep the token secure, create a `.env` file in the same directory as your `docker-compose.yml`:

```yaml
TOKEN=<Your tunnel token>
```

> Replace with the token obtained from the Cloudflare dashboard.

## Start Cloudflared

Run the following command to start Cloudflared:

```bash
docker compose up -d
```

If successful, you will see your tunnel connected within a few seconds. Click **Next** in the Cloudflare dashboard.

## Add a Public Hostname

You don’t need to add a public hostname for Cloudflared itself.

1. Fill in the following fields:&#x20;
   1. Subdomain: e.g., test&#x20;
   2. Domain: Select your domain name from the list.&#x20;
   3. Type: Select HTTP.&#x20;
   4. URL: If the application is on the same Docker network, use the container name. Otherwise, use the IP address and specify the port if it is not the default HTTP port (e.g., :).
2. Click Save Tunnel.

To add more services through the same tunnel, repeat the above steps for each service.

## Monitoring

Cloudflared can expose metrics that can be scraped by Prometheus and displayed in Grafana.

### Metrics

Enable Metrics in Docker Configuration

Add the following environment variable to your `docker-compose.yml`:

```yaml
environment:
    TUNNEL_METRICS: 0.0.0.0:9100
```

Recreate the container to apply the changes:

```bash
docker compose up -d --force-recreate
```

#### Prometheus Configuration

Add the following to your existing `prometheus.yml`:

{% code title="prometheus.yml" %}
```yaml
scrape_configs: # Optional when its the first scrape job
  - job_name: 'cloudflared'
    scrape_interval: 15s
    static_configs:
      - targets: ['<cloudflared>:9100']
```
{% endcode %}

> Replace `<cloudflared>` with the IP Address of your Cloudflared instance.

Restart Prometheus to apply the changes:

```bash
docker restart prometheus
```

## Grafana Dashboard

You can import an existing Cloudflare Tunnel Grafana dashboard to quickly start visualizing your container metrics.

### Download the Dashboard

You can download the dashboard JSON file from this [GitHub repository](https://github.com/svenvg93/Grafana-Dashboard/tree/master/cloudflare\_tunnel).

### Import the Dashboard into Grafana

1. Open your Grafana instance and go to Dashboards > Import.
2. Upload the downloaded JSON file.
3. Choose the correct Prometheus datasource.

### Logging

For logging, you can use either Promtail or Grafana Alloy to send log files to Grafana Loki.

#### Create Logging Directory

Since Cloudflared runs as a rootless container, create a directory for logs:

```bash
sudo mkdir /var/log/cloudflared
sudo chown 65532:65532 /var/log/cloudflared
```

#### Enable Logging in Docker Configuration

Add the following environment variables to your `docker-compose.yml`:

```yaml
environment:
      TUNNEL_LOGLEVEL: info
      TUNNEL_LOGFILE: /log/cloudflared.log
```

Add a volume to mount the log directory:

```yaml
volumes:
  - /var/log/cloudflared:/log
```

Recreate the container to apply the changes:

```bash
docker compose up -d --force-recreate
```

#### Configure Grafana Alloy for Logging

Add the following configuration to your existing `config.alloy`:

{% code title="config.alloy" %}
```bash
loki.source.file "cloudflared" {
  targets    = [{__path__ = "/logs/cloudflared/cloudflared.log", "job" = "cloudflared"},]
  forward_to = [loki.write.default.receiver]
}
```
{% endcode %}

> * Adjust `forward_to` according to your configuration.&#x20;
> * Make sure that the needed volumes are mounted inside your Alloy container.

Restart Alloy to apply the changes:

```bash
docker restart alloy
```

#### Configure Promtail for Logging

Add the following job to your existing `promtail-config.yaml`:

{% code title="promtail-config.yaml" %}
```yaml
scrape_configs: # Optional when its the first scrape job
- job_name: cloudflared
  static_configs:
  - targets:
      - cloudflared
    labels:
      job: cloudflared
      __path__: /logs/cloudflared/tunnel.log
```
{% endcode %}

> Make sure that the needed volumes are mounted inside your Promtail container.

Restart Promtail to apply the changes:

```bash
docker restart promtail
```
