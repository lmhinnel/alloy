## capi

```bash
clusterctl init --infrastructure kubevirt --control-plane kamaji


# gen yaml
KUBEVIRT_PROVIDER=k8s-1.30.2 TENANT_CLUSTER_KUBERNETES_VERSION=v1.30.2 NODE_VM_IMAGE_TEMPLATE=quay.io/capk/ubuntu-2204-container-disk:v1.30.1 CRI_PATH="/var/run/containerd/containerd.sock" CLUSTER_NAME=my-cluster NAMESPACE=default clusterctl generate cluster my-cluster --kubernetes-version v1.30.2 > my-cluster.yaml

k apply -f my-cluster.yaml

kubectl apply -f https://github.com/clastix/cluster-api-control-plane-provider-kamaji/releases/download/v0.14.2/control-plane-components.yaml
kubectl apply -f https://raw.githubusercontent.com/clastix/kamaji/refs/tags/v1.0.0/charts/kamaji/crds/datastore.yaml
kubectl apply -f https://raw.githubusercontent.com/clastix/kamaji/refs/tags/v1.0.0/charts/kamaji/crds/tenantcontrolplane.yaml
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api-provider-kubevirt/releases/download/v0.1.10/infrastructure-components.yaml

clusterctl get kubeconfig my-cluster > my-cluster.kubeconfig

```

## metallb

```bash
kubectl apply -f "https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml"
kubectl wait pods -n metallb-system -l app=metallb,component=controller --for=condition=Ready --timeout=10m
kubectl wait pods -n metallb-system -l app=metallb,component=speaker --for=condition=Ready --timeout=2m

GW_IP=$(docker network inspect -f '{{range .IPAM.Config}}{{.Gateway}}{{end}}' kind)
NET_IP=$(echo ${GW_IP} | sed -E 's|^([0-9]+\.[0-9]+)\..*$|\1|g')
sed "s|__NET_IP__|$NET_IP|g" metallb-config-template.yaml > metallb-config.yaml
kubectl apply -f metallb-config.yaml

# deploy required CRDs
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/v1.5.0/kubevirt-operator.yaml"
# deploy the KubeVirt custom resource
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/v1.5.0/kubevirt-cr.yaml"
kubectl wait -n kubevirt kv kubevirt --for=condition=Available --timeout=10m

# IDK: https://serverfault.com/questions/1137211/failed-to-create-fsnotify-watcher-too-many-open-files

```

## otel
```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm upgrade --install my-opentelemetry-collector-deployment open-telemetry/opentelemetry-collector -n default --version 0.122.4 --values values-otel-collector-deployment.yaml
helm upgrade --install my-opentelemetry-collector-daemonset open-telemetry/opentelemetry-collector -n default --version 0.122.4 --values values-otel-collector-daemonset.yaml

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# prometheus
# prometheus-server.default.svc.cluster.local
# prometheus-prometheus-pushgateway.default.svc.cluster.local
# alertmanager:9093|prometheus:9090|prometheus-pushgateway:9091
helm upgrade --install prometheus prometheus-community/prometheus -n "default" --version 27.5.1 --values values-prometheus.yaml 
kubectl --namespace default port-forward $(kubectl get pods --namespace default -l "app.kubernetes.io/name=alertmanager,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}") 9093
kubectl --namespace default port-forward $(kubectl get pods --namespace default -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}") 9090
kubectl --namespace default port-forward $(kubectl get pods --namespace default -l "app.kubernetes.io/name=prometheus-pushgateway,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}") 9091

# loki
helm upgrade --install loki grafana/loki --version 6.27.0 -n default -f values-loki.yaml
kubectl port-forward --namespace default svc/loki-gateway 3100:80
curl --location 'http://127.0.0.1:3100/loki/api/v1/push' --header 'Content-Type: application/json' --data '{"streams":[{"stream":{"job":"test"},"values":[["1746359511000000000","fizzbuzz"]]}]}'
curl --location 'http://127.0.0.1:3100/loki/api/v1/query_range' --header 'Content-Type: application/x-www-form-urlencoded' --data-urlencode 'query={job="test"}'

# grafana
# grafana.default.svc.cluster.local
helm upgrade --install grafana grafana/grafana --version 8.10.1 -n default -f values-grafana.yaml

# rbac
k apply -f rbac-otel-collector.yaml
```