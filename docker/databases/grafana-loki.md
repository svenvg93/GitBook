# Grafana Loki

Grafana Loki is a horizontally scalable, highly available, multi-tenant log aggregation system inspired by Prometheus. This guide provides detailed instructions for setting up Loki, covering installation, configuration, and integration with Grafana to streamline log aggregation and visualization.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Create Directories

Create a loki folder to store the configuration files:

```shell
mkdir loki
cd loki
```

## Docker Compose Setup

Create a `docker-compose.yml` file in the loki directory with the following content:

{% code title="docker-compose.yml" %}
```yaml
services:
  loki:
    image: grafana/loki
    container_name: loki
    restart: unless-stopped
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
    network_mode: bridge
    ports:
      - 3100:3100
    volumes:
      - ./loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki:/tmp
    command: -config.file=/etc/loki/loki-config.yaml

volumes:
    loki:
      name: loki
```
{% endcode %}

## Create the Loki Configuration File

Create a file named loki-config.yaml to define how Loki handles log ingestion and storage:

{% code title="loki-config.yaml" %}
```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory  # Consider using 'consul' or 'etcd' for production

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

querier:
  max_concurrent: 500  # Adjust based on CPU and memory

query_scheduler:
  max_outstanding_requests_per_tenant: 1000  # Adjust based on load

frontend:
  max_outstanding_per_tenant: 2000  # Adjust based on load

limits_config:
  max_global_streams_per_user: 5000  # Adjust based on actual usage
  ingestion_rate_mb: 50  # Adjust based on actual load
  per_stream_rate_limit: 50MB  # Adjust based on actual load
```
{% endcode %}

## Start Loki

Start the Loki container using the following command:

```shell
docker compose up -d
```

## Adding Loki as a Grafana Datasource

Loki can be integrated into Grafana to visualize log data. This can be configured through the Grafana web interface or using provisioning.

### Using the Grafana Web Interface

1. Go to **Connections** in the left-side menu.
2. Search for **Loki**.
3. Click **Add new Datasource.**
4. Enter **loki** as the name.
5. Set the URL to **http://\<loki-ip>:3100**.

> Replace `<loki-ip>` with the IP Address of your Loki server.

### Using Grafana Provisioning

To automate the process, add the following configuration to your `datasource.yml` file in Grafana’s provisioning directory:

{% code title="datasource.yml" %}
```yaml
apiVersion: 1
datasources:
  - name: loki
    type: loki
    access: proxy
    url: http://<loki-ip>:3100
    jsonData:
      timeout: 60
      maxLines: 1000
```
{% endcode %}

> Replace `<loki-ip>` with the IP Address of your Loki server.

#### Restart Grafana

If you used provisioning to set up the datasource, restart the Grafana container
