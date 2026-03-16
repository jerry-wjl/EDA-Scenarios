EDA-Scenarios: Event-Driven Ansible RemediationThis repository contains Rulebooks and Playbooks developed for Red Hat Ansible Automation Platform (AAP) 2.5+. It demonstrates a full closed-loop automation from monitoring alerts to automated investigation and remediation.📋 Requirements1. AAP 2.5+ ConfigurationRulebooks: Run as Rulebook Activations within AAP.Playbooks: Configured as Job Templates.Event Stream: You must create and enable an Event Stream on AAP first, then attach each Rulebook Activation to the Event Stream.2. DependenciesEnsure the community.general collection is available. You can either:Manual Installation: Run the following in your project directory:Bashansible-galaxy collection install community.general -p collections
Execution Environment (EE): Build a custom EE that includes the community.general collection.3. Managed Node PrerequisitesEnsure the following diagnostic tools are installed on each managed Linux server:Bashyum install sysstat iotop perf node_exporter -y
🏗️ ArchitectureServerRoleFQDNServer 1Ansible Automation Platform 2.5aap25-jw.example.comServer 2Monitoring (Prometheus/Alertmanager)monitoring.example.comServer 3Managed Node 1 (RHEL)rhel7.9-server01.example.comServer 4Managed Node 2 (RHEL)rhel7.9-server02.example.com🛠️ Setup Prometheus & Alertmanager1. Prometheus ConfigurationEdit /etc/prometheus/prometheus.yml to define scrape targets and alerting rules:YAMLglobal:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - monitoring.example.com:9093

rule_files:
  - "linux_normal_alert.yml"

scrape_configs:
  - job_name: "managed_nodes"
    static_configs:
      - targets:
          - "rhel7.9-server01.example.com:9100"
          - "rhel7.9-server02.example.com:9100"
2. Alertmanager ConfigurationEdit /etc/prometheus/alertmanager.yml to route alerts to the AAP Event Stream:YAMLroute:
  group_by: ['alertname', 'instance', 'namespace']
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 5m

receivers:
- name: webhook-all
  webhook_configs:
  - url: "<YOUR_EVENT_STREAM_URL>"
    send_resolved: false
    http_config:
      basic_auth:
        username: "<AUTH_USERNAME>"
        password: "<AUTH_PASSWORD>"
      tls_config:
        insecure_skip_verify: true
3. Service ManagementOn Monitoring Server:Bashsystemctl enable prometheus --now
systemctl enable alertmanager --now
On Managed Nodes:Bashsystemctl enable node_exporter --now
🚀 How it WorksDetection: Prometheus monitors managed nodes via node_exporter.Alerting: When a threshold is hit, Prometheus triggers an alert to Alertmanager.Event Ingestion: Alertmanager sends a webhook to the AAP Event Stream.Action: The EDA Rulebook Activation matches the alert and launches a Job Template (Playbook) to investigate the issue (CPU, Memory, IO, or Network).Reporting: The Playbook captures the real-time status and sends an email report to the administrator.
