# Monitoring Caddy Server with Grafana (Prometheus + Loki) on Debian

![Architecture Caddy-Grafana](https://github.com/Malfhas/caddy-grafana/assets/13700925/f50f7702-078f-4bdf-9a76-6a8c197fd6c6)

# Prerequisites
- Caddy Server already installed and functional
- Server using Debian (it's up to you to adapt the tutorial according to your distribution)

# Caddy setup
In order to monitor the data on our Caddy server, we need to activate metrics and logs.

## Metrics activation
Activating the Caddy metrics (by using its API) will enable Prometheus to collect the data.
To do this, you need to modify the "Caddyfile":
```console
sudo nano /etc/caddy/Caddyfile
```

It is necessary to add "metrics" to the global options and that the admin endpoint is not set to off. If you have the "admin off" command, this will disable Caddy's REST API and make it impossible for metrics to work.
```console
{
    servers {
            metrics
     }
}
```

Restart the Caddy service:
```console
sudo systemctl restart caddy
```

Check that the service is active:
```console
sudo systemctl status caddy
```

## Logs activation
To be able to monitor Caddy logs via Loki, you need to configure them correctly.
To do this, I'll take the example of my configuration.

I have 3 log files to monitor:
- "caddy_main.log": my global log file
- "domain.com.log": My log file for my main domain
- "grafana.domain.com.log": My log file for my sub-domain.

These files are located in "/var/log/caddy".

To do this, add these lines to the global options in "Caddyfile":
```console
{
    log {
            output file /var/log/caddy/caddy_main.log {
                      roll_size 100MiB
                      roll_keep 5
                      roll_keep_for 100d
            }
            format json
            level INFO
     }
}
```

And the same goes for your domains/subdomains:
```console
domaine.com {
    log {
            output file /var/log/caddy/domain.com.log {
                      roll_size 100MiB
                      roll_keep 5
                      roll_keep_for 100d
            }
            format json
            level INFO
     }
}

grafana.domaine.com {
    log {
            output file /var/log/caddy/grafana.domain.com.log {
                      roll_size 100MiB
                      roll_keep 5
                      roll_keep_for 100d
            }
            format json
            level INFO
     }
}
```

Restart the Caddy service:
```console
sudo systemctl restart caddy
```

Check that the service is active:
```console
sudo systemctl status caddy
```

# Prometheus setup
Installing Prometheus:
```console
sudo apt update
sudo apt install prometheus prometheus-node-exporter
```

Modifying the configuration file:
```console
sudo nano /etc/prometheus/prometheus.yml
```

Added lines below "scrape_configs":
```console
- job_name: caddy
      static_configs:
          - targets: ['localhost:2019']
```

You should have a configuration similar to this:
```
# Sample config for Prometheus.

global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'example'

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
#rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    scrape_timeout: 5s

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: node
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['localhost:9100']

  - job_name: caddy
    static_configs:
      - targets: ['localhost:2019']
```

Restart the Prometheus service:
```console
sudo systemctl restart prometheus
```

Check that the service is active:
```console
sudo systemctl status prometheus
```

# Loki & Promtail setup
Add Grafana packages:
```console
# mkdir -p /etc/apt/keyrings/
# wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor > /etc/apt/keyrings/grafana.gpg
# echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | tee /etc/apt/sources.list.d/grafana.list
```

Install Loki and Promtail:
```console
apt update
apt install loki promtail
```

Edit the Promtail configuration file:
```console
sudo nano /etc/promtail/config.yml
```

Add the following lines at the end of the file:
```console
- job_name: caddy
  static_configs:
  - targets:
      - localhost
    labels:
      job: caddy
      __path__: /var/log/caddy/*log
      agent: caddy-promtail
  pipeline_stages:
  - json:
      expressions:
        duration: duration
        status: status
  - labels:
      duration:
      status:
```

You should have a configuration similar to this:
```
# This minimal config scrape only single log file.
# Primarily used in rpm/deb packaging where promtail service can be started during system init process.
# And too much scraping during init process can overload the complete system.
# https://github.com/grafana/loki/issues/11398

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
- url: http://localhost:3100/loki/api/v1/push

scrape_configs:
- job_name: system
  static_configs:
  - targets:
      - localhost
    labels:
      job: varlogs
      #NOTE: Need to be modified to scrape any additional logs of the system.
      __path__: /var/log/messages

- job_name: caddy
  static_configs:
  - targets:
      - localhost
    labels:
      job: caddy
      __path__: /var/log/caddy/*log
      agent: caddy-promtail
  pipeline_stages:
  - json:
      expressions:
        duration: duration
        status: status
  - labels:
     duration:
     status:
```

Restart Promtail and Loki services:
```console
sudo systemctl restart promtail
sudo systemctl restart loki
```

Check that the services is active:
```console
sudo systemctl status promtail
sudo systemctl status loki
```

# Grafana setup
## Installing Grafana
```console
sudo apt-get install -y apt-transport-https software-properties-common wget
```
```console
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
```
```console
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```
```console
sudo apt update
```
```console
sudo apt install grafana-enterprise
```

## Adding Prometheus as a data source
![image](https://github.com/Malfhas/caddy-grafana/assets/13700925/4ed1bd13-2b54-4626-aee9-ba55177f38a8)

## Adding Loki as a data source
![image](https://github.com/Malfhas/caddy-grafana/assets/13700925/15fd9b10-4033-4e61-98ac-aee5f9bee067)

# First tests with the Grafana explorer
## Testing a Caddy API query via Prometheus
![image](https://github.com/Malfhas/caddy-grafana/assets/13700925/2d7cb415-a8c3-4c6e-96bd-97c9bca8ce40)

## Testing a query on Caddy logs via Loki
![image](https://github.com/Malfhas/caddy-grafana/assets/13700925/7c7830af-8c27-438b-885b-f812d8605f64)

# My Grafana Dashboard
This Dashboard integrates Prometheus + Loki.

This is a V1 which is not finished and not verified.

It can nevertheless serve as a basis for making your own Dashoard

![Dashboard1](https://github.com/Malfhas/caddy-grafana/assets/13700925/65e5dc9b-067c-4122-aa7f-986171fb3aba)

![Dashboard2](https://github.com/Malfhas/caddy-grafana/assets/13700925/8238c33e-9556-42ba-8de7-82edf3934073)

Dashboard link : Coming soon...










