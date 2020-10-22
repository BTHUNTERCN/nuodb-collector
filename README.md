<img src="images/nuodb.svg" width="200" height="200" /> 

# NuoDB Collector

[![Build Status](https://travis-ci.com/nuodb/nuodb-collector.svg?token=nYo6yHzhBM9syBKXYk7y&branch=master)](https://travis-ci.com/nuodb/nuodb-collector)

The NuoDB Collector (NuoCD - Collector Daemon) is a replacement for [NuoCA](https://github.com/nuodb/nuoca).

# Introduction

Most modern application monitoring systems consist of the following 3
core components:

* Collector             — daemon(s) to gather metrics such as this repository
* Time Series database  — for storage of real-time, high volume metrics (e.g. InfluxDB, Prometheus, LogStash)
* Query & Visualization — that enables visual monitoring and root cause analysis (e.g. Grafana)

NuoDB Collector utilizes a popular open-source collector - `telegraf`.
It's designed to be used alongside a NuoDB engine process to collect metrics from the engine and publish those metrics to a time series database.

Built into this container are 4 input plugins to collect metrics from the NuoDB engine:

1.  `metrics` - collects the [Engine Metrics](https://doc.nuodb.com/nuodb/4.0.x/reference-information/metrics-published-by-database-processes/)  on a
    regular 10s interval.
2.  `msgtrace` - collects internal NuoDB message tracing data on a regular 30s interval.
3.  `synctrace` - collects internal NuoDB lock tracing data on a regular 30s interval.
4.  `threads` - extends the [Telegraf ProcStat Input plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/procstat) with per-thread data.
Collects host machine resource consumption statistics on a regular 10s interval.

# NuoDB Collector Page Outline
[Setup on Bare Metal](#Setup-on-bare-metal)

[Setup in Docker](#Setup-in-docker)

[Setup in Kubernetes](#Setup-in-Kubernetes)


# Setup on bare metal

## Installation

### 1) Install dependencies
```
pidof - installed - sysvinit-tools package
python2.7
pip
```

### 2) Download and install `telegraf`
These steps are for RedHat or CentOS. For other platforms, see [Telegraf Documentation](https://portal.influxdata.com/downloads/).

```
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.15.2-1.x86_64.rpm
sudo yum localinstall telegraf-1.15.2-1.x86_64.rpm
```

### 3) Download and install NuoDB Collector
```
git clone git@github.com:nuodb/nuodb-collector.git
cd nuodb-collector
pip install -r requirements.txt
sudo cp -r nuocd /opt/
```

## Configuration
The `conf/nuodb.conf` file in this repository configures all 4 input plugins for NuoDB running on localhost as described in the section above.
The `conf/outputs.conf` file configures an output plugin to a InfluxDB instance defined by the `$INFLUXURL` environmental variable.
Replace the `<hostinflux>` placeholder with the URL of a running InfluxDB instance.
```
sudo cp conf/nuodb.conf /etc/telegraf/telegraf.d
sudo cp conf/outputs.conf /etc/telegraf/telegraf.d
sudo edit /etc/telegraf/telegraf.conf
sudo chown -R telegraf.telegraf /etc/telegraf
sudo cat >> /etc/default/telegraf <<EOF
INFLUXURL=http://<hostinflux>:8086
PYTHONPATH=/opt/nuocd/pylib
EOF
```

### Start NuoDB Collector
```
sudo systemctl daemon-reload
sudo systemctl restart telegraf
```

**NOTE:** If not starting telegraf via `systemd` then the variables set in `/etc/default/telegraf` are not picked up automatically.
Instead you can start telegraf with the following command:
```
sh -c "$(cat /etc/default/telegraf | tr '\n' ' ') telegraf --config /etc/telegraf/telegraf.conf --config-directory /etc/telegraf/telegraf.d"
```

## Setup in Docker

### Download the NuoDB Collector Docker Image
```
docker pull docker.pkg.github.com/nuodb/nuodb-collector/nuocd:latest
```

### Building docker image from source
```
git clone git@github.com:nuodb/nuodb-collector.git
cd nuodb-collector
docker build . 
docker tag <SHA> <TAG>
```

### Running Prerequisites
As a prerequisite you must have a running NuoDB domain.
To start NuoDB in Docker, follow the [NuoDB Docker Blog Part I](https://nuodb.com/blog/deploy-nuodb-database-docker-containers-part-i).

To verify your domain, run `nuocmd`:
```
$ docker exec -it nuoadmin1 nuocmd show domain
server version: 4.1.1-3-2203dab8dd, server license: Community
server time: 2020-10-20T19:20:35.915, client token: 2003aa06ce0444b2152b543beff9e95312b47e84
Servers:
  [nuoadmin1] nuoadmin1:48005 [last_ack = 1.02] [member = ADDED] [raft_state = ACTIVE] (LEADER, Leader=nuoadmin1, log=0/19/19) Connected *
  [nuoadmin2] nuoadmin2:48005 [last_ack = 1.02] [member = ADDED] [raft_state = ACTIVE] (FOLLOWER, Leader=nuoadmin1, log=0/18/18) Connected
  [nuoadmin3] nuoadmin3:48005 [last_ack = 1.01] [member = ADDED] [raft_state = ACTIVE] (FOLLOWER, Leader=nuoadmin1, log=0/19/19) Connected
Databases:
  test [state = RUNNING]
    [SM] test-sm-1/172.20.0.5:48006 [start_id = 0] [server_id = nuoadmin1] [pid = 39] [node_id = 1] [last_ack =  7.14] MONITORED:RUNNING
    [TE] test-te-1/172.20.0.6:48006 [start_id = 1] [server_id = nuoadmin1] [pid = 39] [node_id = 2] [last_ack =  0.14] MONITORED:RUNNING
```

This collector also assumes that an InfluxDB instance is already running.
If you do not already have InfluxDB running, you can start a simple install like so.
```
docker run -d --name influxdb \
      --network nuodb-net \
      -p 8086:8086 \
      -p 8082:8082 \
      influxdb:latest
```

### Running NuoDB Collector
Each NuoDB Collector runs colocated with a NuoDB engine in the same process namespace.
As such, you must start a NuoDB collector docker container for every running NuoDB engine you want to monitor.

The following value replacement must be done to start a NuoDB Collector container:
- Replace the `<hostinflux>` placeholder with the URL of a running InfluxDB container. In our example, it will be `influxdb`.
- Replace the `<nuoadmin>` placeholder with the URL of a running NuoDB admin container. In our example, it will be `nuoadmin1`.
- Replace the `<enginecontainer>` placeholder with the URL of a running NuoDB Engine container. In our example, it will be `test-sm-1`.
- Replace the `<hostname>` with the hostname of the monitored engine container. The hostnames must match. In our example it will be `test-sm-1` 

```
docker run -d --name nuocd-sm \
      --hostname <hostname> \
      --network nuodb-net \
      --env INFLUXURL=http://<hostinflux>:8086 \
      --env NUOCMD_API_SERVER=https://<nuoadmin>:8888 \
      --pid container:<enginecontainer> \
      docker.pkg.github.com/nuodb/nuodb-collector/nuocd:latest
```

Repeat the steps above for all running NuoDB engine containers you want to monitor.

### Docker Example
For a complete example on how to set up the NuoDB domain with NuoDB collector, you can use `docker compose`.
The following command will start:
- 2 Admin Processes
- 1 Storage Manager
- 2 Transaction Engines
- 2 NuoDB Collector containers (1 for SM, 1 for TE)
- InfluxDB time-series database

```
DOCKER_IMAGE=nuodb/nuodb-ce:latest docker-compose up -d
```

## Setup in Kubernetes

Follow the documentation in the [NuoDB Helm Charts](https://github.com/nuodb/nuodb-helm-charts) repository.
