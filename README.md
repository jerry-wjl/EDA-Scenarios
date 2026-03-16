# Introduction:
The repository contains rulebooks and playbooks for most typical Ansible Automation Platform EDA using scenarios. 

# Scenarios includes:
|Scenarios                                                 | rulebook                               | playbook                         |
| :----------                                              | :----------                            | :---                             |
| High CPU Usage detect, Email alert admin                 | send_cpu_warning_rule-email.yml        | cpu_execeed_quota_warning.yml    |
| High Memory Usage detect, Email alert admin              | send_memory_warning_rule-email.yml     | memory_execeed_quota_warning.yml |
| High Disk IO Usage detect, Email alert admin             | send_io_warning_rule-email.yml         | high_io_usage_detect_warning.yml |
| High Network Bandwith Usage detect, Email alert admin    | send_network_warning_rule-email.yml    | network_usage_warning.yml        |
| Filesystem exceed quota detect, Email alert admin        | send_fs_warning_rule-email.yml         | fs_exceed_quota_warning.yml      |
