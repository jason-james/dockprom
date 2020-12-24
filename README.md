# Minamonitor

A monitoring solution for Docker hosts and containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor),
[NodeExporter](https://github.com/prometheus/node_exporter) and alerting with [AlertManager](https://github.com/prometheus/alertmanager).

## Install

Clone this repository on your Docker host, cd into minamonitor directory, add your node address in [prometheus/prometheus.yml](/prometheus/prometheus.yml) and run docker-compose up:

```bash
$ git clone https://github.com/jason-james/minamonitor
$ cd minamonitor
$ vim prometheus/prometheus.yml # Then add your node address on line 42 and save
$ ADMIN_USER=admin ADMIN_PASSWORD=admin docker-compose up -d
```

Prerequisites:

- Docker Engine >= 1.13
- Docker Compose >= 1.11

Containers:

- Prometheus (metrics database) `http://<host-ip>:9090`
- Prometheus-Pushgateway (push acceptor for ephemeral and batch jobs) `http://<host-ip>:9091`
- AlertManager (alerts management) `http://<host-ip>:9093`
- Grafana (visualize metrics) `http://<host-ip>:3000`
- NodeExporter (host metrics collector)
- cAdvisor (containers metrics collector)
- Caddy (reverse proxy and basic auth provider for prometheus and alertmanager)

## Setup Grafana

Navigate to `http://<host-ip>:3000` and login with user **_admin_** password **_admin_**. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables on compose up. The config file can be added directly in grafana part like this

```
grafana:
  image: grafana/grafana:7.2.0
  env_file:
    - config

```

and the config file format should have this content

```
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme
GF_USERS_ALLOW_SIGN_UP=false
```

If you want to change the password, you have to remove this entry, otherwise the change will not take effect

```
- grafana_data:/var/lib/grafana
```

Grafana is preconfigured with dashboards and Prometheus as the default data source:

- Name: Prometheus
- Type: Prometheus
- Url: http://prometheus:9090
- Access: proxy

**_Mina Blockchain Metrics Dashboard_**

The Mina Metrics dashboard shows key metrics for your node and the Mina network. Use it to keep track of uptime, hardware info, and block production statistics. An equivalent
dashboard for Snark Producing is coming soon.

![Mina](screens/Mina.png)

For your Mina node metrics to be picked up, you must ensure you pass the `-metrics-port 6060` flag when you start your node, and add your node's IP address on line 42 of `/prometheus/prometheus.yml`. If you are running your Mina node via Docker, you must also add `-p 6060:6060` to your docker run command.

- Docker host health statistics. Server uptime, CPU idle percent, available memory, swap and storage. If you use docker, you can track the docker container's stats too (see below).
- Node uptime, blocks produced, slots missed, peers connected, average messages received, block gossip latency, Mina TPS, average slots won

**_Docker Host Dashboard_**

![Host](/screens/Grafana_Docker_Host.png)

The Docker Host Dashboard shows key metrics for monitoring the resource usage of your server:

- Server uptime, CPU idle percent, number of CPU cores, available memory, swap and storage
- System load average graph, running and blocked by IO processes graph, interrupts graph
- CPU usage graph by mode (guest, idle, iowait, irq, nice, softirq, steal, system, user)
- Memory usage graph by distribution (used, free, buffers, cached)
- IO usage graph (read Bps, read Bps and IO time)
- Network usage graph by device (inbound Bps, Outbound Bps)
- Swap usage and activity graphs

For storage and particularly Free Storage graph, you have to specify the fstype in grafana graph request.
You can find it in `grafana/dashboards/docker_host.json`, at line 480 :

      "expr": "sum(node_filesystem_free_bytes{fstype=\"btrfs\"})",

I work on BTRFS, so i need to change `aufs` to `btrfs`.

You can find right value for your system in Prometheus `http://<host-ip>:9090` launching this request :

      node_filesystem_free_bytes

**_Docker Containers Dashboard_**

![Containers](/screens/Grafana_Docker_Containers.png)

The Docker Containers Dashboard shows key metrics for monitoring running containers:

- Total containers CPU load, memory and storage usage
- Running containers graph, system load graph, IO usage graph
- Container CPU usage graph
- Container memory usage graph
- Container cached memory usage graph
- Container network inbound usage graph
- Container network outbound usage graph

If you run your Mina node via Docker, you will see its statistics here.

Note that this dashboard doesn't show the containers that are part of the monitoring stack.

**_Monitor Services Dashboard_**

![Monitor Services](/screens/Grafana_Prometheus.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

- Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
- Container CPU usage graph
- Container memory usage graph
- Prometheus chunks to persist and persistence urgency graphs
- Prometheus chunks ops and checkpoint duration graphs
- Prometheus samples ingested rate, target scrapes and scrape duration graphs
- Prometheus HTTP requests graph
- Prometheus alerts graph

## Define alerts

Three alert groups have been setup within the [alert.rules](/prometheus/alert.rules) configuration file:

- Monitoring services alerts [targets](/prometheus/alert.rules#L2-L11)
- Docker Host alerts [host](/prometheus/alert.rules#L13-L40)
- Docker Containers alerts [containers](/prometheus/alert.rules#L42-L69)

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

**_Monitoring services alerts_**

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
- alert: monitor_service_down
    expr: up == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Monitor service non-operational"
      description: "Service {{ $labels.instance }} is down."
```

**_Docker Host alerts_**

Trigger an alert if the Docker host CPU is under high load for more than 30 seconds:

```yaml
- alert: high_cpu_load
    expr: node_load1 > 1.5
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server under high load"
      description: "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

Modify the load threshold based on your CPU cores.

Trigger an alert if the Docker host memory is almost full:

```yaml
- alert: high_memory_load
    expr: (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) ) / sum(node_memory_MemTotal_bytes) * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server memory is almost full"
      description: "Docker host memory usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

Trigger an alert if the Docker host storage is almost full:

```yaml
- alert: high_storage_load
    expr: (node_filesystem_size_bytes{fstype="aufs"} - node_filesystem_free_bytes{fstype="aufs"}) / node_filesystem_size_bytes{fstype="aufs"}  * 100 > 85
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Server storage is almost full"
      description: "Docker host storage usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}."
```

**_Docker Containers alerts_**

Trigger an alert if a container is down for more than 30 seconds:

```yaml
- alert: jenkins_down
    expr: absent(container_memory_usage_bytes{name="jenkins"})
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Jenkins down"
      description: "Jenkins container is down for more than 30 seconds."
```

Trigger an alert if a container is using more than 10% of total CPU cores for more than 30 seconds:

```yaml
- alert: jenkins_high_cpu
    expr: sum(rate(container_cpu_usage_seconds_total{name="jenkins"}[1m])) / count(node_cpu_seconds_total{mode="system"}) * 100 > 10
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Jenkins high CPU usage"
      description: "Jenkins CPU usage is {{ humanize $value}}%."
```

Trigger an alert if a container is using more than 1.2GB of RAM for more than 30 seconds:

```yaml
- alert: jenkins_high_memory
    expr: sum(container_memory_usage_bytes{name="jenkins"}) > 1200000000
    for: 30s
    labels:
      severity: warning
    annotations:
      summary: "Jenkins high memory usage"
      description: "Jenkins memory consumption is at {{ humanize $value}}."
```

## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose **_incoming web hooks_** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the **_api_url_** field and specify a Slack **_channel_**.

```yaml
route:
  receiver: 'slack'

receivers:
  - name: 'slack'
    slack_configs:
      - send_resolved: true
        text: '{{ .CommonAnnotations.description }}'
        username: 'Prometheus'
        channel: '#<channel>'
        api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](/screens/Slack_Notifications.png)

## Sending metrics to the Pushgateway

The [pushgateway](https://github.com/prometheus/pushgateway) is used to collect data from batch jobs or from services.

To push data, simply execute:

    echo "some_metric 3.14" | curl --data-binary @- http://user:password@localhost:9091/metrics/job/some_job

Please replace the `user:password` part with your user and password set in the initial configuration (default: `admin:admin`).

Thanks to stefanprodan for Dockprom, which Minamonitor is built upon.
