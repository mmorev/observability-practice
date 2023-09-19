# Install tools

## Git, Minikube, Docker
```bash
apt update && apt install -y git docker minikube
```

## kubectl, helm
kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -sSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
```bash
curl -L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## OpenLens
https://github.com/MuhammedKalkan/OpenLens/releases

# Repos

## Practice git repo
```bash
git clone https://github.com/mmorev/observability-practice
cd observability-practice
```

## Required examples git repos
```bash
git clone https://github.com/open-telemetry/opentelemetry-java-examples.git
git clone https://github.com/eugenp/tutorials.git
```

## Helm repos
```bash
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jetstack https://charts.jetstack.io
helm repo add twuni https://helm.twun.io
helm repo add opentelemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

# Prepare environment

## Minikube
```bash 
minikube start --driver=docker --addons=default-storageclass --addons=ingress --addons=ingress-dns --memory=4g --cpus=max
```

## OpenLens
Запускаем, проверяем что кластер стартовал.

Добавляем плагин для работы с Node/Pod Logs/CLI:

File -> Extensions -> @alebcay/openlens-node-pod-menu -> Install

## Ingress DNS
#### Необязательно! Требует рестарта NetworkManager, при этом сеть отключится и вас выкинет из Zoom

Manual: https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/#installation

## Make thing easier :)
```bash
alias hui='helm upgrade --install'
```

## Cert Manager
```bash
hui cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set installCRDs=true
kubectl apply -f ./minikube/minikube-cert-manager.yaml
```

## Docker Registry
```bash
hui registry twuni/docker-registry -n kube-system --values ./minikube/registry-values.yaml

kubectl create secret docker-registry registry-auth --docker-server=registry.test --docker-username=admin --docker-password=admin

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-auth"}]}'
```

Резолвим адрес
```bash
echo $(minikube ip) registry.test | sudo tee -a /etc/hosts
```

Логинимся в регистри

```bash
docker login registry.test -u admin -p admin
```

# LGTM Stack

## Dedicated Grafana - для общего развития, ПРОПУСКАЕМ!
```bash
hui grafana grafana/grafana --namespace grafana --values ./lgtm/grafana-values.yaml
```

## Mimir - для общего развития, ПРОПУСКАЕМ!
```bash
hui mimir-distributed grafana/mimir-distributed --values ./lgtm/mimir-distributed-values.yaml -n mimir --create-namespace
```

## Kube-Prometheus Stack

Установка одним чартом Grafana, Prometheus, Kube-State-Metrics, Node Exporter и экспортеров control-plane
```bash
kubectl create ns prometheus
kubectl create ns grafana

hui prometheus prometheus/kube-prometheus-stack -n prometheus --values ./lgtm/prom-stack-values.yaml
```

Резолвим адрес
```bash
echo $(minikube ip) registry.test | sudo tee -a /etc/hosts
```

Loki
```bash
hui loki grafana/loki --namespace grafana --values ./lgtm/loki-values.yaml
```

Tempo
```bash
hui tempo grafana/tempo --namespace grafana --values ./lgtm/tempo-values.yaml
```

Получаем пароль от графаны, логин - admin
```bash
kubectl get secret --namespace grafana prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Открываем графану, проверяем - https://grafana.test/

# Grafana - Provision Datasources

``` yaml
  additionalDataSources:
  - name: Tempo
    uid: tempo
    type: tempo
    url: http://tempo.grafana:3100
    jsonData:
      tracesToLogsV2:
        datasourceUid: 'loki'
        spanStartTimeShift: '1h'
        spanEndTimeShift: '-1h'
        tags: [{key: "service_name", value: "service_name"}]
        customQuery: true
        query: '{$${__tags}} |= `"traceid":"$${__trace.traceId}"` | json'
      tracesToMetrics:
        datasourceUid: prometheus
        queries:
        - name: 95% Latency
          query: >
            histogram_quantile(0.95, sum(rate(latency_bucket{$$__tags}[$$__rate_interval])) by (le, operation))
        - name: RPS
          query: sum(rate(latency_count{$$__tags}[$$__rate_interval])) by (operation)
        spanEndTimeShift: -1h
        spanStartTimeShift: 1h
        tags:
        - key: service.name
          value: service_name
        - key: span.name
          value: operation
      serviceMap:
        datasourceUid: 'prometheus'
      lokiSearch:
        datasourceUid: 'loki'
      nodeGraph:
        enabled: true
  - name: Loki
    uid: loki
    type: loki
    url: http://loki.grafana:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: '"traceid":"(\w+)"'
          name: TraceID
          url: '$${__value.raw}'
  - name: Prometheus
    uid: prometheus
    type: prometheus
    url: http://promstack-prometheus.prometheus:9090
    jsonData:
      prometheusType: Prometheus
      exemplarTraceIdDestinations:
        - datasourceUid: tempo
          name: trace_id
```

# Opentelemetry

```kubectl create ns opentelemetry```

## Simple install
```
kubectl -n opentelemetry create configmap otelcol --from-file ./opentelemetry/simple/otelcol.yaml
kubectl -n opentelemetry apply -f ./opentelemetry/simple/deployment.yaml -f ./opentelemetry/simple/service.yaml
```

### Undo changes
```
kubectl -n opentelemetry delete service otelcol
kubectl -n opentelemetry delete deployment otelcol
kubectl -n opentelemetry delete configmap otelcol
```

## Advanced install
```
hui opentelemetry-operator opentelemetry/opentelemetry-operator -n opentelemetry --create-namespace --values ./opentelemetry/operator/operator-values.yaml

kubectl apply -f ./opentelemetry/operator/collector-crd.yaml -n opentelemetry
kubectl apply -f ./opentelemetry/operator/instrumentation-crd.yaml -n opentelemetry
```

# Spring Javaagent example

cd 