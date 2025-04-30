## cluster

```bash
# Create a Kind cluster
kind create cluster --config kind-config.yaml
```


## loki

```bash
helm upgrade --install loki grafana/loki --version 6.27.0 -n loki --create-namespace -f values-loki.yaml

# Loki has been configured with a gateway (nginx) to support reads and writes from a single component.
# You can send logs from inside the cluster using the cluster DNS:
# http://loki-gateway.loki.svc.cluster.local/loki/api/v1/push
# You can test to send data from outside the cluster by port-forwarding the gateway to your local machine:
kubectl port-forward --namespace loki svc/loki-gateway 3100:80

# And then using http://127.0.0.1:3100/loki/api/v1/push URL as shown below:
curl -H "Content-Type: application/json" -XPOST -s "http://127.0.0.1:3100/loki/api/v1/push"  \
--data-raw "{\"streams\": [{\"stream\": {\"job\": \"test\"}, \"values\": [[\"$(date +%s)000000000\", \"fizzbuzz\"]]}]}"

# Then verify that Loki did receive the data using the following command:
curl "http://127.0.0.1:3100/loki/api/v1/query_range" --data-urlencode 'query={job="test"}' | jq .data.result

# If Grafana operates within the cluster, you'll set up a new Loki datasource by utilizing the following URL:
# http://loki-gateway.loki.svc.cluster.local/
```


## promtail

```bash
helm upgrade --install promtail grafana/promtail --version 6.16.6 -n loki -f values-promtail.yaml

# Verify the application is working by running these commands:
kubectl --namespace loki port-forward daemonset/promtail 3101
curl http://127.0.0.1:3101/metrics
```


## prometheus

```bash
helm upgrade --install prometheus prometheus-community/prometheus --version 27.5.1 -n loki

# The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
# prometheus-server.loki.svc.cluster.local
# Get the Prometheus server URL by running these commands in the same shell:
export POD_NAME=$(kubectl get pods --namespace loki -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace loki port-forward $POD_NAME 9090

# The Prometheus alertmanager can be accessed via port 9093 on the following DNS name from within your cluster:
# prometheus-alertmanager.loki.svc.cluster.local
# Get the Alertmanager URL by running these commands in the same shell:
export POD_NAME=$(kubectl get pods --namespace loki -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace loki port-forward $POD_NAME 9093

# The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
# prometheus-prometheus-pushgateway.loki.svc.cluster.local
# Get the PushGateway URL by running these commands in the same shell:
export POD_NAME=$(kubectl get pods --namespace loki -l "app=prometheus-pushgateway,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace loki port-forward $POD_NAME 9091
```


## grafana

```bash
helm upgrade --install grafana grafana/grafana --version 8.10.1 -n loki -f values-grafana.yaml

# Get your 'admin' user password by running:
kubectl get secret --namespace loki grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:
# grafana.loki.svc.cluster.local
# Get the Grafana URL to visit by running these commands in the same shell:
export POD_NAME=$(kubectl get pods --namespace loki -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace loki port-forward $POD_NAME 3000
```


## kamaji

```bash
# Install cert-manager
helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install cert-manager bitnami/cert-manager --version 1.4.12 -n certmanager-system --create-namespace --set "installCRDs=true"

# Install Metal LB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
GW_IP=$(docker network inspect -f '{{range .IPAM.Config}}{{.Gateway}}{{end}}' kind)
NET_IP=$(echo ${GW_IP} | sed -E 's|^([0-9]+\.[0-9]+)\..*$|\1|g')
sed "s|__NET_IP__|$NET_IP|g" metallb-config-template.yaml > metallb-config.yaml
kubectl apply -f metallb-config.yaml

# Install Kamaji
helm repo add clastix https://clastix.github.io/charts
helm upgrade --install kamaji clastix/kamaji --version 1.0.0 -n kamaji-system --create-namespace --set 'resources=null'

# Create a Tenant Control Plane
kubectl apply -f kamaji_v1alpha1_datastore_etcd.yaml
kubectl apply -f kamaji_v1alpha1_tenantcontrolplane.yaml
kubectl get tcp -w -n kamaji-client

# Get kube config
k get -n kamaji-client secrets/k8s-133-admin-kubeconfig -o jsonpath='{.data.admin\.conf}'|base64 -d > conf/kamaji.conf
```


## k8sviz

```bash
# Gen diagram by namespace
./k8sviz.sh -n kamaji-system
```


## kind cloud provider

```bash
docker registry.k8s.io/cloud-provider-kind/cloud-controller-manager:v0.6.0 run --rm --network kind -v /var/run/docker.sock:/var/run/docker.sock cloud-provider-kind

kubectl label node kamaji-cluster-control-plane node.kubernetes.io/exclude-from-external-load-balancers-
```

## minio

```bash
# Prerequisites
k describe -n kube-system pod/kube-controller-manager-kamaji-cluster-control-plane|grep cluster-signing-cert-file -A1

helm repo add minio-operator https://operator.min.io
helm upgrade --install operator minio-operator/operator --version 7.0.0 -n minio-operator --create-namespace
kubectl get all -n minio-operator

helm upgrade --install tenant minio-operator/tenant --version 7.0.0 -f values-minio.yaml -n minio-tenant --create-namespace
watch kubectl get all -n minio-tenant
kubectl get svc myminio-console -n minio-tenant -o jsonpath="{.status.loadBalancer.ingress[0].ip}:{.spec.ports[0].port}"
# https minio/minio123
```

## kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 70.0.0 -n loki -f values-kube-prometheus-stack.yaml

# kube-prometheus-stack has been installed. Check its status by running:
kubectl --namespace loki get pods -l "release=kube-prometheus-stack"

# Get Grafana 'admin' user password by running:
kubectl --namespace loki get secrets kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Access Grafana local instance:
export POD_NAME=$(kubectl --namespace loki get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=kube-prometheus-stack" -oname)
kubectl --namespace loki port-forward $POD_NAME 3000

# TODO: https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/config-other-methods/helm-operator-migration/
```