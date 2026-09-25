# Linux Observability Lab

A multi-node Linux observability lab built to monitor Linux infrastructure metrics, centralize system logs, visualize system health, and deliver infrastructure alerts.

The project started with an Ubuntu Server and Rocky Linux server being monitored by an Ubuntu Desktop machine acting as the central observability platform. It was later expanded with a CentOS server to create a three-node monitoring environment.

The logging pipeline was also migrated from Promtail to Grafana Alloy, with Alloy running natively as a systemd-managed service on the monitored Linux systems.

Ansible was subsequently introduced to automate the configuration and management of Node Exporter and Grafana Alloy across the monitoring nodes.

## Architecture

```text
                         Ubuntu Desktop
                     Observability Server
                  ┌─────────────────────────┐
                  │                         │
                  │      Prometheus         │
                  │        :9090            │
                  │                         │
                  │        Grafana          │
                  │        :3000            │
                  │                         │
                  │         Loki            │
                  │        :3100            │
                  │                         │
                  │      Alertmanager       │
                  │        :9093            │
                  │                         │
                  │    Alloy (Docker)       │
                  │                         │
                  └────────────┬────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              │                │                │
              ▼                ▼                ▼
       Rocky Server      Ubuntu Server      CentOS Server
       192.168.0.10      192.168.0.20       192.168.0.40
              │                │                │
              │                │                │
        Node Exporter    Node Exporter    Node Exporter
           :9100            :9100            :9100
              │                │                │
           Alloy            Alloy            Alloy
          systemd           systemd           systemd
              │                │                │
              └────────────────┼────────────────┘
                               │
                         Journald Logs
                               │
                               ▼
                              Loki
                               │
                               ▼
                            Grafana

                         Alerting Flow
                         
        Node Exporter
              │
              ▼
          Prometheus
              │
         Alert Rules
              │
              ▼
         Alertmanager
              │
              ▼
           Discord
```

## Project Evolution

### Initial Implementation

The original lab consisted of:

- Ubuntu Desktop as the observability server
- Ubuntu Server as a monitored node
- Rocky Linux as a monitored node
- Prometheus for metrics collection
- Node Exporter for Linux host metrics
- Grafana for visualization
- Loki for centralized logging
- Promtail for log collection
- Alertmanager for alert routing
- Discord for alert notifications

This provided the foundation for learning Linux monitoring, centralized logging, dashboards, and infrastructure alerting.

### Expansion to Three Monitoring Nodes

The lab was expanded with a CentOS server.

The current monitored infrastructure consists of:

| Host | Role | Metrics | Logs |
|---|---|---|---|
| Ubuntu Desktop | Observability server | Node Exporter | Alloy |
| Ubuntu Server | Monitoring node | Node Exporter | Alloy |
| Rocky Linux | Monitoring node | Node Exporter | Alloy |
| CentOS | Monitoring node | Node Exporter | Alloy |

The Ubuntu Desktop observability server hosts the central monitoring stack.

The Ubuntu, Rocky, and CentOS systems act as monitored Linux nodes.

## Technology Stack

| Component | Purpose |
|---|---|
| Prometheus | Metrics collection and alert rule evaluation |
| Node Exporter | Linux host metrics |
| Grafana | Metrics and log visualization |
| Loki | Centralized log storage |
| Grafana Alloy | System journal collection and forwarding |
| Alertmanager | Alert routing and notification management |
| Discord | External alert notifications |
| Docker Compose | Runs the central observability stack |
| Ansible | Configuration management and automation |
| systemd | Service management for Node Exporter and Alloy on monitoring nodes |

## Metrics Monitoring

Prometheus collects infrastructure metrics from Node Exporter running on the Linux systems.

The monitored infrastructure provides visibility into:

- CPU utilization
- Memory utilization
- Disk utilization
- Filesystem usage
- System load
- Network traffic
- Host availability
- Node Exporter health

Prometheus also evaluates infrastructure alert rules and monitors the availability of configured scrape targets.

The current Prometheus targets include:

```text
prometheus:9090
node-exporter:9100
192.168.0.10:9100
192.168.0.20:9100
192.168.0.40:9100
```

The first two targets represent services running on the observability platform, while the remaining targets represent the Rocky, Ubuntu, and CentOS monitoring nodes.

## Centralized Logging

The original implementation used Promtail for log collection.

The project was later migrated to Grafana Alloy.

Alloy now runs directly on the three monitored Linux systems as a native systemd-managed service rather than as a Docker container.

Each monitoring node reads persistent systemd journal data from:

```text
/var/log/journal
```

and forwards the logs to Loki on the observability server.

The current logging pipeline is:

```text
Linux systemd journal
        │
        ▼
Grafana Alloy
        │
        ▼
      Loki
        │
        ▼
     Grafana
```

Each host adds its own hostname as a Loki label, allowing logs to be queried by individual server.

Example LogQL queries:

```logql
{host="ubuntu-server"}
```

```logql
{host="rocky-server"}
```

```logql
{host="cent-server"}
```

Logs can also be filtered for specific messages:

```logql
{job="journal"} |= "error"
```

## Grafana Alloy

Grafana Alloy is installed natively on the three monitored Linux systems.

Alloy is managed by systemd:

```bash
systemctl status alloy
```

The Alloy service:

- Reads persistent journald logs
- Adds host-specific labels
- Forwards logs to Loki
- Runs continuously as a system service
- Starts automatically with the system

The monitored nodes therefore do not require Docker to run their logging agent.

The observability server retains its separate Alloy deployment as part of the central observability environment.

## Alerting

Prometheus evaluates infrastructure alert rules and sends firing alerts to Alertmanager.

Configured alerts include conditions such as:

- Instance down
- High CPU utilization
- High memory utilization
- High disk utilization

The alert flow is:

```text
Node Exporter
     │
     ▼
Prometheus
     │
     ▼
Alert Rule
     │
     ▼
Alertmanager
     │
     ▼
Discord
```

Alertmanager handles alert routing and notification delivery.

The lab has been tested by generating resource pressure on monitored systems and verifying that Prometheus detects the condition, Alertmanager processes the alert, and Discord receives the notification.

## Ansible Configuration Management

Ansible was introduced to automate the configuration of the three monitoring nodes.

The Ansible project is located in:

```text
ansible/
├── ansible.cfg
├── group_vars/
│   └── monitoring_nodes.yml
├── inventory/
│   └── hosts
└── playbooks/
    ├── files/
    │   ├── prometheus-node-exporter.j2
    │   └── config.alloy.j2
    ├── alloy.yml
    └── node_exporter.yml
```

The inventory defines the three monitoring nodes:

```text
rocky-server
ubuntu-server
cent-server
```

### Node Exporter Automation

Ansible manages the Node Exporter configuration and systemd service.

The playbook ensures that:

- Node Exporter configuration exists
- Configuration has the expected ownership and permissions
- Node Exporter is enabled
- Node Exporter is running

### Alloy Automation

Ansible also manages Grafana Alloy on the three monitoring nodes.

The Alloy playbook ensures that:

- Grafana Alloy is installed
- The Alloy configuration exists
- The configuration is generated from an Ansible template
- Each host receives its own hostname label
- Alloy is enabled
- Alloy is running
- Alloy is restarted when its configuration changes

The Alloy playbook was tested for idempotency so that subsequent runs do not unnecessarily modify the systems or restart the service.

Example:

```bash
ansible-playbook playbooks/alloy.yml
```

Check mode can be used before applying changes:

```bash
ansible-playbook playbooks/alloy.yml --check --diff
```

Connectivity can be tested with:

```bash
ansible monitoring_nodes -m ping
```

Service state can be checked with:

```bash
ansible monitoring_nodes -a "systemctl is-active alloy"
```

and:

```bash
ansible monitoring_nodes -a "systemctl is-enabled alloy"
```

## Project Structure

```text
linux-observability-lab/
├── alertmanager/
│   └── alertmanager.yml.template
├── alloy/
├── ansible/
│   ├── ansible.cfg
│   ├── group_vars/
│   │   └── monitoring_nodes.yml
│   ├── inventory/
│   │   └── hosts
│   └── playbooks/
│       ├── files/
│       │   ├── prometheus-node-exporter.j2
│       │   └── config.alloy.j2
│       ├── alloy.yml
│       └── node_exporter.yml
├── docs/
│   └── screenshots/
├── loki/
│   └── loki-config.yml
├── prometheus/
│   ├── prometheus.yml
│   └── alerts.yml
├── docker-compose.yml
└── README.md
```

## Running the Central Observability Stack

The central observability components run on the Ubuntu Desktop observability server.

Clone the repository:

```bash
git clone https://github.com/OB-Adams/linux-observability-lab.git
cd linux-observability-lab
```

Start the central stack:

```bash
docker compose up -d
```

Check the containers:

```bash
docker compose ps
```

The central stack uses Docker Compose, while the three monitored nodes run their Node Exporter and Alloy services natively through systemd.

## Running the Ansible Automation

From the repository root:

```bash
cd ansible
```

Test connectivity:

```bash
ansible monitoring_nodes -m ping
```

Manage Node Exporter:

```bash
ansible-playbook playbooks/node_exporter.yml
```

Manage Grafana Alloy:

```bash
ansible-playbook playbooks/alloy.yml
```

Check the Alloy configuration without making changes:

```bash
ansible-playbook playbooks/alloy.yml --check --diff
```

Verify Alloy across the monitoring nodes:

```bash
ansible monitoring_nodes -a "systemctl is-active alloy"
```

```bash
ansible monitoring_nodes -a "systemctl is-enabled alloy"
```

## Access

| Service | Port |
|---|---:|
| Grafana | 3000 |
| Prometheus | 9090 |
| Alertmanager | 9093 |
| Loki | 3100 |
| Node Exporter | 9100 |
| Alloy HTTP API | 12345 |

## Security

The Discord webhook is kept outside version control.

The Alertmanager configuration uses an environment variable rather than storing the webhook directly in the repository.

Generated configuration containing the webhook is excluded from Git.

This keeps notification credentials out of the repository.

## What I Learned

This project was built as a practical Linux and infrastructure engineering lab.

Key areas explored include:

- Linux system administration
- Ubuntu Server
- Rocky Linux
- CentOS
- systemd
- systemd journal
- journald persistence
- Prometheus
- Node Exporter
- Grafana
- Loki
- Grafana Alloy
- LogQL
- Prometheus alert rules
- Alertmanager
- Discord notifications
- Docker Compose
- Ansible inventory
- Ansible variables
- Ansible templates
- Ansible systemd management
- Idempotent configuration management
- Multi-node monitoring
- Centralized logging
- Infrastructure alerting

## Screenshots

### Grafana Infrastructure Dashboard

Overview of infrastructure metrics collected from the monitored Linux systems.

### Centralized Logs

System logs from the monitored Linux servers centralized in Loki and visualized through Grafana.

### Prometheus Targets

Prometheus showing the monitored Linux systems and their scrape status.

### Prometheus Alert

Prometheus detecting an infrastructure condition and transitioning the alert through its evaluation states.

### Alertmanager Notification

Alertmanager receiving and processing a firing infrastructure alert.

### Discord Alert

Alertmanager delivering the infrastructure alert to Discord.

## Future Work

This project focuses on building and operating the observability stack itself.

The next infrastructure project will be developed separately and will focus on using Terraform and Ansible to provision and configure Linux infrastructure from the ground up.

The next project will deliberately avoid depending on Docker for the underlying Linux systems and will focus more deeply on:

- Terraform
- Ansible
- Linux system provisioning
- Network configuration
- System configuration
- Service management
- Infrastructure automation
- Configuration management
- Infrastructure as Code

## Author

OB Adams

Linux, Cloud, DevOps, and Infrastructure Automation
