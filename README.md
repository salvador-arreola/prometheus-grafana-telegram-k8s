# Prometheus-Grafana-Telegram-Kubernetes

With this files you can deploy a Kubernetes environment that includes monitoring with [Prometheus](https://github.com/prometheus/prometheus), alerts to Telegram using [Alert Manager](https://github.com/prometheus/alertmanager) and visualize metrics with [Grafana](https://github.com/grafana/grafana).

## Description of files

The first file ```00-prometheus-namespace.yaml```, like his name indicate, only creates a namespace named monitoring (You can change its name, but you need to replace new namespace name in almost all files).

For ```01-prometheus-cluster-role.yaml``` file, gives to Prometheus necessary permissions access to Kubernetes cluster metrics.
