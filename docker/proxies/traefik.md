# Traefik

Traefik is a powerful open-source reverse proxy and load balancer for HTTP and TCP-based applications. This guide provides a detailed tutorial on setting up Traefik with Let’s Encrypt, covering configuration, SSL certificate automation, and secure deployment for encrypted traffic and scalable web services.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server
* Cloudflare Account
* Cloudflare registered/managed Domain Name
* Port 80 and 443 forwared to your docker host

</details>

## Cloudflare API Key

First we need to create the needed API keys with Cloudflare. This API Key will be used by Traefik durng the DNS challenge for the LetsEncrypt certificate.

Go the [API page](https://dash.cloudflare.com/profile/api-tokens) and login with your Cloudflare Account.

Create a API Key on the link above.

1. Select **Create Token**
2. Select a template **Edit zone DNS**
3. Make sure that the **Permissions** is set to **Zone** / **DNS** / **Edit**
4. By **Zone Resources** you can select for which domain the API key will be used
5. Select **Continue to summary**.
6. Review the token summary. Select **Edit token** to make adjustments. You can also edit a token after creation.
7. Select **Create Token** to generate the token’s secret.

Save this API Key. We will need this later on.

## Create Directories

Create a `treafik` folder to store the configuration files:

```shell
mkdir treafik
cd treafik
```

## Docker Compose Setup

Make the docker compose file containing all information to start the container.

```yaml
services:
  traefik:
    image: traefik
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
      CF_API_EMAIL: ${CF_API_EMAIL}
      CF_DNS_API_TOKEN: ${CF_DNS_API_TOKEN}
    networks:
      - traefik
    ports:
      - 80:80 # HTTP entryPoints
      - 443:443 # HTTPS entryPoints
      - 8080:8080 # Dashboard WebGui 
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Docker socket to watch for Traefik
      - ./traefik.yml:/traefik.yml:ro # Traefik config file
      - traefik-certs:/certs # Docker volume to store the acme file for the Certifactes

volumes:
  traefik-certs:
    name: traefik-certs

networks:
  traefik:
    name: traefik
```

### Environment File

Create a `.env` file in the same directory to store your Cloudflare API key and email:

```title=".env"
CF_API_EMAIL= <Your cloudflare email>
CF_DNS_API_TOKEN= <Your API Token>
```

## Configuration

Create a `traefik.yml` configuration file with the following content:

```yaml
api:
  dashboard: true # Optional can be disabled
  insecure: true # Optional can be disabled
  debug: false # Optional can be Enabled if needed for troubleshooting 
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"
serversTransport:
  insecureSkipVerify: true
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: traefik # Optional; Only use the "traefik" Docker network, even if containers are on multiple networks.
certificatesResolvers:
  letencrypt:
    acme:
      email: youremail@email.com
      storage: /certs/acme.json
      caServer: https://acme-v02.api.letsencrypt.org/directory # prod (default)
      # caServer: https://acme-staging-v02.api.letsencrypt.org/directory # staging
      dnsChallenge:
        provider: cloudflare
        delayBeforeCheck: 10 # Optional to wait x second before checking with the DNS Server
```

> Replace youremail@email.com with your email address for certificate notifications.

## Start Traefik

Start the Traefik container with the following command

```shell
docker compose up -d
```

## Adding a Service

To expose a service through Traefik, add the following labels to the `docker-compose.yml` of that service:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<yourservice>.rule=Host(`test.example.com`)"
  - "traefik.http.routers.<yourservice>.entrypoints=websecure"
  - "traefik.http.routers.<yourservice>.tls=true"
  - "traefik.http.routers.<yourservice>.tls.certresolver=letencrypt"
```

> \* Change `<yourservice>` to a unique name of your service. \* Change `test.example.com` to your domain name to use

## Monitoring

Traefik can expose metrics about EntryPoints, Routers, and Services. We’ll use Prometheus to scrape metrics and Loki to store logs, with Grafana for visualization.

### Metrics

#### Traefik Configuration

Add the following metrics section to your `traefik.yml`:

```yaml
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
```

Metrics can be accessed at `:8080/metrics`.

#### Prometheus Configuration

Add the following to your existing `prometheus.yml`:

```yaml
scrape_configs: # Optional when its the first scrape job
  - job_name: 'traefik'
    scrape_interval: 5s
    static_configs:
      - targets: ['<traefik>:8080']
```
> Replace `<traefik>` with the IP Address of your Traefik server.

Restart Prometheus to apply the changes:

```bash
docker restart prometheus
```

## Grafana Dashboard

You can import an existing Traefik Grafana dashboard to quickly start visualizing your container metrics.

### Download the Dashboard

You can download the dashboard JSON file from this [GitHub repository](https://github.com/svenvg93/Grafana-Dashboard/tree/master/traefik).

### Import the Dashboard into Grafana

1. Open your Grafana instance and go to Dashboards > Import.
2. Upload the downloaded JSON file.
3. Choose the correct Prometheus datasource.

### Logging

You can use either Promtail or Grafana Alloy to send log files to Grafana Loki.

#### Update Traefik Logging Configuration

Add the following accesslog and log sections to your `traefik.yml`:

```yaml
accessLog:
  filePath: "/log/access.log"
  format: json
  fields:
    defaultMode: keep
    names:
      StartUTC: drop

log:
  filePath: "/log/traefik.log"
  format: json
```

Add a volume to your `docker-compose.yml` to mount the log directory:

```yaml
volumes:
  - /var/log/treafik:/log
```

Re-create the Traefik container to apply the new volumes:

```shell
docker compose up -d --force-recreate
```

#### Grafana Alloy Configuration

Add the following to your existing config.alloy:

```yaml
loki.source.file "traefik" {
  targets    = [
	{__path__ = "/logs/treafik/access.log", "job" = "traefik"},
	{__path__ = "/logs/treafik/traefik.log", "job" = "traefik"},	
	]
  forward_to = [loki.write.default.receiver]
}
```

Adjust `forward_to` according to your configuration.

Restart Alloy to apply configuration.

```
docker restart alloy
```


#### Promtail Configuration

Add the following to your existing `promtail-config.yaml`:

```yaml
scrape_configs: # Optional when its the first scrape job
  - job_name: treafik
    static_configs:
    - targets:
        - treafik
      labels:
        job: treafik
        __path__: /logs/treafik/*log
```

Restart Promtail to apply configuration.

```
docker restart promtail
```
