-------------------
| 1. POWERSHELL   |
-------------------
1.1 Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

1.2 Kubectl, Helm, Git, VirtualBox, Minikube
choco install -y kubernetes-cli kubernetes-helm git virtualbox minikube podman-cli podman-machine base64

----- BEFORE SLIDES -----
minikube start --driver=virtualbox --download-only
-------------------------


---- SLIDES -----
Observability:
- Logs
- Metrics
- Traces
+ Interference

Demo:
- Trace example
- Spanmetrics example
- Metric->Trace
- Logs->Trace->Logs

Stack:
- Minikube
  - VirtualBox
  - Podman
  - Docker
- OpenLens
- Twuni Registry
- LGTM
- OpenTelemetry

- Loki architecture
  https://grafana.com/docs/loki/latest/get-started/components/
- Tempo architecture
  https://grafana.com/docs/tempo/latest/operations/architecture/
- Mimir architecture:
  https://grafana.com/docs/mimir/latest/get-started/about-grafana-mimir-architecture/

- Loki <> Prometheus labels
  - Cardinality

- MetricsQL advantages
  https://docs.victoriametrics.com/MetricsQL.html#metricsql-features

- OpenTelemetry
  History: OpenCensus(libs), OpenTracing(api), Zipkin, Jaeger
  - Transport: http, grpc, kafka
  - Standard: fields, name conventions, config, envvars
  - Instrumentation: sdk, starter, agents
  
  


1.3 Helm repos
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jetstack https://charts.jetstack.io
helm repo add twuni https://helm.twun.io
helm repo add opentelemetry https://open-telemetry.github.io/opentelemetry-helm-charts

1.4 Git Repo
git clone https://github.com/mmorev/observability-practice
cd observability-practice

1.5 Minikube start
minikube start --driver=virtualbox --no-vtx-check --memory=6g --dns-proxy --image-repository=auto --addons=default-storageclass --addons=ingress --addons=ingress-dns
minikube start --driver=podman --addons=default-storageclass --addons=ingress --addons=ingress-dns

1.6 Minikube ingress
https://minikube.sigs.k8s.io/docs/handbook/addons/ingress-dns/#installation
Add-DnsClientNrptRule -Namespace ".test" -NameServers "$(minikube ip)"
# Get-DnsClientNrptRule | Remove-DnsClientNrptRule -Force

OPTIONAL DRIVER=VBOX
minikube stop
& 'C:\Program Files\Oracle\VirtualBox\VBoxManage.exe' modifyvm minikube --paravirt-provider=hyperv --vram=16
minikube start

-------------------
| 2. WSL          |
-------------------
# 2.1 sudoers
# sudo sed -Ei 's/^(root.*) ALL$/\1 NOPASSWD:ALL/' /etc/sudoers
# 2.2 git, kubectl, helm
# 2.2.1 kubectl
# curl -LO "https://dl.k8s.io/release/$(curl -sSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
# sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
# 2.2.2 Helm
# curl -L https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# 2.2.3 git and other
# sudo apt install git

-------------------
| 3. GUI          |
-------------------
3.1 OpenLens
https://github.com/MuhammedKalkan/OpenLens/releases

3.2 OpenLens - pod tools
File - Extensions - @alebcay/openlens-node-pod-menu

-------------------
function hui {& helm upgrade --install $args}
alias hui='helm upgrade --install'

1.7 Minikube CA
hui cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set installCRDs=true
kubectl apply -f .\minikube\minikube-cert-manager.yaml

1.8 Minikube Registry
hui registry twuni/docker-registry -n kube-system --values .\minikube\registry-ingress.yaml

kubectl create secret docker-registry registry-auth --docker-server=registry.test --docker-username=admin --docker-password=admin

kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-auth"}]}'

-------------------
| 5. LGTM         |
-------------------

----- SKIP -----
# GRAFANA
# hui grafana grafana/grafana --namespace grafana --values .\lgtm\grafana-values.yaml
#
# MIMIR
# hui mimir-distributed grafana/mimir-distributed --values .\lgtm\mimir-distributed-values.yaml -n mimir --create-namespace
----- END ------

kubectl create ns prometheus
kubectl create ns grafana

hui prometheus prometheus/kube-prometheus-stack -n prometheus --values .\lgtm\prom-stack-values.yaml
hui loki grafana/loki --namespace grafana --values .\lgtm\loki-values.yaml
hui tempo grafana/tempo --namespace grafana --values .\lgtm\tempo-values.yaml

----- LOGIN GRAFANA
kubectl get secret --namespace grafana prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d | Set-Clipboard

--------------------
| 5.2 Data Sources |
--------------------

#- Loki - http://loki:3100
#  Derived
#  TraceID
#  "traceid":"(\w+)"
#  ${__value.raw}
#  Internal
#- Tempo - http://tempo:3100
#  Trace to Logs:
#  {${__tags}} |= `"traceid":"${__trace.traceId}"`
#- Prometheus - http://prometheus-server.prometheus:9090
#  ver 2.47.0

--------------------
| Opentelemetry    |
--------------------
kubectl create ns opentelemetry

----- BASIC    -----
kubectl -n opentelemetry create configmap otelcol --from-file .\opentelemetry\simple\otelcol.yaml
kubectl -n opentelemetry apply -f .\opentelemetry\simple\deployment.yaml -f .\opentelemetry\simple\service.yaml

----- DROP     -----
kubectl -n opentelemetry delete service otelcol
kubectl -n opentelemetry delete deployment otelcol
kubectl -n opentelemetry delete configmap otelcol

----- ADVANCED -----
hui opentelemetry-operator opentelemetry/opentelemetry-operator -n opentelemetry --create-namespace --values .\opentelemetry\operator\operator-values.yaml

kubectl apply -f .\opentelemetry\operator\collector-crd.yaml -n opentelemetry
kubectl apply -f .\opentelemetry\operator\instrumentation-crd.yaml -n opentelemetry

-----------------------
| 6. Spring Javaagent |
-----------------------
podman machine init --rootful
podman machine start

