# Prometheus-Grafana-Telegram-Kubernetes

With this files you can deploy a Kubernetes environment that includes monitoring with [Prometheus](https://github.com/prometheus/prometheus), alerts to Telegram using [Alert Manager](https://github.com/prometheus/alertmanager) and visualize metrics with [Grafana](https://github.com/grafana/grafana).

## Description of files

For implementation, you need to change certain values in certain files, let's describe them.

### Prometheus

The first file ```00-prometheus-namespace.yaml```, like his name indicate, only creates a namespace named monitoring (You can change its name, but you need to replace new namespace name in almost all files).

For ```01-prometheus-cluster-role.yaml``` file, gives to Prometheus necessary permissions access to Kubernetes cluster metrics.

In file ```02-prometheus-configmap.yaml``` has important configurations of this implementation, such as metrics name and alerts. First configuration is   __prometheus.rules__, in this part you can add custom rules for Alert Manager, there are example rules such as _High Node Memory_, _High Memory Usage in Pod_, _High Node CPU_, _Node Failover_, _etc_., all based in kubernetes metrics. For more information about how this rules works, consult [Prometheus Alerting](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) documentation. Second configuration is __prometheus.yml__, that contains values such as scrap interval, evaluation time of rules, path location of __prometheus.rules__ and where is allocated Alert Manager (in this case, Kubernetes service). More information in [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

Using ```03-prometheus-deployment.yaml``` can deploy Prometheus application, this file uses version ```v2.29.2 or higher```. You need to change value ```<Your Time Zone>``` of environment variable __TZ__ with you respective [Time Zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) or just delete it.

```yml
env:
  - name: TZ
    value: <Your Time Zone>
```
Last file for Prometheus is ```04-prometheus-svc.yaml```, that is only a __Kubernetes NodePort__ service in port 30000, you can change this exposed port or change service type.

### Alert Manager

First file ```05-prometheus-alertmanager-configmap.yaml``` is a ConfigMap for __alertmanager.yml__ parameters, such as timers for alerts, type of alerts and receivers. In this part, the Telegram receiver is configured to receive alerts, so that you need to create Telegram bot with [BotFather](https://t.me/BotFather), it will return your bot token. After that, create a chat group in Telegram and add your bot there, and make the following GET request ```https://api.telegram.org/bot<Your Bot Token>/getUpdates``` and you will obtain your Chat ID.

Replace ```<Your Telegram Chat ID>``` with the value you got from your bot, _**with everything inside the quotes**_. (Some Chat ID's start with a ```-```, in this case, you must also include the ```-``` in the url).

The URL ```http://prometheus-bot:9087``` is a __Kubernetes ClusterIP__ service and behind it is a deployment with a Prometheus Bot that recive alerts and send to chat group in Telegram.

```yml
receivers:
- name: webhook-telegram
  webhook_configs:
    - send_resolved: false
      url: 'http://prometheus-bot:9087/alert/<Your Telegram Chat ID>'
```

For more information [Alert Manager Configuration](https://prometheus.io/docs/alerting/latest/configuration/) is your ally.

Next file, ```06-prometheus-alertmanager-deployment.yaml``` is deployment for Alert Manager, using version ```v0.23.0 or higher``` . You need to change value ```<Your Time Zone>``` of environment variable __TZ__ too or just delete it.

Last file, ```07-prometheus-alertmanager-svc.yaml``` is a __Kubernetes ClusterIP__ service, because in this case, no need to expose deployment to the world, only Prometheus can access to this service.

### Telegram Prometheus Bot

This part is based on [prometheus_bot](https://github.com/inCaller/prometheus_bot) by inCaller, with a little changes, such as the Dockerfile. 

```Dockerfile
FROM golang:1.17.1-alpine3.14 as builder
RUN apk add --no-cache git ca-certificates make tzdata
COPY . /app
RUN cd /app && \
    go get -d -v && \
    CGO_ENABLED=0 GOOS=linux go build -v -a -installsuffix cgo -o prometheus_bot

FROM alpine:3.13.6
COPY --from=builder /app/prometheus_bot /
RUN apk add --no-cache ca-certificates tzdata tini
RUN mkdir /etc/telegrambot/
USER nobody
EXPOSE 9087
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["/prometheus_bot","-c","/etc/telegrambot/prometheus-bot.yml","-d"]
```
As you can see, golang and alpine version has been updated, a directory ```/etc/telegrambot/``` has been created and flags has been added in CMD, __-c__ for path of config file (named ```prometheus-bot.yml```) and __-d__ for debug. Docker image has been uploaded in Docker Hub as ```salvadorarreola/telegram-prometheus-bot```. If you want to upload your own Docker image, clone Github repository [prometheus_bot](https://github.com/inCaller/prometheus_bot) and update Dockerfile as show before __*(you can change directory and config file names, but you need to update configmap and deployment too)*__.

In this file, ```08-prometheus-bot-configmap.yaml```, describes two config files, __prometheus-bot.yml__ and __alert-template.tmpl__. First of them, __prometheus-bot.yml__, is for Bot Configuration options, such as template path, time zone, telegram token, etc. As you can see, you need to replace ```<Your Telegram bot Token>```

```yml
telegram_token: "<Your Telegram bot Token>"
template_path: "/etc/telegrambot/alert-template.tmpl"
time_zone: "<Your Time Zone>"
split_token: "|"
split_msg_byte: 4096
```
Next config file __alert-template.tmpl__ is about what information will be send to Telegram Bot (using labels and annotations of ```prometheus.rules``` in file ```08-prometheus-bot-configmap.yaml```), this is a template but you need to pay attention in syntax, that is [go templating language](https://pkg.go.dev/text/template). By the way, we can use some HTML tags to further customize the message.

Template example and Telegram message received:

```
    {{if eq .CommonLabels.alertname "High Disk Space" -}}
    {{ range .Alerts }}
    Alertname: <b>{{ .Labels.alertname }}</b>
    Summary: <b>{{ .Annotations.summary }}</b>
    Node: <b>{{ .Labels.instance }}</b>
    Percentage Disk Usage: <b>{{ .Labels.value }} %</b>
    Severity: <b>{{ .Labels.severity }}</b>
    Status:  <b>{{ .Status }}</b>
    {{ end }}
    {{ end -}}
```
![telegram_message](https://user-images.githubusercontent.com/68822231/149646664-855dcef8-910c-4be9-af71-a2aa91618804.png)

Finally, we have __Kubernetes Deployment__ ```09-prometheus-bot-deployment.yaml``` for Telgram Bot. Like the previous ones, change value ```<Your Time Zone>``` (you can change Docker Image too if you create a new one) and a __Kubernetes ClusterIP__ service ```10-prometheus-bot-svc.yaml```for internal connection.

### Grafana

ConfigMap file ```11-prometheus-grafana-configmap.yaml``` contains connection parameters to Prometheus (```prometheus.yaml```) using its Kubernetes Service (```04-prometheus-svc.yaml```). Next file is optional, ```home.json``` is a __Grafana Dashboard__ that override default home.json in Grafana, so if Pod dies, this dashboard still be available. You can use it, create a custom or using dashboards availables in [Grafana Dashboards](https://grafana.com/grafana/dashboards/). Dashboard used: 315 [Kubernetes cluster monitoring (via Prometheus)](https://grafana.com/grafana/dashboards/315).

Next file ```12-prometheus-grafana-secret.yaml``` is __Kubernetes Secret__ that contains user and password for Grafana Deployment, replace ```<Your user in base64>``` and ```<Your password in base64>```.

Next, a __Kubernetes Deployment__ ```13-prometheus-grafana-deployment.yaml```, using ConfigMap to mount configuration files and Secret to set the user and password. Finally (I promise you it's the last time) replace value ```<Your Time Zone>```.

Last file is a __Kubernetes NodePort__ service ```14-prometheus-grafana-svc.yaml``` to expose port 3000 of Grafana in port 31000 (or any other port that you want).

### KSM

Last 5 files about [kubernetes-state-metrics](https://github.com/kubernetes/kube-state-metrics) are use to get more useful metrics about Kubernetes cluster and health state of the ojects.

### Be careful with CVE-2021-43798 Path Traversal Vulnerability in Grafana and Log4Shell

