# Grafana

Grafana is the open source analytics & monitoring solution for every database. This guide walks you through getting started with Grafana.

{% hint style="info" %}
The step below might need adjustment to work in your environment!
{% endhint %}

<details>

<summary>Prerequisites</summary>

* Docker installed on your server

</details>

## Directories

Create a `grafana` folder, which will hold the needed files.

```shell
mkdir grafana
cd grafana
```

## Docker

Make the docker compose file containing all information to start the container.

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

```shellell
docker compose up -d
```

Go to `http://<Host IP>:3000` , login with `admin`/`admin`.

## Optional

You can configure Grafana using environment variables, allowing you to avoid reconfiguration every time you rebuild your Grafana instance.

### Configuration

Add the following lines to the environment section of your `docker-compose.yml` file:

{% code title="docker-compose.yml" %}
```
GF_SECURITY_ADMIN_USER: ${GF_SECURITY_ADMIN_USER}
GF_SECURITY_ADMIN_PASSWORD: ${GF_SECURITY_ADMIN_PASSWORD}
```
{% endcode %}

Create a `.env` file to store your settings:

<pre class="language-bash" data-title=".env"><code class="lang-bash">GF_SECURITY_ADMIN_USER=YourUsername
<strong>GF_SECURITY_ADMIN_PASSWORD=YourPassword
</strong></code></pre>

### Provisioning

Grafana can automatically provision datasources and dashboards, so you don't have to recreate them every time you rebuild your Grafana instance.

#### Directories

Grafana expects the **dashboards** and **datasource** files to be in a certain directory.

Make sure your in the same directory as where your `docker-compose.yml` for grafana is.

```bash
mkdir -p provisioning/dashboards provisioning/datasources
```

#### Docker Volumes

Add the following line in the `volume` section of your `docker-compose.yml`

{% code title="docker-compose.yml" %}
```yaml
- ./provisioning:/etc/grafana/provisioning
```
{% endcode %}

#### Datasource

Make a `datasource.yml` file in the **provisioning/datasources** directory.

This example adds Prometheus as datasource.

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

#### Dashboards

Make a `dashboard.yml` file in the **provisioning/dashboards** directory.

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

Place the Dashboard **json** files in **provisioning/dashboards**.

Make sure the json files have the correct datasource set matching your grafana datasources
