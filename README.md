# Comprehensive Guide to setup RabbitMQ Monitoring with Datadog Integration

## Prerequisites

A system running Ubuntu (the guide assumes commands for Debian-based distributions).
Sudo privileges for the user executing these commands.

## Initial setup

### Step 1: Update your system (Same)

```console
$ sudo apt update
$ sudo apt upgrade
```

### Step 2: Install Prometheus (Ignore for RabbitMQ < 3.8 - Management Plugin)

Download and extract Prometheus

```console
$ wget https://github.com/prometheus/prometheus/releases/download/v2.49.1/prometheus-2.49.1.linux-amd64.tar.gz
$ tar xvzf prometheus-2.49.1.linux-amd64.tar.gz
```

Create necessary directories and copy files

```console
$ sudo mkdir /var/lib/prometheus
$ sudo mkdir /etc/prometheus
$ sudo mkdir /etc/prometheus/consoles
$ sudo mkdir /etc/prometheus/console_libraries

$ sudo cp prometheus-2.49.1.linux-amd64/prometheus /usr/local/bin/
$ sudo cp prometheus-2.49.1.linux-amd64/promtool /usr/local/bin/
$ sudo cp prometheus-2.49.1.linux-amd64/prometheus.yml /etc/prometheus/
$ sudo cp -r prometheus-2.49.1.linux-amd64/consoles /etc/prometheus
$ sudo cp -r prometheus-2.49.1.linux-amd64/console_libraries/ /etc/prometheus
```

Create a Prometheus user and set permissions

```console
$ sudo useradd --no-create-home --shell /bin/false prometheus

$ sudo chown prometheus:prometheus /var/lib/prometheus
$ sudo chown prometheus:prometheus /etc/prometheus
$ sudo chown -R prometheus:prometheus /etc/prometheus/consoles
$ sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Modify the Prometheus configuration file

```console
$ sudo vim /etc/prometheus/prometheus.yml
```

Add the following content under the scrape_configs section

```yml
- job_name: "rabbitmq"
  static_configs:
    - targets: ["localhost:15692"]
```

Create a new systemd service file for Prometheus

```console
$ sudo vim /etc/systemd/system/prometheus.service
```

Paste the following content into the file

```yml
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

Reload systemd, enable, and start Prometheus

```console
$ sudo systemctl daemon-reload
$ sudo systemctl enable prometheus
$ sudo systemctl start prometheus
$ systemctl status prometheus
```

### Step 3: Install Esl-Erlang (Same)

Install required dependencies for Esl-Erlang

```console
$ sudo apt install libncurses5 libsctp1 --fix-broken install
```

Download and install Esl-Erlang

```console
$ wget https://binaries2.erlang-solutions.com/ubuntu/pool/contrib/e/esl-erlang/esl-erlang_26.1.2-1~ubuntu~jammy_amd64.deb
$ sudo dpkg -i esl-erlang_26.1.2-1~ubuntu~jammy_amd64.deb
```

Link for Esl-Erlang old packages - https://packages.erlang-solutions.com/erlang/debian/pool/esl-erlang_21.0-1~ubuntu~bionic_amd64.deb

### Step 4: Install RabbitMQ (Same)

Download and install RabbitMQ

```console
$ wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.12.12/rabbitmq-server_3.12.12-1_all.deb
$ sudo dpkg -i rabbitmq-server_3.12.12-1_all.deb
```

Enable RabbitMQ Plugins

```console
$ sudo rabbitmq-plugins enable rabbitmq_management
$ sudo rabbitmq-plugins enable rabbitmq_prometheus
$ sudo rabbitmq-plugins list
```

Note : On RabbitMQ < 3.8 only ```sudo rabbitmq-plugins enable rabbitmq_management``` can be enabled

Create an Admin User

For RabbitMQ < 3.8 - Management Plugin

```console
$ sudo rabbitmqctl add_user datadog admin
$ sudo rabbitmqctl set_permissions  -p / datadog "^aliveness-test$" "^amq\.default$" ".*"
$ sudo rabbitmqctl set_user_tags datadog monitoring
```

For RabbitMQ >= 3.8 - Prometheus Plugin

```console
$ sudo rabbitmqctl add_user admin admin
$ sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
$ sudo rabbitmqctl set_user_tags admin administrator
```

### Step 4: Installing the Datadog Agent for RabbitMQ Monitoring

1. Install the Datadog Agent
Follow the official Datadog documentation to install the Datadog agent on your system.

2. Configure RabbitMQ Integration:
Configure the Datadog agent to monitor RabbitMQ by editing the conf.yaml file.

```console
$ sudo vim /etc/datadog-agent/conf.d/rabbitmq.d/conf.yaml
```

For RabbitMQ < 3.8 - Management Plugin

```yml
init_config:

instances:
  - rabbitmq_api_url: http://localhost:15672/api/
    rabbitmq_user: datadog
    rabbitmq_pass: admin
```

For RabbitMQ >= 3.8 - Prometheus Plugin

```yml
init_config:

instances:
  - prometheus_plugin:
      url: http://localhost:15692
```

This guide has enhanced the clarity of the instructions, organized the steps for a smoother installation and configuration process, and included additional context where necessary. Always ensure to replace the version numbers and configurations according to your specific requirements or the latest available versions.
