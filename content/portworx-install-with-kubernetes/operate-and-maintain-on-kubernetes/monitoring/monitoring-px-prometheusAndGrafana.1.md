---
title: Prometheus and Grafana
keywords: portworx, container, Kubernetes, storage, Docker, k8s, pv, persistent disk, monitoring, prometheus, alertmanager, servicemonitor, grafana
description: How to use Prometheus and Grafana for monitoring Portworx on Kubernetes
---

## About Prometheus
Prometheus is an opensource monitoring and alerting toolkit. Prometheus consists of several components some of which are listed below.
- The Prometheus server which scrapes(collects) and stores time series data based on a pull mechanism.
- A rules engine which allows generation of Alerts based on the scraped metrices.
- An alertmanager for handling alerts.
- Multiple integrations for graphing and dashboarding.

In this document we would explore the monitoring of Portworx via Prometheus. The integration is natively supported by Portworx since portworx stands up metrics on a REST endpoint which can readily be scraped by Prometheus.

The following instructions allows you to monitor Portworx via Prometheus and allow the Alertmanager to provide alerts based on configured rules.

The Prometheus [Operator](https://coreos.com/operators/prometheus/docs/latest/user-guides/getting-started.html) creates, configures and manages a prometheus cluster.

The prometheus operator manages 3 customer resource definitions namely:
- Prometheus: The Prometheus CRD defines a Prometheus setup to be run on a Kubernetes cluster. The Operator creates a Statefulset for each definition of the Prometheus resource.

- ServiceMonitor: The ServiceMonitor CRD allows the definition of how Kubernetes services could be monitored based on label selectors. The Service abstraction allows Prometheus to in turn monitor underlying Pods.

- Alertmanager: The Alertmanager CRD allows the definition of an Alertmanager instance within the Kubernetes cluster. The alertmanager expects a valid configuration in the form of a `secret` called `alertmanager-name`.

## About Grafana
Grafana is a dashboarding and visualization tool with integrations to several timeseries datasources. It is used to create dashboards for the monitoring data with customizable visualizations. We would use Prometheus as the source of data to view Portworx monitoring metrics.

## Prerequisites
- A running Portworx cluster.

## Installation

### Install the Prometheus Operator
Create a file named `prometheus-operator.yaml` with the below contents and apply the spec.

```text
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-operator
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-operator
subjects:
  - kind: ServiceAccount
    name: prometheus-operator
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-operator
  namespace: kube-system
rules:
  - apiGroups:
      - extensions
    resources:
      - thirdpartyresources
    verbs: ["*"]
  - apiGroups:
      - apiextensions.k8s.io
    resources:
      - customresourcedefinitions
    verbs: ["*"]
  - apiGroups:
      - monitoring.coreos.com
    resources:
      - alertmanagers
      - prometheuses
      - prometheuses/finalizers
      - servicemonitors
      - prometheusrules
    verbs: ["*"]
  - apiGroups:
      - apps
    resources:
      - statefulsets
    verbs: ["*"]
  - apiGroups: [""]
    resources:
      - configmaps
      - secrets
    verbs: ["*"]
  - apiGroups: [""]
    resources:
      - pods
    verbs: ["list", "delete"]
  - apiGroups: [""]
    resources:
      - services
      - endpoints
    verbs: ["get", "create", "update"]
  - apiGroups: [""]
    resources:
      - nodes
    verbs: ["list", "watch"]
  - apiGroups: [""]
    resources:
      - namespaces
    verbs: ["list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-operator
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    k8s-app: prometheus-operator
  name: prometheus-operator
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: prometheus-operator
    spec:
      containers:
        - args:
            - --kubelet-service=kube-system/kubelet
            - --config-reloader-image=quay.io/coreos/configmap-reload:v0.0.1
          image: quay.io/coreos/prometheus-operator:v0.29.0
          name: prometheus-operator
          ports:
            - containerPort: 8080
              name: http
          resources:
            limits:
              cpu: 200m
              memory: 100Mi
            requests:
              cpu: 100m
              memory: 50Mi
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-operator
```

`kubectl apply -f <prometheus-operator.yaml>`

### Install the Service Monitor

Create a file named `service-monitor.yaml` with the below contents and apply the spec.
```text
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  namespace: kube-system
  name: portworx-prometheus-sm
  labels:
    name: portworx-prometheus-sm
spec:
  selector:
    matchLabels:
      name: portworx
  namespaceSelector:
    any: true
  endpoints:
  - port: px-api
    targetPort: 9001
```

`kubectl apply -f <service-monitor.yaml>`

### Install the Alertmanager
Create a file named `alertmanager.yaml` with the following contents and create a secret from it.
Make sure you add the relevant email addresses in the below config.
```text
global:
  # The smarthost and SMTP sender used for mail notifications.
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: '<sender-email-address>'
  smtp_auth_username: "<sender-email-address>"
  smtp_auth_password: '<sender-email-password>'
route:
  group_by: [Alertname]
  # Send all notifications to me.
  receiver: email-me
receivers:
- name: email-me
  email_configs:
  - to: <receiver-email-address>
    from: <sender-email-address>
    smarthost: smtp.gmail.com:587
    auth_username: "<sender-email-address>"
    auth_identity: "<sender-email-address>"
    auth_password: "<sender-email-password>"
## Edit the file and create a secret with it using the following command
```
`kubectl create secret generic alertmanager-portworx --from-file=alertmanager.yaml -n kube-system`


Create a file named `alertmanager-cluster.yaml` with the below contents and apply the spec on your cluster.
```text
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  name: portworx #This name is important since the Alertmanager pods wont start unless a secret named alertmanager-${ALERTMANAGER_NAME} is created. in this case if would expect alertmanager-portworx secret in the kube-system namespace
  namespace: kube-system
  labels:
    alertmanager: portworx
spec:
  replicas: 3
```

`kubectl apply -f alertmanager-cluster.yaml`


Create a file named `alertmanager-service.yaml` with the following contents and apply the spec.

```text
apiVersion: v1
kind: Service
metadata:
  name: alertmanager-portworx
  namespace: kube-system
spec:
  type: NodePort
  ports:
  - name: web
    port: 9093
    protocol: TCP
    targetPort: web
  selector:
    alertmanager: portworx
```

`kubectl apply -f alertmanager-service.yaml`

### Install Prometheus

Create a file named `prometheus-rules.yaml` with the following contents and apply the spec.

```text
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: portworx
    role: prometheus-portworx-rulefiles
  name: prometheus-portworx-rules-portworx.rules.yaml
  namespace: kube-system
spec:
  groups:
  - name: portworx.rules
    rules:
    - alert: PortworxVolumeUsageCritical
      annotations:
        description: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is over 80% used for
          more than 10 minutes.
        summary: Portworx volume capacity is at {{$value}}% used.
      expr: 100 * (px_volume_usage_bytes / px_volume_capacity_bytes) > 80
      for: 5m
      labels:
        issue: Portworx volume {{$labels.volumeid}} usage on {{$labels.host}} is high.
        severity: critical
    - alert: PortworxVolumeUsage
      annotations:
        description: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is over 70% used for
          more than 10 minutes.
        summary: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is at {{$value}}% used.
      expr: 100 * (px_volume_usage_bytes / px_volume_capacity_bytes) > 70
      for: 5m
      labels:
        issue: Portworx volume {{$labels.volumeid}} usage on {{$labels.host}} is critical.
        severity: warning
    - alert: PortworxVolumeWillFill
      annotations:
        description: Disk volume {{$labels.volumeid}} on {{$labels.host}} is over 70% full and has
          been predicted to fill within 2 weeks for more than 10 minutes.
        summary: Portworx volume {{$labels.volumeid}} on {{$labels.host}} is over 70% full and is
          predicted to fill within 2 weeks.
      expr: (px_volume_usage_bytes / px_volume_capacity_bytes) > 0.7 and predict_linear(px_cluster_disk_available_bytes[1h],
        14 * 86400) < 0
      for: 10m
      labels:
        issue: Disk volume {{$labels.volumeid}} on {{$labels.host}} is predicted to fill within
          2 weeks.
        severity: warning
    - alert: PortworxStorageUsageCritical
      annotations:
        description: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 80% used
          for more than 10 minutes.
        summary: Portworx storage capacity is at {{$value}}% used.
      expr: 100 * (1 - px_cluster_disk_utilized_bytes / px_cluster_disk_total_bytes)
        < 20
      for: 5m
      labels:
        issue: Portworx storage {{$labels.volumeid}} usage on {{$labels.host}} is high.
        severity: critical
    - alert: PortworxStorageUsage
      annotations:
        description: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 70% used
          for more than 10 minutes.
        summary: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is at {{$value}}% used.
      expr: 100 * (1 - (px_cluster_disk_utilized_bytes / px_cluster_disk_total_bytes))
        < 30
      for: 5m
      labels:
        issue: Portworx storage {{$labels.volumeid}} usage on {{$labels.host}} is critical.
        severity: warning
    - alert: PortworxStorageWillFill
      annotations:
        description: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 70% full
          and has been predicted to fill within 2 weeks for more than 10 minutes.
        summary: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is over 70% full and
          is predicted to fill within 2 weeks.
      expr: (100 * (1 - (px_cluster_disk_utilized_bytes / px_cluster_disk_total_bytes)))
        < 30 and predict_linear(px_cluster_disk_available_bytes[1h], 14 * 86400) <
        0
      for: 10m
      labels:
        issue: Portworx storage {{$labels.volumeid}} on {{$labels.host}} is predicted to fill within
          2 weeks.
        severity: warning
    - alert: PortworxStorageNodeDown
      annotations:
        description: Portworx Storage Node has been offline for more than 5 minutes.
        summary: Portworx Storage Node is Offline.
      expr: max(px_cluster_status_nodes_storage_down) > 0
      for: 5m
      labels:
        issue: Portworx Storage Node is Offline.
        severity: critical
    - alert: PortworxQuorumUnhealthy
      annotations:
        description: Portworx cluster Quorum Unhealthy for more than 5 minutes.
        summary: Portworx Quorum Unhealthy.
      expr: max(px_cluster_status_cluster_quorum) > 1
      for: 5m
      labels:
        issue: Portworx Quorum Unhealthy.
        severity: critical
    - alert: PortworxMemberDown
      annotations:
        description: Portworx cluster member(s) has(have) been down for more than
          5 minutes.
        summary: Portworx cluster member(s) is(are) down.
      expr: (max(px_cluster_status_cluster_size) - count(px_cluster_status_cluster_size))
        > 0
      for: 5m
      labels:
        issue: Portworx cluster member(s) is(are) down.
        severity: critical
```
`kubectl apply -f prometheus-rules.yaml`

Create a file named `prometheus-cluster.yaml` with the following contents and apply the spec.

```text
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - services
      - endpoints
      - pods
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
      - configmaps
    verbs: ["get"]
  - nonResourceURLs: ["/metrics", "/federate"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: kube-system
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: kube-system
spec:
  replicas: 2
  logLevel: debug
  serviceAccountName: prometheus
  alerting:
    alertmanagers:
      - namespace: kube-system
        name: alertmanager-portworx
        port: web
  serviceMonitorSelector:
    matchLabels:
      name: portworx-prometheus-sm
    namespaceSelector:
      matchNames:
        - kube-system
    resources:
      requests:
        memory: 400Mi
  ruleSelector:
    matchLabels:
      role: prometheus-portworx-rulefiles
      prometheus: portworx
    namespaceSelector:
      matchNames:
        - kube-system
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - name: web
      port: 9090
      protocol: TCP
      targetPort: 9090
  selector:
    prometheus: prometheus
```
`kubectl apply -f prometheus-cluster.yaml`

### Post Install verification


#### Prometheus access details

Prometheus can be be accessed using a NodePort service.

First get the node port that prometheus is using

  ```text
  kubectl get svc -n kube-system prometheus
  ```

Navigate to the Prometheus web UI by going to `http://<master_ip>:<service_nodeport>`. You should be able to navigate to the `Targets` and `Rules` section of the Prometheus dashboard which lists the Portworx cluster endpoints as well as the Alerting rules as specified earlier.

### Installing Grafana

Create a file named `grafana-deployment.yaml` with the below contents and apply the spec.

```text
kind: ConfigMap
apiVersion: v1
metadata:
  name: grafana-dashboards
  namespace: kube-system
  labels:
    role: grafana-dashboardfiles
    grafana: portworx
data:
  portworx-cluster-dashboard.json: |-
    {
      "__inputs": [
        {
          "name": "DS_PROMETHEUS",
          "label": "prometheus",
          "description": "",
          "type": "datasource",
          "pluginId": "prometheus",
          "pluginName": "Prometheus"
        }
      ],
      "__requires": [
        {
          "type": "grafana",
          "id": "grafana",
          "name": "Grafana",
          "version": "5.0.0-beta1"
        },
        {
          "type": "panel",
          "id": "heatmap",
          "name": "Heatmap",
          "version": ""
        },
        {
          "type": "datasource",
          "id": "prometheus",
          "name": "Prometheus",
          "version": "1.0.0"
        },
        {
          "type": "panel",
          "id": "singlestat",
          "name": "Singlestat",
          "version": ""
        }
      ],
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "iteration": 1541099197839,
      "links": [],
      "panels": [
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 54,
          "panels": [],
          "repeat": null,
          "title": "Portworx Cluster \"[[Cluster]]\"",
          "type": "row"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": true,
          "colors": [
            "#fce2de",
            "#eab839",
            "#bf1b00"
          ],
          "datasource": "prometheus",
          "format": "percent",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": true,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 4,
            "w": 4,
            "x": 0,
            "y": 1
          },
          "id": 56,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "sum(px_cluster_disk_utilized_bytes{cluster=~\"[[Cluster]]\"})/sum(px_cluster_disk_total_bytes{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "80,90",
          "title": "Usage Meter",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "total"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "bytes",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 4,
            "w": 4,
            "x": 4,
            "y": 1
          },
          "id": 63,
          "interval": null,
          "links": [
            {
              "dashUri": "db/portworx-volume-dashboard",
              "dashboard": "Portworx Volume Dashboard",
              "includeVars": true,
              "keepTime": true,
              "targetBlank": true,
              "title": "Portworx Volume Dashboard",
              "type": "dashboard"
            }
          ],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "sum(px_cluster_disk_utilized_bytes{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "Capacity Used",
          "transparent": false,
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "percent",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 4,
            "w": 4,
            "x": 8,
            "y": 1
          },
          "id": 61,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "avg(px_cluster_cpu_percent{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "Avg. Cluster CPU",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 2,
            "w": 4,
            "x": 12,
            "y": 1
          },
          "id": 62,
          "interval": null,
          "links": [
            {
              "dashUri": "db/portworx-node-dashboard",
              "dashboard": "Portworx Node Dashboard",
              "includeVars": true,
              "keepTime": true,
              "targetBlank": true,
              "title": "Portworx Node Dashboard",
              "type": "dashboard"
            }
          ],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_cluster_size{cluster=~\"[[Cluster]]\"}) ",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "# Nodes (total)",
          "transparent": false,
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": true,
          "colorValue": false,
          "colors": [
            "#299c46",
            "#629e51",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "short",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 4,
            "w": 4,
            "x": 16,
            "y": 1
          },
          "hideTimeOverride": true,
          "id": 81,
          "interval": "15s",
          "links": [],
          "mappingType": 2,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "2",
              "text": "Quorum Unhealthy",
              "to": "1000"
            },
            {
              "from": "0",
              "text": "Quorum Healthy",
              "to": "1.99"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_cluster_size{cluster=~\"[[Cluster]]\"})/sum(px_cluster_status_cluster_quorum{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "interval": "",
              "intervalFactor": 2,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "1.1,1.9",
          "timeFrom": "1m",
          "title": "Quorum",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "2"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 2,
            "w": 4,
            "x": 20,
            "y": 1
          },
          "id": 85,
          "interval": null,
          "links": [
            {
              "dashUri": "db/portworx-node-dashboard",
              "dashboard": "Portworx Node Dashboard",
              "includeVars": true,
              "keepTime": true,
              "targetBlank": true,
              "title": "Portworx Node Dashboard",
              "type": "dashboard"
            }
          ],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_nodes_online{cluster=~\"[[Cluster]]\"}) ",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "# Nodes online",
          "transparent": false,
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 2,
            "w": 4,
            "x": 12,
            "y": 3
          },
          "id": 83,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_storage_nodes_online{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "intervalFactor": 1,
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "Storage Providers",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": true,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "short",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 2,
            "w": 4,
            "x": 20,
            "y": 3
          },
          "id": 84,
          "interval": null,
          "links": [],
          "mappingType": 2,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "0",
              "text": "All members online",
              "to": "0"
            },
            {
              "from": "1",
              "text": "Members offline",
              "to": "1"
            },
            {
              "from": "2",
              "text": "Members offline",
              "to": "1000"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_cluster_size{cluster=\"[[Cluster]]\"})-min(px_cluster_status_storage_nodes_online{cluster=\"[[Cluster]]\"})",
              "format": "time_series",
              "instant": false,
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "0.5,1",
          "title": "",
          "type": "singlestat",
          "valueFontSize": "50%",
          "valueMaps": [
            {
              "op": "=",
              "text": "",
              "value": "0"
            },
            {
              "op": "=",
              "text": "",
              "value": ""
            }
          ],
          "valueName": "current"
        },
        {
          "cards": {
            "cardPadding": null,
            "cardRound": null
          },
          "color": {
            "cardColor": "#b4ff00",
            "colorScale": "sqrt",
            "colorScheme": "interpolateOranges",
            "exponent": 0.5,
            "mode": "spectrum"
          },
          "dataFormat": "timeseries",
          "datasource": "prometheus",
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 5
          },
          "heatmap": {},
          "highlightCards": true,
          "id": 78,
          "legend": {
            "show": true
          },
          "links": [
            {
              "dashUri": "db/portworx-node-dashboard",
              "dashboard": "Portworx Node Dashboard",
              "includeVars": true,
              "keepTime": true,
              "targetBlank": true,
              "title": "Portworx Node Dashboard",
              "type": "dashboard"
            }
          ],
          "targets": [
            {
              "expr": "px_cluster_cpu_percent{cluster =~ \"[[Cluster]]\"}",
              "format": "time_series",
              "interval": "5m",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "title": "CPU utilization heat map",
          "tooltip": {
            "show": true,
            "showHistogram": true
          },
          "transparent": false,
          "type": "heatmap",
          "xAxis": {
            "show": true
          },
          "xBucketNumber": null,
          "xBucketSize": null,
          "yAxis": {
            "decimals": null,
            "format": "percent",
            "logBase": 1,
            "max": null,
            "min": null,
            "show": true,
            "splitFactor": null
          },
          "yBucketNumber": null,
          "yBucketSize": null
        },
        {
          "cards": {
            "cardPadding": null,
            "cardRound": null
          },
          "color": {
            "cardColor": "#b4ff00",
            "colorScale": "sqrt",
            "colorScheme": "interpolateOranges",
            "exponent": 0.5,
            "mode": "spectrum"
          },
          "dataFormat": "timeseries",
          "datasource": "prometheus",
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 5
          },
          "heatmap": {},
          "highlightCards": true,
          "id": 79,
          "legend": {
            "show": true
          },
          "links": [
            {
              "dashUri": "db/portworx-node-dashboard",
              "dashboard": "Portworx Node Dashboard",
              "includeVars": true,
              "keepTime": true,
              "targetBlank": true,
              "title": "Node Drilldown",
              "type": "dashboard"
            }
          ],
          "repeat": null,
          "repeatDirection": "h",
          "targets": [
            {
              "expr": "px_cluster_memory_utilized_percent{cluster =~ \"[[Cluster]]\"}",
              "format": "time_series",
              "interval": "5m",
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}",
              "refId": "A"
            }
          ],
          "title": "Memory utilization heat map",
          "tooltip": {
            "show": true,
            "showHistogram": true
          },
          "type": "heatmap",
          "xAxis": {
            "show": true
          },
          "xBucketNumber": null,
          "xBucketSize": null,
          "yAxis": {
            "decimals": null,
            "format": "percent",
            "logBase": 1,
            "max": null,
            "min": null,
            "show": true,
            "splitFactor": null
          },
          "yBucketNumber": null,
          "yBucketSize": null
        }
      ],
      "refresh": "30s",
      "schemaVersion": 16,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": [
          {
            "allValue": null,
            "current": {},
            "datasource": "prometheus",
            "hide": 1,
            "includeAll": false,
            "label": "",
            "multi": false,
            "name": "Cluster",
            "options": [],
            "query": "px_cluster_cpu_percent",
            "refresh": 1,
            "regex": "/.*cluster=\"([^\"]*).*/",
            "sort": 0,
            "tagValuesQuery": "",
            "tags": [],
            "tagsQuery": "",
            "type": "query",
            "useTags": false
          }
        ]
      },
      "time": {
        "from": "now-3h",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": [
          "5s",
          "10s",
          "30s",
          "1m",
          "5m",
          "15m",
          "30m",
          "1h",
          "2h",
          "1d"
        ],
        "time_options": [
          "5m",
          "15m",
          "1h",
          "6h",
          "12h",
          "24h",
          "2d",
          "7d",
          "30d"
        ]
      },
      "timezone": "",
      "title": "Portworx Cluster Dashboard",
      "uid": "xLgt8oTik",
      "version": 6
    }
  portworx-node-dashboard.json: |-
    {
      "__inputs": [
        {
          "name": "DS_PROMETHEUS",
          "label": "prometheus",
          "description": "",
          "type": "datasource",
          "pluginId": "prometheus",
          "pluginName": "Prometheus"
        }
      ],
      "__requires": [
        {
          "type": "grafana",
          "id": "grafana",
          "name": "Grafana",
          "version": "5.0.0-beta1"
        },
        {
          "type": "panel",
          "id": "graph",
          "name": "Graph",
          "version": ""
        },
        {
          "type": "datasource",
          "id": "prometheus",
          "name": "Prometheus",
          "version": "1.0.0"
        },
        {
          "type": "panel",
          "id": "singlestat",
          "name": "Singlestat",
          "version": ""
        }
      ],
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "iteration": 1541099248953,
      "links": [],
      "panels": [
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 72,
          "panels": [],
          "title": "",
          "type": "row"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 4,
            "w": 5,
            "x": 0,
            "y": 1
          },
          "id": 62,
          "interval": null,
          "links": [
            {
              "dashUri": "db/portworx-cluster-dashboard",
              "dashboard": "Portworx Cluster Dashboard",
              "includeVars": true,
              "keepTime": true,
              "title": "Porworx Cluster Dashboard",
              "type": "dashboard"
            }
          ],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": true,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_cluster_size{cluster=~\"[[Cluster]]\"}) ",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "Members",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 4,
            "w": 5,
            "x": 5,
            "y": 1
          },
          "id": 83,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "min(px_cluster_status_storage_nodes_online{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "intervalFactor": 1,
              "refId": "A"
            }
          ],
          "thresholds": "",
          "title": "Storage Providers",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 4,
            "w": 14,
            "x": 10,
            "y": 1
          },
          "id": 53,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": true,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null as zero",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Cluster Disk Total",
              "dsType": "influxdb",
              "expr": "px_cluster_disk_total_bytes{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "host"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_disk_total_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "sum"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Cluster Disk Used",
              "dsType": "influxdb",
              "expr": "px_cluster_disk_utilized_bytes{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "host"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_disk_utilized_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "sum"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "PWX Disk Usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 5
          },
          "id": 65,
          "panels": [],
          "repeat": null,
          "title": "Instances \"[[Instances]]\"",
          "type": "row"
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 0,
            "y": 6
          },
          "id": 1,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": true,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "CPU Usage: $tag_cluster - $tag_node",
              "dsType": "influxdb",
              "expr": "px_cluster_cpu_percent{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "node"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "cluster"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_cpu_percent",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "mean"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Node CPU Usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": "100",
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 8,
            "y": 6
          },
          "id": 2,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": true,
            "rightSide": false,
            "show": true,
            "sort": "avg",
            "sortDesc": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Memory Usage: $tag_node",
              "dsType": "influxdb",
              "expr": "px_cluster_memory_utilized_percent{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "1m"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "node"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "instant": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_memory_utilized_percent",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  },
                  {
                    "params": [
                      "Memory Usage"
                    ],
                    "type": "alias"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "PWX Memory Usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": "100",
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 16,
            "y": 6
          },
          "id": 85,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": true,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null as zero",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [
            {
              "alias": "IO Pending",
              "color": "#890F02"
            }
          ],
          "spaceLength": 10,
          "stack": true,
          "steppedLine": false,
          "targets": [
            {
              "alias": "IO Bytes TX",
              "dsType": "influxdb",
              "expr": "px_network_io_bytessent{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "Bytes TX",
              "measurement": "px_network_io_bytessent",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "IO Bytes RX",
              "dsType": "influxdb",
              "expr": "px_network_io_received_bytes{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "Bytes RX",
              "measurement": "px_network_io_received_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "C",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Network TX/RX",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 0,
            "y": 13
          },
          "id": 6,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": true,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null as zero",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [
            {
              "alias": "IO Pending",
              "color": "#890F02"
            }
          ],
          "spaceLength": 10,
          "stack": true,
          "steppedLine": false,
          "targets": [
            {
              "alias": "IO Pending",
              "dsType": "influxdb",
              "expr": "irate(px_disk_stats_read_seconds {cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}[5m])",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "instant": false,
              "intervalFactor": 1,
              "legendFormat": "Read {{cluster}} {{instance}}",
              "measurement": "px_cluster_pendingio",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "IO Pending",
              "dsType": "influxdb",
              "expr": "irate(px_disk_stats_write_seconds {cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}[5m])",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "instant": false,
              "intervalFactor": 1,
              "legendFormat": "Write  {{cluster}} {{instance}}",
              "measurement": "px_cluster_pendingio",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "PWX  Disk IO",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "dtdurations",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 8,
            "y": 13
          },
          "id": 86,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": true,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null as zero",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Cluster Disk Total",
              "dsType": "influxdb",
              "expr": "irate(px_disk_stats_read_bytes{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}[5m])",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "host"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_disk_total_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "sum"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Cluster Disk Used",
              "dsType": "influxdb",
              "expr": "irate(px_disk_stats_write_bytes{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}[5m])",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "host"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_disk_utilized_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "sum"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "PWX Disk throughput",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 16,
            "y": 13
          },
          "id": 87,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": true,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null as zero",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Cluster Disk Total",
              "dsType": "influxdb",
              "expr": "px_disk_stats_read_latency_seconds{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "host"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_disk_total_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "sum"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Cluster Disk Used",
              "dsType": "influxdb",
              "expr": "px_disk_stats_write_latency_seconds{cluster=~\"[[Cluster]]\",instance=~\"[[Instances]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "host"
                  ],
                  "type": "tag"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "{{cluster}}{{instance}}",
              "measurement": "px_cluster_disk_utilized_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "sum"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "PWX Disk Latency",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "dtdurations",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        }
      ],
      "refresh": "30s",
      "schemaVersion": 16,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": [
          {
            "allValue": null,
            "current": {},
            "datasource": "prometheus",
            "hide": 1,
            "includeAll": false,
            "label": "",
            "multi": false,
            "name": "Cluster",
            "options": [],
            "query": "px_cluster_cpu_percent",
            "refresh": 1,
            "regex": "/.*cluster=\"([^\"]*).*/",
            "sort": 0,
            "tagValuesQuery": "",
            "tags": [],
            "tagsQuery": "",
            "type": "query",
            "useTags": false
          },
          {
            "allValue": null,
            "current": {},
            "datasource": "prometheus",
            "hide": 1,
            "includeAll": false,
            "label": null,
            "multi": true,
            "name": "Instances",
            "options": [],
            "query": "px_cluster_cpu_percent",
            "refresh": 1,
            "regex": "/.*instance=\"([^\"]*).*/",
            "sort": 1,
            "tagValuesQuery": "",
            "tags": [],
            "tagsQuery": "",
            "type": "query",
            "useTags": false
          }
        ]
      },
      "time": {
        "from": "now-3h",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": [
          "5s",
          "10s",
          "30s",
          "1m",
          "5m",
          "15m",
          "30m",
          "1h",
          "2h",
          "1d"
        ],
        "time_options": [
          "5m",
          "15m",
          "1h",
          "6h",
          "12h",
          "24h",
          "2d",
          "7d",
          "30d"
        ]
      },
      "timezone": "",
      "title": "Portworx Node Dashboard",
      "uid": "Vhz18oTik",
      "version": 3
    }
  portworx-volume-dashboard.json: |-
    {
      "__inputs": [
        {
          "name": "DS_PROMETHEUS",
          "label": "prometheus",
          "description": "",
          "type": "datasource",
          "pluginId": "prometheus",
          "pluginName": "Prometheus"
        }
      ],
      "__requires": [
        {
          "type": "grafana",
          "id": "grafana",
          "name": "Grafana",
          "version": "5.0.0-beta1"
        },
        {
          "type": "panel",
          "id": "graph",
          "name": "Graph",
          "version": ""
        },
        {
          "type": "datasource",
          "id": "prometheus",
          "name": "Prometheus",
          "version": "1.0.0"
        },
        {
          "type": "panel",
          "id": "singlestat",
          "name": "Singlestat",
          "version": ""
        }
      ],
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "iteration": 1541099296083,
      "links": [],
      "panels": [
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 63,
          "panels": [],
          "title": "All Volumes",
          "type": "row"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "decimals": null,
          "format": "dtdurations",
          "gauge": {
            "maxValue": 30,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 3,
            "w": 5,
            "x": 0,
            "y": 1
          },
          "id": 77,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "Value",
          "targets": [
            {
              "expr": "avg(px_volume_vol_read_latency_seconds{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "hide": false,
              "instant": false,
              "interval": "",
              "intervalFactor": 4,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "15,25",
          "title": "Avg Read Latency (1m)",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "avg"
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 10,
            "x": 5,
            "y": 1
          },
          "id": 82,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": true,
            "hideEmpty": true,
            "hideZero": true,
            "max": false,
            "min": false,
            "rightSide": false,
            "show": true,
            "sort": "current",
            "sortDesc": false,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "topk (10,(px_volume_usage_bytes{cluster=~\"[[Cluster]]\"}/px_volume_capacity_bytes{cluster=~\"[[Cluster]]\"})*100)",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "{{volumename}}",
              "refId": "A"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Top n Volumes by Capacity",
          "tooltip": {
            "shared": false,
            "sort": 1,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 9,
            "x": 15,
            "y": 1
          },
          "id": 83,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": true,
            "hideEmpty": true,
            "hideZero": true,
            "max": false,
            "min": false,
            "rightSide": false,
            "show": true,
            "sideWidth": 250,
            "sort": "current",
            "sortDesc": false,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "topk(10,px_volume_depth_io{cluster=\"[[Cluster]]\"})",
              "format": "time_series",
              "intervalFactor": 1,
              "legendFormat": "{{volumename}}",
              "refId": "A"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Top n Volumes by IO depth",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "decimals": null,
          "format": "dtdurations",
          "gauge": {
            "maxValue": 30,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 3,
            "w": 5,
            "x": 0,
            "y": 4
          },
          "id": 78,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "Value",
          "targets": [
            {
              "expr": "avg(px_volume_vol_write_latency_seconds{cluster=~\"[[Cluster]]\"})",
              "format": "time_series",
              "hide": false,
              "interval": "",
              "intervalFactor": 4,
              "legendFormat": "",
              "refId": "B"
            }
          ],
          "thresholds": "15,25",
          "title": "Avg Write Latency (1m)",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "avg"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 7
          },
          "id": 55,
          "panels": [],
          "repeat": "Volumes",
          "title": "Volume: [[Volumes]]",
          "type": "row"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "rgba(245, 54, 54, 0.9)",
            "rgba(237, 129, 40, 0.89)",
            "rgba(50, 172, 45, 0.97)"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 3,
            "minValue": 0,
            "show": true,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 3,
            "w": 4,
            "x": 0,
            "y": 8
          },
          "id": 84,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "dsType": "influxdb",
              "expr": "px_volume_currhalevel{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "$__interval"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "{{volumename}}",
              "measurement": "px_volume_halevel",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": "",
          "title": "Current Replication Level (HA)",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "rgba(245, 54, 54, 0.9)",
            "rgba(237, 129, 40, 0.89)",
            "rgba(50, 172, 45, 0.97)"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 3,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 6,
            "w": 4,
            "x": 4,
            "y": 8
          },
          "id": 61,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "dsType": "influxdb",
              "expr": "px_volume_iopriority{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "$__interval"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "{{volumename}}",
              "measurement": "px_volume_halevel",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": "",
          "title": "I/O Priority",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "High",
              "value": "1"
            },
            {
              "op": "=",
              "text": "Medium",
              "value": "2"
            },
            {
              "op": "=",
              "text": "Low",
              "value": "3"
            }
          ],
          "valueName": "current"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "decimals": null,
          "format": "short",
          "gauge": {
            "maxValue": 30,
            "minValue": 0,
            "show": true,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 6,
            "w": 4,
            "x": 8,
            "y": 8
          },
          "id": 79,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "Value",
          "targets": [
            {
              "expr": "avg(px_volume_vol_read_latency_seconds{cluster=~\"[[Cluster]]\",volumename=~\"[[Volumes]]\"})",
              "format": "time_series",
              "hide": false,
              "instant": false,
              "interval": "",
              "intervalFactor": 4,
              "legendFormat": "",
              "refId": "A"
            }
          ],
          "thresholds": "15,25",
          "title": "Avg Read Latency",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "avg"
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "#299c46",
            "rgba(237, 129, 40, 0.89)",
            "#d44a3a"
          ],
          "datasource": "prometheus",
          "decimals": null,
          "format": "short",
          "gauge": {
            "maxValue": 30,
            "minValue": 0,
            "show": true,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 6,
            "w": 4,
            "x": 12,
            "y": 8
          },
          "id": 80,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": true
          },
          "tableColumn": "Value",
          "targets": [
            {
              "expr": "avg(px_volume_vol_write_latency_seconds{cluster=~\"[[Cluster]]\",volumename=~\"[[Volumes]]\"})",
              "format": "time_series",
              "hide": false,
              "intervalFactor": 4,
              "legendFormat": "",
              "refId": "B"
            }
          ],
          "thresholds": "15,25",
          "title": "Avg Write Latency",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "avg"
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 0,
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 16,
            "y": 8
          },
          "id": 8,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": true,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Disk Capacity",
              "dsType": "influxdb",
              "expr": "px_volume_capacity_bytes{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "1m"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": " Capacity",
              "measurement": "px_volume_capacity_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Disk Usage",
              "dsType": "influxdb",
              "expr": "px_volume_usage_bytes{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "1m"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": " Usage",
              "measurement": "px_volume_usage_bytes",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Volume Usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "decimals": 0,
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "rgba(245, 54, 54, 0.9)",
            "rgba(237, 129, 40, 0.89)",
            "rgba(50, 172, 45, 0.97)"
          ],
          "datasource": "prometheus",
          "format": "none",
          "gauge": {
            "maxValue": 3,
            "minValue": 0,
            "show": true,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 3,
            "w": 4,
            "x": 0,
            "y": 11
          },
          "id": 4,
          "interval": null,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "dsType": "influxdb",
              "expr": "px_volume_halevel{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "$__interval"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "{{volumename}}",
              "measurement": "px_volume_halevel",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "A",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": "",
          "title": "Configured Replication Level (HA)",
          "type": "singlestat",
          "valueFontSize": "80%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "current"
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 2,
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 0,
            "y": 14
          },
          "id": 59,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": true,
            "max": true,
            "min": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Bytes Written Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_num_long_reads{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "No. of  Reads ",
              "measurement": "px_volume_writethroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "C",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Bytes Read Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_num_long_writes{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "No. of  Writes ",
              "measurement": "px_volume_readthroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "D",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Volume total reads/writes",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "decimals": 0,
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 2,
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 8,
            "y": 14
          },
          "id": 60,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": true,
            "max": true,
            "min": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Bytes Written Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_iops{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "IOPs",
              "measurement": "px_volume_writethroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "C",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Volume IOPs",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "decimals": 0,
              "format": "none",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 2,
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 16,
            "y": 14
          },
          "id": 58,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": true,
            "max": true,
            "min": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Bytes Written Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_read_bytes{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "Bytes Read ",
              "measurement": "px_volume_writethroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "C",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Bytes Read Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_written_bytes{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "Bytes Written ",
              "measurement": "px_volume_readthroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "D",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Volume Read/Written Bytes",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "decimals": 0,
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 2,
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 0,
            "y": 20
          },
          "id": 5,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": true,
            "max": true,
            "min": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "repeat": null,
          "repeatDirection": "h",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Read Latency",
              "dsType": "influxdb",
              "expr": "px_volume_vol_read_latency_seconds{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "0"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "Read Latency",
              "measurement": "px_volume_vol_read_latency_seconds",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "C",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Write Latency",
              "dsType": "influxdb",
              "expr": "px_volume_vol_write_latency_seconds{volumename=~\"[[Volumes]]\"}\n",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "10s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "0"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "Write Latency",
              "measurement": "px_volume_vol_write_latency_seconds",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "B",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Volume Latency",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "dtdurations",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 8,
            "y": 20
          },
          "hideTimeOverride": true,
          "id": 57,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": true,
            "hideEmpty": true,
            "hideZero": true,
            "max": true,
            "min": false,
            "show": true,
            "sort": "current",
            "sortDesc": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "connected",
          "percentage": true,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "px_volume_depth_io{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "I/O Depth",
              "refId": "A",
              "step": 10
            }
          ],
          "thresholds": [],
          "timeFrom": "1h",
          "timeShift": null,
          "title": "Volume IO Depth",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "none",
              "label": "",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 2,
          "fill": 1,
          "gridPos": {
            "h": 6,
            "w": 8,
            "x": 16,
            "y": 20
          },
          "id": 3,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": true,
            "max": true,
            "min": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "repeat": null,
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Bytes Written Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_readthroughput{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "hide": false,
              "intervalFactor": 1,
              "legendFormat": "Bytes Read Throughput",
              "measurement": "px_volume_writethroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "C",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            },
            {
              "alias": "Bytes Read Throughput",
              "dsType": "influxdb",
              "expr": "px_volume_writethroughput{volumename=~\"[[Volumes]]\"}",
              "format": "time_series",
              "groupBy": [
                {
                  "params": [
                    "30s"
                  ],
                  "type": "time"
                },
                {
                  "params": [
                    "null"
                  ],
                  "type": "fill"
                }
              ],
              "intervalFactor": 1,
              "legendFormat": "Bytes Written Throughput",
              "measurement": "px_volume_readthroughput",
              "orderByTime": "ASC",
              "policy": "default",
              "refId": "D",
              "resultFormat": "time_series",
              "select": [
                [
                  {
                    "params": [
                      "gauge"
                    ],
                    "type": "field"
                  },
                  {
                    "params": [],
                    "type": "last"
                  }
                ]
              ],
              "tags": [
                {
                  "key": "volumename",
                  "operator": "=~",
                  "value": "/^$Volumes$/"
                },
                {
                  "condition": "AND",
                  "key": "cluster",
                  "operator": "=~",
                  "value": "/^$Cluster$/"
                }
              ]
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Volume Throughput",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "transparent": false,
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "decimals": 0,
              "format": "Bps",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": "0",
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        }
      ],
      "refresh": false,
      "schemaVersion": 16,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": [
          {
            "allValue": null,
            "current": {},
            "datasource": "prometheus",
            "hide": 0,
            "includeAll": false,
            "label": "Cluster",
            "multi": false,
            "name": "Cluster",
            "options": [],
            "query": "px_cluster_cpu_percent",
            "refresh": 1,
            "regex": "/.*cluster=\"([^\"]*).*/",
            "sort": 0,
            "tagValuesQuery": "",
            "tags": [],
            "tagsQuery": "",
            "type": "query",
            "useTags": false
          },
          {
            "allValue": null,
            "current": {},
            "datasource": "prometheus",
            "hide": 0,
            "includeAll": false,
            "label": "Volume Name",
            "multi": true,
            "name": "Volumes",
            "options": [],
            "query": "px_volume_depth_io {cluster=\"[[Cluster]]\"}",
            "refresh": 1,
            "regex": "/.*volumename=\"([^\"]*).*/",
            "sort": 1,
            "tagValuesQuery": "",
            "tags": [],
            "tagsQuery": "",
            "type": "query",
            "useTags": false
          }
        ]
      },
      "time": {
        "from": "now-3h",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": [
          "5s",
          "10s",
          "30s",
          "1m",
          "5m",
          "15m",
          "30m",
          "1h",
          "2h",
          "1d"
        ],
        "time_options": [
          "5m",
          "15m",
          "1h",
          "6h",
          "12h",
          "24h",
          "2d",
          "7d",
          "30d"
        ]
      },
      "timezone": "",
      "title": "Portworx Volume Dashboard",
      "uid": "LAgtVanmz",
      "version": 82
    }
  portworx-etcd-dashboard.json: |-
    {
      "__inputs": [
        {
          "name": "DS_PROMETHEUS",
          "label": "prometheus",
          "description": "",
          "type": "datasource",
          "pluginId": "prometheus",
          "pluginName": "Prometheus"
        }
      ],
      "__requires": [
        {
          "type": "grafana",
          "id": "grafana",
          "name": "Grafana",
          "version": "5.0.0-beta1"
        },
        {
          "type": "panel",
          "id": "graph",
          "name": "Graph",
          "version": ""
        },
        {
          "type": "datasource",
          "id": "prometheus",
          "name": "Prometheus",
          "version": "1.0.0"
        },
        {
          "type": "panel",
          "id": "singlestat",
          "name": "Singlestat",
          "version": ""
        }
      ],
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": "-- Grafana --",
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "type": "dashboard"
          }
        ]
      },
      "description": "etcd sample Grafana dashboard with Prometheus",
      "editable": true,
      "gnetId": null,
      "graphTooltip": 0,
      "id": null,
      "links": [],
      "panels": [
        {
          "cacheTimeout": null,
          "colorBackground": false,
          "colorValue": false,
          "colors": [
            "rgba(245, 54, 54, 0.9)",
            "rgba(237, 129, 40, 0.89)",
            "rgba(50, 172, 45, 0.97)"
          ],
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "format": "none",
          "gauge": {
            "maxValue": 100,
            "minValue": 0,
            "show": false,
            "thresholdLabels": false,
            "thresholdMarkers": true
          },
          "gridPos": {
            "h": 7,
            "w": 6,
            "x": 0,
            "y": 0
          },
          "id": 28,
          "interval": null,
          "isNew": true,
          "links": [],
          "mappingType": 1,
          "mappingTypes": [
            {
              "name": "value to text",
              "value": 1
            },
            {
              "name": "range to text",
              "value": 2
            }
          ],
          "maxDataPoints": 100,
          "nullPointMode": "connected",
          "nullText": null,
          "postfix": "",
          "postfixFontSize": "50%",
          "prefix": "",
          "prefixFontSize": "50%",
          "rangeMaps": [
            {
              "from": "null",
              "text": "N/A",
              "to": "null"
            }
          ],
          "sparkline": {
            "fillColor": "rgba(31, 118, 189, 0.18)",
            "full": false,
            "lineColor": "rgb(31, 120, 193)",
            "show": false
          },
          "tableColumn": "",
          "targets": [
            {
              "expr": "sum(etcd_server_has_leader)",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "",
              "metric": "etcd_server_has_leader",
              "refId": "A",
              "step": 20
            }
          ],
          "thresholds": "",
          "title": "Up",
          "type": "singlestat",
          "valueFontSize": "200%",
          "valueMaps": [
            {
              "op": "=",
              "text": "N/A",
              "value": "null"
            }
          ],
          "valueName": "avg"
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 0,
          "gridPos": {
            "h": 7,
            "w": 10,
            "x": 6,
            "y": 0
          },
          "id": 23,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(rate(grpc_server_started_total{grpc_type=\"unary\"}[5m]))",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "RPC Rate",
              "metric": "grpc_server_started_total",
              "refId": "A",
              "step": 2
            },
            {
              "expr": "sum(rate(grpc_server_handled_total{grpc_type=\"unary\",grpc_code!=\"OK\"}[5m]))",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "RPC Failed Rate",
              "metric": "grpc_server_handled_total",
              "refId": "B",
              "step": 2
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "RPC Rate",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "ops",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 0,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 16,
            "y": 0
          },
          "id": 41,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": true,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(grpc_server_started_total{grpc_service=\"etcdserverpb.Watch\",grpc_type=\"bidi_stream\"}) - sum(grpc_server_handled_total{grpc_service=\"etcdserverpb.Watch\",grpc_type=\"bidi_stream\"})",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "Watch Streams",
              "metric": "grpc_server_handled_total",
              "refId": "A",
              "step": 4
            },
            {
              "expr": "sum(grpc_server_started_total{grpc_service=\"etcdserverpb.Lease\",grpc_type=\"bidi_stream\"}) - sum(grpc_server_handled_total{grpc_service=\"etcdserverpb.Lease\",grpc_type=\"bidi_stream\"})",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "Lease Streams",
              "metric": "grpc_server_handled_total",
              "refId": "B",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Active Streams",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": "",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": null,
          "editable": true,
          "error": false,
          "fill": 0,
          "grid": {},
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 0,
            "y": 7
          },
          "id": 1,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "etcd_debugging_mvcc_db_total_size_in_bytes",
              "format": "time_series",
              "hide": false,
              "interval": "",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} DB Size",
              "metric": "",
              "refId": "A",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "DB Size",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "cumulative"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "bytes",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 0,
          "grid": {},
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 8,
            "y": 7
          },
          "id": 3,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 1,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": true,
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) by (instance, le))",
              "format": "time_series",
              "hide": false,
              "intervalFactor": 2,
              "legendFormat": "{{instance}} WAL fsync",
              "metric": "etcd_disk_wal_fsync_duration_seconds_bucket",
              "refId": "A",
              "step": 4
            },
            {
              "expr": "histogram_quantile(0.99, sum(rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) by (instance, le))",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} DB fsync",
              "metric": "etcd_disk_backend_commit_duration_seconds_bucket",
              "refId": "B",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Disk Sync Duration",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "cumulative"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "s",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 0,
          "gridPos": {
            "h": 7,
            "w": 8,
            "x": 16,
            "y": 7
          },
          "id": 29,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "process_resident_memory_bytes",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} Resident Memory",
              "metric": "process_resident_memory_bytes",
              "refId": "A",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Memory",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 5,
          "gridPos": {
            "h": 7,
            "w": 6,
            "x": 0,
            "y": 14
          },
          "id": 22,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": true,
          "steppedLine": false,
          "targets": [
            {
              "expr": "rate(etcd_network_client_grpc_received_bytes_total[5m])",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} Client Traffic In",
              "metric": "etcd_network_client_grpc_received_bytes_total",
              "refId": "A",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Client Traffic In",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 5,
          "gridPos": {
            "h": 7,
            "w": 6,
            "x": 6,
            "y": 14
          },
          "id": 21,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": true,
          "steppedLine": false,
          "targets": [
            {
              "expr": "rate(etcd_network_client_grpc_sent_bytes_total[5m])",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} Client Traffic Out",
              "metric": "etcd_network_client_grpc_sent_bytes_total",
              "refId": "A",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Client Traffic Out",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "Bps",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 0,
          "gridPos": {
            "h": 7,
            "w": 6,
            "x": 12,
            "y": 14
          },
          "id": 20,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(rate(etcd_network_peer_received_bytes_total[5m])) by (instance)",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} Peer Traffic In",
              "metric": "etcd_network_peer_received_bytes_total",
              "refId": "A",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Peer Traffic In",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "Bps",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": null,
          "editable": true,
          "error": false,
          "fill": 0,
          "grid": {},
          "gridPos": {
            "h": 7,
            "w": 6,
            "x": 18,
            "y": 14
          },
          "id": 16,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(rate(etcd_network_peer_sent_bytes_total[5m])) by (instance)",
              "format": "time_series",
              "hide": false,
              "interval": "",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} Peer Traffic Out",
              "metric": "etcd_network_peer_sent_bytes_total",
              "refId": "A",
              "step": 4
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Peer Traffic Out",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "cumulative"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "Bps",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "editable": true,
          "error": false,
          "fill": 0,
          "gridPos": {
            "h": 7,
            "w": 12,
            "x": 0,
            "y": 21
          },
          "id": 40,
          "isNew": true,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "sum(rate(etcd_server_proposals_failed_total[5m]))",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "Proposal Failure Rate",
              "metric": "etcd_server_proposals_failed_total",
              "refId": "A",
              "step": 2
            },
            {
              "expr": "sum(etcd_server_proposals_pending)",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "Proposal Pending Total",
              "metric": "etcd_server_proposals_pending",
              "refId": "B",
              "step": 2
            },
            {
              "expr": "sum(rate(etcd_server_proposals_committed_total[5m]))",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "Proposal Commit Rate",
              "metric": "etcd_server_proposals_committed_total",
              "refId": "C",
              "step": 2
            },
            {
              "expr": "sum(rate(etcd_server_proposals_applied_total[5m]))",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "Proposal Apply Rate",
              "refId": "D",
              "step": 2
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Raft Proposals",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": "",
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "prometheus",
          "decimals": 0,
          "editable": true,
          "error": false,
          "fill": 0,
          "gridPos": {
            "h": 7,
            "w": 12,
            "x": 12,
            "y": 21
          },
          "id": 19,
          "isNew": true,
          "legend": {
            "alignAsTable": false,
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "rightSide": false,
            "show": false,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 2,
          "links": [],
          "nullPointMode": "connected",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "expr": "changes(etcd_server_leader_changes_seen_total[1d])",
              "format": "time_series",
              "intervalFactor": 2,
              "legendFormat": "{{instance}} Total Leader Elections Per Day",
              "metric": "etcd_server_leader_changes_seen_total",
              "refId": "A",
              "step": 2
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "Total Leader Elections Per Day",
          "tooltip": {
            "msResolution": false,
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        }
      ],
      "refresh": false,
      "schemaVersion": 16,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-15m",
        "to": "now"
      },
      "timepicker": {
        "now": true,
        "refresh_intervals": [
          "5s",
          "10s",
          "30s",
          "1m",
          "5m",
          "15m",
          "30m",
          "1h",
          "2h",
          "1d"
        ],
        "time_options": [
          "5m",
          "15m",
          "1h",
          "6h",
          "12h",
          "24h",
          "2d",
          "7d",
          "30d"
        ]
      },
      "timezone": "browser",
      "title": "Etcd Dashboard",
      "uid": "CRWfZ_tiz",
      "version": 15
    }
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: grafana-source-config
  namespace: kube-system
  labels:
    role: grafana-source-configfiles
    grafana: portworx
data:
  datasource.yml: |-
    # config file version
    apiVersion: 1
    # list of datasources that should be deleted from the database
    deleteDatasources:
      - name: prometheus
        orgId: 1
    # list of datasources to insert/update depending
    # whats available in the database
    datasources:
      # <string, required> name of the datasource. Required
    - name: prometheus
      # <string, required> datasource type. Required
      type: prometheus
      # <string, required> access mode. direct or proxy. Required
      access: proxy
      # <int> org id. will default to orgId 1 if not specified
      orgId: 1
      # <string> url
      url: http://prometheus:9090
      # <string> database password, if used
      password:
      # <string> database user, if used
      user:
      # <string> database name, if used
      database:
      # <bool> enable/disable basic auth
      basicAuth: true
      # <string> basic auth username
      basicAuthUser: admin
      # <string> basic auth password
      basicAuthPassword: foobar
      # <bool> enable/disable with credentials headers
      withCredentials:
      # <bool> mark as default datasource. Max one per org
      isDefault:
      # <map> fields that will be converted to json and stored in json_data
      jsonData:
        graphiteVersion: "1.1"
        tlsAuth: false
        tlsAuthWithCACert: false
      # <string> json object of data that will be encrypted.
      secureJsonData:
        tlsCACert: "..."
        tlsClientCert: "..."
        tlsClientKey: "..."
      version: 1
      # <bool> allow users to edit datasources from the UI.
      editable: true
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: grafana-dashboard-config
  namespace: kube-system
  labels:
    role: grafana-dashboard-configfiles
    grafana: portworx
data:
  dashboards.yml: |-
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-system
  labels:
    app: grafana
spec:
  type: NodePort
  ports:
    - port: 3000
  selector:
    app: grafana
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: grafana
  namespace: kube-system
  labels:
    app: grafana
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - image: grafana/grafana:5.0.4
          name: grafana
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              cpu: 100m
              memory: 100Mi
            requests:
              cpu: 100m
              memory: 100Mi
          readinessProbe:
            httpGet:
              path: /login
              port: 3000
          volumeMounts:
            - name: grafana-dash-config
              mountPath: /etc/grafana/provisioning/dashboards
            - name: dashboard-templates
              mountPath: /var/lib/grafana/dashboards
            - name: grafana-source-config
              mountPath: /etc/grafana/provisioning/datasources
      volumes:
        - name: grafana-source-config
          configMap:
            name: grafana-source-config
        - name: grafana-dash-config
          configMap:
            name: grafana-dashboard-config
        - name: dashboard-templates
          configMap:
            name: grafana-dashboards
```

`kubectl apply -f grafana-deployment.yaml`

#### Grafana access details

Grafana can be be accessed using a NodePort service.

First get the node port that grafana is using

  ```text
  kubectl get svc -n kube-system grafana
  ```

Access the Grafana dashboard by navigating to `http://<master_ip>:<service_nodeport>`. You would need to create a datasource for the Portworx grafana dashboard metrics to be populated.
Navigate to Configurations --> Datasources.
Create a datasource named `prometheus`. Enter the Prometheus endpoint as obtained in the install verification step for Prometheus from the above section.

![grafanadatasource](/img/datasource-creation-grafana.png)

### Post install verification

Select the Portworx volume metrics dashboard on Grafana to view the Portworx metrics.
![grafanadashboard](/img/grafana-portworx-dashboard.png)
