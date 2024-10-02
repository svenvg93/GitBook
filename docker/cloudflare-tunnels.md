# Cloudflare Tunnels

Cloudflare Tunnel is tunneling software that secures and encrypts application traffic, hiding server IP addresses and blocking direct attacks, allowing you to focus on delivering great applications. Unlock the secrets of Cloudflare Tunnel with this all-inclusive guide! Learn how to configure, encrypt traffic, and securely deploy your setup to shield web server IP addresses and create rock-solid, attack-resistant web services.

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

## Directories

Create a **cloudflared** folder, which will hold the needed files.

```shell
mkdir cloudflared
cd cloudflared
```

## Docker

Make the docker compose file containing all information to start the container.

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

### Environment File

As we don't want the token to be in the docker-compose file we create a **.**`env` file in the same directory as your `docker-compose.yml`.

In this `.env` file place the following content.

```yaml
TOKEN=<Your tunnel token>
```

Replace the `<Your tunnel token>` with the long strong of characters that it behind the `-- token` of the command we just copied from the Cloudflare website.

Now we will start Cloudlfared by running:

```bash
docker compose up -d
```

If everything is correct you will see your tunnel connected within a couple of seconds. When its conncted click **Next**.

## Adding a Public Hostname

You don't need to add a public hostname for clouflared itself.

Fill in the follwing field:

* **Subdomain** — the subdomain to use for example; **test**
* **Domain** — pick the domain name to use form the list;
* **Type** — Select **HTTP**
* **URL** — If the application is on the same docker network you can use the container name. If thats not the case use the IP Address. If it does not use the default http port specifiy the ports with `:<portnumber>` behind the IP Address/Container name.

As last step click **Save Tunnel.** Now the tunnel will get the configuration and you can access your service via Cloudflare.

If you want to add more services through the same tunnel. Go to the **Networks** -> **Tunnels** page

* Click on the Tunnel you want to add the Public Hostname to.
* Click on **Edit**
* Go to **Public Hostname**
* Click on **Add a public hostname**
* Fill in the fields **Subdomain**, **Domain**, **Type**, **URL**. Like described above.

## Monitoring

Cloudflared can expose metrics about the tunnels. We will use Prometheus to scrape the metrics and Grafana to display them.

### Metrics

#### Docker Configuration

In order to enable the endpoint we need to add the environment variables below to the `docker-compose.yml`

```yaml
environment:
    TUNNEL_METRICS: 0.0.0.0:9100
```

We will need to recreate the contianer for the environment variables to be picked up.

```bash
docker compose up -d --force-recreate
```

#### Prometheus Configuration

Add the below to your existing `prometheus.yml`

```yaml
scrape_configs: # Optional when its the first scrape job
  - job_name: 'cloudflared'
    scrape_interval: 15s
    static_configs:
      - targets: ['<cloudflared>:9100']
```

> Replace `<cloudflared>` with the IP Address of your Cloudflared instance.

To apply all the configuration changes we made we need to Prometheus.

```bash
docker restart prometheus
```

#### Grafana Dashboard

You can import this [Dashboard](https://github.com/svenvg93/Grafana-Dashboard/tree/master/cloudflare\_tunnel) to get started.

### Logging

For the Logging you can either use Promtail or Grafana Alloy to grab the log file to be sent to Grafana Loki.

#### Directory

Because cloudflared a rootless container we need make the directory and give it the needed rights

```bash
sudo mkdir /var/log/cloudflared
sudo chown 65532:65532 /var/log/cloudflared
```

#### Docker Configuration

In order to enable the logging we need to add the environment variables below to the `docker-compose.yml`.

```yaml
environment:
      TUNNEL_LOGLEVEL: info
      TUNNEL_LOGFILE: /log/cloudflared.log
```

Add a volume to mount the right directory in the container.

```yaml
volumes:
  - /var/log/cloudflared:/log
```

We will need to recreate the contianer for the environment variables to be picked up.

```bash
docker compose up -d --force-recreate
```

#### Grafana Alloy Configuration

Add the below to your existing `config.alloy`.

```bash
loki.source.file "cloudflared" {
  targets    = [{__path__ = "/logs/cloudflared/cloudflared.log", "job" = "cloudflared"},]
  forward_to = [loki.write.default.receiver]
}
```

> \* Adjust `forward_to` according to your configuration. \* Make sure that the needed volumes are mounted inside your Alloy container.

#### Promtail Configuration

Add job to your existing `promtail-config.yaml` configuration

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

> Make sure that the needed volumes are mounted inside your Promtail container.

Restart Promtail to apply configuration.

```bash
docker restart promtail
```
