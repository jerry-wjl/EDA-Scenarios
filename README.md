# Introduction:
The repository contains rulebooks and playbooks for most typical EDA on Ansible Automation Platform(AAP) using scenarios. For thoese scenarios, we use Prometheus & Alertmanager as monitoring platform, working with AAP EDA to automatically tracking event and trigger job template for some actions, such as sending alert to admin, auto remediation., etc. In each scenarios contains a rulebook(set as "rulebook activation" on AAP) which mapping a playbook(set as "job template" on AAP).


<br>

# Scenarios includes:
|Scenarios                                                 | rulebook                               | playbook                         |
| :----------                                              | :----------                            | :---                             |
| High CPU Usage detect, Email alert admin                 | send_cpu_warning_rule-email.yml        | cpu_execeed_quota_warning.yml    |
| High Memory Usage detect, Email alert admin              | send_memory_warning_rule-email.yml     | memory_execeed_quota_warning.yml |
| High Disk IO Usage detect, Email alert admin             | send_io_warning_rule-email.yml         | high_io_usage_detect_warning.yml |
| High Network Bandwith Usage detect, Email alert admin    | send_network_warning_rule-email.yml    | network_usage_warning.yml        |
| Filesystem exceed quota detect, Email alert admin        | send_fs_warning_rule-email.yml         | fs_exceed_quota_warning.yml      |

<br>

# The Architecture(Take for reference):
|Role                                                      | HostName                               |
| :----------                                              | :----------                            |
| AAP 2.5 Controller                                       | aap25.example.com                      |
| Monitor server with promethues/alertmanager              | monitoring.example.com                 | 
| Managed node1                                            | rhel7.9-server01.example.com           |
| Managed node2                                            | rhel7.9-server02.example.com           |

<br>

# Monitoring platform (Prometheus + Alertmanager) install and setup:

1. Install prometheus, alertmanager on monitor server.
2. edit prometheus and alertmanager config file as example below:

<br>

## Below is prometheus.conf example:
```
[root@monitor ~]# cat /etc/prometheus/prometheus.yml 
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - monitoring.example.com:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
    - "linux_normal_alert.yml"


# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  # - job_name: "prometheus"
  - job_name: "managed_nodes"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets:
          - "rhel7.9-server01.example.com:9090"
          - "rhel7.9-server02.example.com:9090"

```

<br>

## Below is alertmanager.conf example:
```
[root@monitor ~]# cat /etc/prometheus/alertmanager.yml
route:
  group_by: ['alertname','instance','namespace'] 
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 5m
  receiver: 'webhook-all' 

receivers:
- name: webhook-all
  webhook_configs:
  - url: "<Your Event Stream URL>"
    send_resolved: false
    max_alerts: 0
    http_config:
      basic_auth:
        username: "<Your basic Event Stream Auth username>"
        password: "<Your basic Event Stream Auth password>"

      tls_config:
        insecure_skip_verify: true

```

<br>

## Below is rules example:

```
[root@monitor ~]# cat /etc/prometheus/linux_normal_alert.yml 
groups:
- name: linux_formal_alerts
  rules:
  - alert: HostOutOfMemory
    expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Memory is not sufficient, instance:{{ $labels.instance }}"
      description: "Available memory <10%，current value：{{ $value }}"
  - alert: HostMemoryUnderMemoryPressure
    expr: rate(node_vmstat_pgmajfault[1m]) > 1000
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Memory is in presure, instance:{{ $labels.instance }}"
      description: "Memory is under presure, Page fault number is high, current value：{{ $value }}"
  - alert: HostUnusualNetworkThroughputIn
    expr: sum by (instance) (rate(node_network_receive_bytes_total[2m])) / 1024 / 1024 > 100
    # for: 5m
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Abnormal incoming network traffic, instance:{{ $labels.instance }}"
      description: "Network traffic In  > 100 MB/s, current value：{{ $value }}"
  - alert: HostUnusualNetworkThroughputOut
    expr: sum by (instance) (rate(node_network_transmit_bytes_total[2m])) / 1024 / 1024 > 100
    # for: 5m
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Abnormal outcoming network traffi，instance:{{ $labels.instance }}"
      description: "Network traffic Out > 100 MB/s，current value：{{ $value }}"
  - alert: HostUnusualDiskReadRate
    expr: sum by (instance) (rate(node_disk_read_bytes_total[2m])) / 1024 / 1024 > 50
    # for: 5m
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Abnormal Disk IO In, instance:{{ $labels.instance }}"
      description: "Disk Read> 50 MB/s，current value：{{ $value }}"
  - alert: HostUnusualDiskWriteRate
    expr: sum by (instance) (rate(node_disk_written_bytes_total[2m])) / 1024 / 1024 > 50
    # for: 2m
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Abnormal Disk IO Out, instance:{{ $labels.instance }}"
      description: "Disk Write> 50 MB/s，current value：{{ $value }}"
  - alert: HostOutOfDiskSpace
    expr: (node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10 and ON (instance, device, mountpoint) node_filesystem_readonly == 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Insufficient disk space warning, instance:{{ $labels.instance }}"
      description: "Available filesystem space < 10% ，current value：{{ $value }}"
  - alert: HostDiskWillFillIn24Hours
    expr: (node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10 and ON (instance, device, mountpoint) predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs"}[1h], 24 * 3600) < 0 and ON (instance, device, mountpoint) node_filesystem_readonly == 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Disk space is exaust in 24 hours, instance:{{ $labels.instance }}"
      description: "According current write rate, disk space will be exaust in 24 hours，current value：{{ $value }}"
  - alert: HostOutOfInodes
    expr: node_filesystem_files_free{mountpoint ="/"} / node_filesystem_files{mountpoint="/"} * 100 < 10 and ON (instance, device, mountpoint) node_filesystem_readonly{mountpoint="/"} == 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "File system Inodes insufficient, instance :{{ $labels.instance }}"
      description: "Available inodes < 10%，current value： {{ $value }}"
  - alert: HostUnusualDiskReadLatency
    expr: rate(node_disk_read_time_seconds_total[1m]) / rate(node_disk_reads_completed_total[1m]) > 0.1 and rate(node_disk_reads_completed_total[1m]) > 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Abnormal disk read latency detected, instance:{{ $labels.instance }}"
      description: "disk read latency > 100ms，current value：{{ $value }}"
  - alert: HostUnusualDiskWriteLatency
    expr: rate(node_disk_write_time_seconds_total[1m]) / rate(node_disk_writes_completed_total[1m]) > 0.1 and rate(node_disk_writes_completed_total[1m]) > 0
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Abnormal disk write latency detected, instance:{{ $labels.instance }}"
      description: "disk write latency > 100ms，current value：{{ $value }}"
  - alert: high_load 
    expr: node_load1 > 4
    for: 2m
    labels:
      severity: page
    annotations:
      summary: "High workload in 1 minitue, instance:{{ $labels.instance }}"
      description: "CPU 1 min load >4 for 2 minitues。current value：{{ $value }}"
  - alert: HostCpuIsUnderUtilized
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "cpu work load is high, instance:{{ $labels.instance }}"
      description: "cpu workload > 80%，current value：{{ $value }}"
  - alert: HostCpuStealNoisyNeighbor
    expr: avg by(instance) (rate(node_cpu_seconds_total{mode="steal"}[5m])) * 100 > 10
    for: 0m
    labels:
      severity: warning
    annotations:
      summary: "Swap usage is high, instance:{{ $labels.instance }}"
      description: "Swap usage>50%"
  - alert: HostNetworkReceiveErrors
    expr: rate(node_network_receive_errs_total[2m]) / rate(node_network_receive_packets_total[2m]) > 0.01
    for: 2m
    labels:
      severity: warning
```
<br>    
