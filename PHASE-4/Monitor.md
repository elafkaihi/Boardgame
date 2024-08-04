# README: Setup Prometheus, Node Exporter, Black Box Exporter, and Grafana on Jenkins and Deployed Website

## Overview

This guide will walk you through the process of installing and setting up Prometheus, Node Exporter, Black Box Exporter, and Grafana for monitoring your Jenkins server and deployed website. 

## Step 1: Install Docker and Docker Compose

If Docker and Docker Compose are not already installed on your server, install them by following these steps:

1. **Install Docker**:
    ```sh
    sudo apt update
    sudo apt install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

2. **Install Docker Compose**:
    ```sh
    sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```

## Step 2: Create Docker Compose File

Create a `docker-compose.yml` file to set up Prometheus, Node Exporter, Black Box Exporter, and Grafana.

```yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus:/etc/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - '9090:9090'

  node_exporter:
    image: prom/node-exporter:latest
    ports:
      - '9100:9100'
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro,rslave'

  blackbox_exporter:
    image: prom/blackbox-exporter:latest
    ports:
      - '9115:9115'
    volumes:
      - ./blackbox:/etc/blackbox

  grafana:
    image: grafana/grafana:latest
    ports:
      - '3000:3000'
    volumes:
      - grafana-storage:/var/lib/grafana

volumes:
  grafana-storage:
```

## Step 3: Configure Prometheus

Create a `prometheus.yml` configuration file for Prometheus.

```yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://example.com  # Replace with your website URL
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox_exporter:9115
```

## Step 4: Configure Black Box Exporter

Create a `blackbox.yml` configuration file for Black Box Exporter.

```yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: [ "HTTP/1.1", "HTTP/2" ]
      valid_status_codes: []  # Defaults to 2xx
      method: GET
      no_follow_redirects: false
      fail_if_ssl: false
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: false
```

## Step 5: Start Services

Run the Docker Compose setup to start all services.

```sh
docker-compose up -d
```

## Step 6: Access Services

- **Prometheus**: `http://<your-server-ip>:9090`
- **Node Exporter**: `http://<your-server-ip>:9100/metrics`
- **Black Box Exporter**: `http://<your-server-ip>:9115`
- **Grafana**: `http://<your-server-ip>:3000`

## Step 7: Configure Grafana

1. Open Grafana in your browser.
2. Log in with the default username and password (`admin`/`admin`).
3. Add Prometheus as a data source:
   - Go to **Configuration** > **Data Sources**.
   - Click **Add data source**.
   - Select **Prometheus**.
   - Set the URL to `http://prometheus:9090`.
   - Click **Save & Test**.

4. Import Grafana dashboards:
   - Go to **Dashboards** > **Manage**.
   - Click **Import**.
   - Use the Grafana.com Dashboard IDs or JSON files to import dashboards for Prometheus, Node Exporter, and Black Box Exporter.

## Step 8: Monitor Jenkins

Add Jenkins job configurations to Prometheus to scrape metrics from Jenkins:

1. Add the Jenkins Prometheus Metrics Plugin to your Jenkins instance.
2. Update your `prometheus.yml` file to include Jenkins:
    ```yml
    - job_name: 'jenkins'
      static_configs:
        - targets: ['<your-jenkins-ip>:8080']
    ```
3. Restart Prometheus to apply changes:
    ```sh
    docker-compose restart prometheus
    ```

## Conclusion

You have successfully set up Prometheus, Node Exporter, Black Box Exporter, and Grafana on Jenkins and your deployed website. You can now monitor your systems and visualize the metrics using Grafana dashboards.