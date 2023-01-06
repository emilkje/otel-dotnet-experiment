
# OpenTelemetry Automatic instrumentation

## Motivation

Implement a cluster-wide mechanism to automatically instrument workloads placed inside the cluster. The instrumentation will collect metrics and tracing and export this data to a central collector.

This methodology enables an infrastructure team to take ownership of the application instrumentation without burdening each application development team with cross cutting concerns.

## Prerequisites

* [docker](https://www.docker.com/products/docker-desktop/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)
* [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
* [helm](https://helm.sh/docs/intro/install/)

## Provision a sandbox cluster with kind

```sh
kind create cluster --config ./otel-cluster.yaml
```

### Install cert-manager

The OpenTelemetry controller is dependent on [cert-manager](https://cert-manager.io/). Install cert-manager with and let it create self-signed certs:

```sh
# add repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# install chart and crds
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.10.1 \
  --set installCRDs=true
```

You should see a new namespace `cert-manager` inside the cluster:

```sh
kubectl get namespaces
NAME                 STATUS   AGE
cert-manager         Active   11s
default              Active   2m39s
kube-node-lease      Active   2m40s
kube-public          Active   2m40s
kube-system          Active   2m40s
local-path-storage   Active   2m34s
```

To visualize the data collected by OpenTelemetry we can use Jaeger collector to source the data and visualize it through a GUI

```sh
helm repo add elastic https://helm.elastic.co
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts

helm repo update

# prepare the namespace to place 
# all the workloads related to telemetry
kubectl create namespace otel-system

# install storage backend for jaeger
helm install \
  elasticsearch \
  elastic/elasticsearch \
  --namespace elasticsearch-system \
  --create-namespace \
  --set secret.password=YouShallNotPass

# install jaeger
helm install \
  jaeger \
  jaegertracing/jaeger \
  --namespace otel-system \
```

In a new terminal forward the jaeger UI
```sh
export POD_NAME=$(kubectl get pods --namespace otel-system -l "app.kubernetes.io/instance=jaeger,app.kubernetes.io/component=query" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080/
  kubectl port-forward --namespace otel-system $POD_NAME 8080:16686
```

Now lets install the OpenTelemetry Operator with helm

```sh
# Add repository
helm repo add \
  open-telemetry \
  https://open-telemetry.github.io/opentelemetry-helm-charts

helm repo update

# Install chart
helm install \
  opentelemetry-operator \
  open-telemetry/opentelemetry-operator \
  --namespace otel-system \
  --set admissionWebhooks.certManager.create=true
```

Once the operator is installed and the controller are running we can install a collector and an instrumentation configuration:

```sh
# install collector
kubectl apply -f otel-collector.yaml

# install instrumentation configuration
kubectl apply -f instrumentation.yaml
```

finally deploy the dotnet application to see if it will be instrumented automatically

```sh
kubectl apply -f dotnet-application.yaml