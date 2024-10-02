# Grafana

Grafana is an open-source analytics and monitoring solution compatible with various databases. This guide walks you through the initial setup of Grafana using Docker and covers optional configurations for automated provisioning.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Create Directories

Create a grafana folder to store the configuration files:

```shell
mkdir grafana
cd grafana
```

## Docker Compose Setup

Create a `docker-compose.yml` file in the grafana directory with the following content:

{% code title="docker-compose.yml" %}
```yaml
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: unless-stopped
    environment:
      TZ: Europe/Amsterdam # Change this to your timezone
      GF_USERS_ALLOW_SIGN_UP: false
      GF_AUTH_LOGIN_MAXIMUM_INACTIVE_LIFETIME_DURATION: 2h # Change the duration to your liking
      GF_AUTH_LOGIN_MAXIMUM_LIFETIME_DURATION: 2h # Change the duration to your liking
      GF_AUTH_ANONYMOUS_HIDE_VERSION: true
    network_mode: bridge
    ports:
      - 3000:3000
    volumes:
      - grafana:/var/lib/grafana

volumes:
    grafana:
      name: grafana
```
{% endcode %}

## Start Grafana

Start the Grafana container using the following command:

```shellell
docker compose up -d
```

Access Grafana at `http://<Host IP>:3000`, and log in using the default credentials:

* Username: admin&#x20;
* Password: admin

### Provisioning

Grafana can automatically provision datasources and dashboards, so you won’t need to recreate them each time you rebuild your Grafana instance.

### Create Provisioning Directories

Create the required directories for provisioning files in the same directory as your `docker-compose.yml` file:

```bash
mkdir -p provisioning/dashboards provisioning/datasources
```

#### Update Docker Volumes

Add the following line in the `volumes` section of your `docker-compose.yml` file:

{% code title="docker-compose.yml" %}
```yaml
- ./provisioning:/etc/grafana/provisioning
```
{% endcode %}

#### Create a Datasource Configuration File

Create a file named `datasource.yml` in the `provisioning/datasources` directory. This example sets up Prometheus as a datasource:

{% code title="datasource.yml" %}
```yaml
apiVersion: 1 # Only needed at the beginning of your file
datasources: # Only needed at the beginning of your file
  - name: prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://<prometheus-ip>:9090
    basicAuth: false
    isDefault: true
    editable: false
```
{% endcode %}

> Replace `<prometheus-ip>` with the IP Address of your prometheus server.

#### Create a Dashboard Configuration File

Create a `dashboard.yml` file in the `provisioning/dashboards` directory:

{% code title="dashboard.yml" %}
```yaml
apiVersion: 1 # Only needed at the beginning of your file
providers: # Only needed at the beginning of your file
  - name: 'dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: true
    editable: false
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```
{% endcode %}

#### Add Dashboard JSON Files

Place your dashboard JSON files in the `provisioning/dashboards` directory. Ensure that each dashboard JSON file references the correct datasource matching your Grafana configuration.

#### Restart Grafana to Apply Changes

If you’ve added or updated the provisioning files, restart the Grafana container to apply the changes:

```bash
docker restart grafana
```
