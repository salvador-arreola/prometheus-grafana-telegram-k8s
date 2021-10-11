# Prometheus-Grafana-Telegram-Kubernetes

With this files you can deploy a Kubernetes environment that includes monitoring with [Prometheus](https://github.com/prometheus/prometheus), alerts to Telegram using [Alert Manager](https://github.com/prometheus/alertmanager) and visualize metrics with [Grafana](https://github.com/grafana/grafana).

## Description of files

For implementation, you need to change certain values in certain files, let's describe them.

### Prometheus

The first file ```00-prometheus-namespace.yaml```, like his name indicate, only creates a namespace named monitoring (You can change its name, but you need to replace new namespace name in almost all files).

For ```01-prometheus-cluster-role.yaml``` file, gives to Prometheus necessary permissions access to Kubernetes cluster metrics.

In file ```02-prometheus-configmap.yaml``` has important configurations of this implementation, such as metrics name and alerts. First configuration is   __prometheus.rules__, in this part you can add custom rules for Alert Manager, there are example rules such as _High Node Memory_, _High Memory Usage in Pod_, _High Node CPU_, _Node Failover_, _etc_., all based in kubernetes metrics. For more information about how this rules works, consult [Prometheus Alerting](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) documentation. Second configuration is __prometheus.yml__, that contains values such as scrap interval, evaluation time of rules, path location of __prometheus.rules__ and where is allocated Alert Manager (in this case, Kubernetes service). More information in [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

Using ```03-prometheus-deployment.yaml``` can deploy Prometheus application, this file uses version ```v2.29.2```. You need to change value of environment variable __TZ__ with you respective [Time Zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) or just delete it.

```yml
env:
  - name: TZ
    value: <Your Time Zone>
```
Last file for Prometheus is ```04-prometheus-svc.yaml```, that is only a __Kubernetes NodePort__ service in port 30000, you can change this exposed port or change service type.

### Alert Manager

First file ```05-prometheus-alertmanager-configmap.yaml``` is a ConfigMap for __alertmanager.yml__ parameters, such as timers for alerts, type of alerts and receivers. In this part, the Telegram receiver is configured to receive alerts. You need to replace:

```yml
receivers:
- name: webhook-telegram
  webhook_configs:
    - send_resolved: false
      url: 'http://prometheus-bot:9087/alert/<Your Telegram Chat ID>'
```

For more information [Alert Manager Configuration](https://prometheus.io/docs/alerting/latest/configuration/) is your ally.

