# Prometheus

> 系統監控報警框架

<img src="https://prometheus.io/assets/architecture.png"></img>

## 主要組件

- Prometheus Server：核心的伺服器端，主要負責存儲時間序列數據
- Prometheus Client : 客戶端，將需要被監控的服務生成對應的 metrics
- Push Gateway : 通過 Push Gateway 主動向服務端推送數據，主要用於存在時間較短的任務
- Exporter：類似 Agent，安裝在客戶端上，用來監控數據，並向伺服器端提供監控數據樣本
- Alertmanager：用來處理報警，將告警信息發送給用戶

## Installation

### [minikube](https://minikube.sigs.k8s.io/docs/start/)

#### Installaion

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

### [Helm](https://helm.sh/)

#### Installaion

```
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -

sudo apt-get install apt-transport-https --yes

echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update

sudo apt-get install helm
```

### [Promethues](https://prometheus.io/)


#### Installaion

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace -f values.yaml -n monitoring
```

#### Upgrade

```
helm upgrade prometheus prometheus-community/prometheus -f values.yaml -n monitoring
```

#### Rules

- [example 1](https://github.com/camilb/prometheus-kubernetes/blob/master/manifests/prometheus/prometheus-k8s-rules.yaml)
- [example 2](https://awesome-prometheus-alerts.grep.to/rules#kubernetes)

```
- alert: KubernetesMemoryPressure
  expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: Kubernetes memory pressure (instance {{ $labels.instance }})
    description: "{{ $labels.node }} has MemoryPressure condition\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

- alert: KubernetesDiskPressure
  expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: Kubernetes disk pressure (instance {{ $labels.instance }})
    description: "{{ $labels.node }} has DiskPressure condition\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

- alert: KubernetesPodNotHealthy
  expr: min_over_time(sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown|Failed"})[15m:1m]) > 0
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: Kubernetes Pod not healthy (instance {{ $labels.instance }})
    description: "Pod has been in a non-ready state for longer than 15 minutes.\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"

- alert: KubernetesApiServerErrors
  expr: sum(rate(apiserver_request_total{job="apiserver",code=~"^(?:5..)$"}[1m])) / sum(rate(apiserver_request_total{job="apiserver"}[1m])) * 100 > 3
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: Kubernetes API server errors (instance {{ $labels.instance }})
    description: "Kubernetes API server is experiencing high error rate\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

#### PromQL

- [example](https://prometheus.io/docs/prometheus/latest/querying/examples/#query-examples)

- Container CPU Usage:

```
sum by(pod)(rate(container_cpu_usage_seconds_total{namespace="monitoring"}[1h]))
```

- Container Memory Usage:

```
sum by(pod)(rate(container_memory_max_usage_bytes{namespace="monitoring"}[1h]))
```

#### Others
- post-forward in the same shell:

```
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace monitoring port-forward $POD_NAME 9090 --address 0.0.0.0 &
```
- sudo:
```  
export POD_NAME=$(sudo kubectl get pods --namespace  monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")

sudo kubectl --namespace monitoring port-forward $POD_NAME 9090 --address 0.0.0.0 &
```

### [AlertManger](https://prometheus.io/docs/alerting/latest/alertmanager/)


#### Config

```
global:
  resolve_timeout: 2m
  smtp_smarthost: "host:port"
  smtp_hello: "hello.abc.com"
  smtp_from: "DEV <DEV@hello.abc.com>"
  smtp_auth_username: "username"
  smtp_auth_password: "password"
  smtp_require_tls: true

receivers:
  - name: "email"
    email_configs:
      - to: "recevier@abc.com"
        send_resolved: true
        tls_config:
          insecure_skip_verify: true
route:
  group_by: ["alertname"]
  group_wait: 10s
  group_interval: 30m
  receiver: email
  repeat_interval: 30m
```

#### Others

- post-forward in the same shell:

```
export POD_NAME=$(kubectl get pods --namespace  monitoring -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace  monitoring port-forward $POD_NAME 9093  --address 0.0.0.0 &
```
- sudo:
```
export POD_NAME=$(sudo kubectl get pods --namespace  monitoring -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")

sudo kubectl --namespace  monitoring port-forward $POD_NAME 9093  --address 0.0.0.0 &  
```

### [Grafana](https://grafana.com/oss/grafana/)


#### Installaion

```
helm repo add grafana https://grafana.github.io/helm-charts

helm install grafana grafana/grafana -n monitoring
```

#### Upgrade

```
upgrade with helm

helm upgrade grafana grafana/grafana -f values.yaml -n monitoring
```

#### Others

- Get your 'admin' user password by running:

```
kubectl get secret --namespace  monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

- post-forward in the same shell:

```
export POD_NAME=$(kubectl get pods --namespace  monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace  monitoring port-forward $POD_NAME 3000  --address 0.0.0.0 &
```

- sudo:

```
sudo kubectl get secret --namespace  monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

export POD_NAME=$(sudo kubectl get pods --namespace  monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

sudo kubectl --namespace  monitoring port-forward $POD_NAME 3000  --address 0.0.0.0 &
```

## 數據流程

> 從監控目標(Prometheus target)中或者間接通過推送網關(Pushgateway)來獲取監控指標，在本地存儲所有抓取到的樣本數據，並對此數據執行一系列規則，來匯總和記錄現有數據的新時間序列或生成告警。最終通過 Grafana 或者其他工具來實現監控數據的可視化。

## Metrics - Data Type

> 指標型態

- Counter : 收集的數據是按照某個趨勢(增加/減少)一直變化的
- Gauge : 收集的數據為一個瞬時的值，與時間沒有關係，可任意變高變低
- Histogram : 表示一段時間範圍內對數據進行採樣，並能夠對其指- Summary : 表示採樣內部數據採樣結果，直接存儲了 Quantile 數據，而不是根據統計區間計算出來的

```
※ Histogram vs Summary

Histogram 需要通過 _bucket 計算分位數，而 Summary 直接存儲了分位數的值
```

## Metrics - PromQL

> Prometheus query language

- Operators : +, -, \*, / …
- Functions : increase, rate, delta …

*Example :*

```
sum(increase(test[1m]))/sum(increase(test [1m]))
```

*Reference :*
[https://prometheus.io/docs/prometheus/latest/querying/basics/](https://prometheus.io/docs/prometheus/latest/querying/basics/)

### Endpoint

```
/metrics
```
