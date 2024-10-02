# Traefik

Traefik is the leading open-source reverse proxy and load balancer for HTTP and TCP-based applications that is easy, dynamic and full-featured. This guide offers a detailed tutorial on setting up Traefik with Let's Encrypt, covering configuration, SSL certificate automation, and secure deployment to ensure encrypted traffic and seamless, scalable web services.

!!! note "Your mileage may vary" The step below might need adjustment to work in your environment!

??? tip "Prerequisites" \* Docker installed on your server \* Cloudflare Account \* Cloudflare registered/managed Domain Name \* Port 80 and 443 forwared to your docker host

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

## Directories

Create a **treafik** folder, which will hold the needed files.

```shell
mkdir treafik
cd treafik
```

## Docker

Make the docker compose file containing all information to start the container.

```yaml
services:
  traefik:
    image: traefik:v3.1.2
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

Create a `.env` file. In this `.env` file we will place the Cloudflare API Key and e-mail address.

```title=".env"
CF_API_EMAIL= <Your cloudflare email>
CF_DNS_API_TOKEN= <Your API Token>
```

## Configuration

Create Traefik configuration file

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

Change `youremail@email.com` too your mail address. It will be used for the registration of the certificates. Lets Encrypt will sent notifications if something is wrong it them.

Start Traefik

```shell
docker compose up -d
```

## Adding a Service

Add the following labels to every `docker-compose.yml` you want to expose with Traefik

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.<yourservice>.rule=Host(`test.example.com`)"
  - "traefik.http.routers.<yourservice>.entrypoints=websecure"
  - "traefik.http.routers.<yourservice>.tls=true"
  - "traefik.http.routers.<yourservice>.tls.certresolver=letencrypt"
```

!!! info \* Change `<yourservice>` to a unique name of your service. \* Change `test.example.com` to your domain name to use

## Monitoring

Traefik can expose metrics about the EntryPoints, Routers and Service etc. We will use Prometheus to scrape the metrics and Loki to store the log files. Grafana will display these metrics and logs.

### Metrics

#### Traefik Configuration

Add the **metrics** sections to your existing `traefik.yml`

```yaml
metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
```

By default Traefik will use the EntryPoint `traefik` to expose the metrics. It can be accessed on `:8080/metrics`

#### Prometheus Configuration

Add the below to your existing `prometheus.yml`.

```yaml
scrape_configs: # Optional when its the first scrape job
  - job_name: 'traefik'
    scrape_interval: 5s
    static_configs:
      - targets: ['<traefik>:8080']
```

!!! info Replace `<traefik>` with the IP Address of your Traefik server.

#### Grafana Dashboard

You can import this [Dashboard](https://github.com/svenvg93/Grafana-Dashboard/tree/master/traefik) to get started.

### Logging

For the Logging you can either use Promtail or Grafana Alloy to grab the log file to be sent to Grafana Loki.

#### Traefik Configuration

Add the **accesslog** and **log** sections to your existing `traefik.yml`

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

Traefik will timestamp each log line in UTC time by default. To make sure the logs will be with our local timezone we drop the `StartUTC`.

add a volume to the Traefik `docker-compose.yml` to mount the right directory in the container.

```yaml
volumes:
  - /var/log/treafik:/log
```

Re-create the Traefik container to add the new volumes.

```shell
docker compose up -d --force-recreate
```

#### Grafana Alloy Configuration

Add the below to your existing `config.alloy`

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

#### Promtail Configuration

Add the below to your existing `promtail-config.yaml`

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
