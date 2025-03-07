# Istio for Microservices Security and Monitoring

## Overview

Istio is an open-source service mesh that provides a uniform way to secure, connect, and monitor microservices. It helps manage the complexity of microservices deployments by providing behavioral insights and operational control over the service mesh.

## Prerequisites (initial setup)

### Create GKE Cluster

```sh
gcloud container clusters create my-istio-cluster \
 --cluster-version latest \
 --num-nodes "3" \
 --network "default" \
 --zone us-central1-b
```

Cluster created:

```
NAME: my-istio-cluster
LOCATION: us-central1-b
MASTER_VERSION: 1.32.1-gke.1357001
MASTER_IP: 34.122.19.116
MACHINE_TYPE: e2-medium
NODE_VERSION: 1.32.1-gke.1357001
NUM_NODES: 3
STATUS: RUNNING
```

### Get the credentials for your GKE cluster

```sh
gcloud container clusters get-credentials my-istio-cluster --zone us-central1-b
```

Now `kubectl` is configured to use your GKE cluster:

```sh
kubectl version
```

```
Client Version: v1.32.1
Kustomize Version: v5.5.0
Server Version: v1.32.1-gke.1357001
```

## Install Istio

### Run the download script

```sh
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.21.0 sh -
```

The Istio release is downloaded and unpacked to the folder called `istio-1.21.0`.

### Add Istio CLI to PATH

```sh
cd istio-1.21.0
export PATH=$PWD/bin:$PATH
```

To check that the Istio CLI is on the path, run:

```sh
istioctl version
```

```
no ready Istio pods in "istio-system"
1.21.0
```

### Deploy the IstioOperator resource

```sh
istioctl install -f demo-profile.yaml
```

Alternatively, use the `--set` flag:

```sh
istioctl install --set profile=demo
```

### Check the deployed resource

```sh
kubectl get po -n istio-system
```

### Enable automatic sidecar injection

```sh
kubectl label namespace default istio-injection=enabled
```

To check that the namespace is labeled, run:

```sh
kubectl get namespace -L istio-injection
```

(image)

### Check Istio system pods

```sh
kubectl get po -n istio-system
```

```
NAME READY STATUS RESTARTS AGE
istio-egressgateway-5d94fbb578-n9x5t 1/1 Running 0 17h
istio-ingressgateway-6c557996fd-n87lq 1/1 Running 0 17h
istiod-74777ffbbf-57jld 1/1 Running 0 17h
```

### Check Istio version

```sh
istioctl version
```

```
client version: 1.24.2
control plane version: 1.21.0
data plane version: 1.21.0 (2 proxies)
```

### Uninstall Istio

```sh
istioctl uninstall --purge
```

## Istio Observability

### Istio Exposes Workload Metrics

Ensure that Istio is deployed with the demo profile:

```sh
istioctl install --set profile=demo
```

Next, ensure that the default namespace is labeled for sidecar injection:

```sh
kubectl label ns default istio-injection=enabled
```

Verify that the label has been applied:

```sh
kubectl get ns -Listio-injection
```

Navigate to the base directory of your Istio distribution:

```sh
cd ~/istio-1.21.0
```

### Deploy the BookInfo sample application

```sh
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

### Deploy the bundled Sleep sample service

```sh
kubectl apply -f samples/sleep/sleep.yaml
```

result - paste image here

### Make a token HTTP request against the productpage service

First, identify the name of the pod corresponding to the productpage deployment:

```sh
PRODUCTPAGE_POD=$(kubectl get pod -l app=productpage -ojsonpath='{.items[0].metadata.name}')
```

Next, use curl to call the productpage service's main page from the running sleep pod:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage | head
```

screenshoot

### Query the scrape endpoint

```sh
kubectl exec $PRODUCTPAGE_POD -c istio-proxy -- curl localhost:15090/stats/prometheus
```

### Access the Envoy dashboard

```sh
istioctl dashboard envoy deploy/productpage-v1.default
```

image

### Prometheus Metrics

Deploy Prometheus:

```sh
kubectl apply -f samples/addons/prometheus.yaml
```

With Prometheus now running and collecting metrics, send another request to the productpage service:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage | head
```

Expose the Prometheus server's dashboard locally:

```sh
istioctl dashboard prometheus
```

Enter the metric name `istio_requests_total` into the search field and press Execute.

screenshoot

### Prometheus Queries

To find out the subset of requests that returned an HTTP 200 response code:

```sh
istio_requests_total{response_code="200"}
```

To locate the call from the sleep pod to the productpage service:

```sh
istio_requests_total{source_app="sleep",destination_app="productpage"}
```

screenshoot

To query the rate of incoming requests over the last 5 minutes:

```sh
rate(istio_requests_total{destination_app="productpage"}[5m])
```

screenshoot

To query the productpage service every 1-2 seconds:

```sh
while true; do kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage; sleep 1; done
```

## Grafana Dashboards for Istio

Deploy Grafana:

```sh
kubectl apply -f samples/addons/grafana.yaml
```

Launch the Grafana UI:

```sh
istioctl dashboard grafana
```

Navigate to the Istio dashboards in Grafana.

screenshoot

### Send traffic to the productpage service

Capture the name of the sleep pod:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
```

Run the following script to make repeated calls to the productpage endpoint:

```sh
while true; do kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage; sleep 0.3; done
```

### Istio Mesh Dashboard

Navigate to the Istio Mesh Dashboard.

screenshoot

### Istio Service Dashboard

Visit the Istio Service Dashboard, select the service named `productpage.default.svc.cluster.local`, and expand the General panel.

3 screenshoots

### Istio Workload Dashboard

Navigate to the Workload Dashboard, select the `reviews-v3` workload, and inspect its vitals.

two screenshoots

## Distributed Tracing

Deploy Jaeger:

```sh
kubectl apply -f samples/addons/jaeger.yaml
```

Store the name of the sleep pod in an environment variable:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
```

Send requests to the productpage service every 1-2 seconds:

```sh
while true; do kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage; sleep 1; done
```

Open the Jaeger dashboard:

```sh
istioctl dashboard jaeger
```

Select the service `productpage.default` and click Find Traces.

## Kiali Console

Deploy Kiali:

```sh
kubectl apply -f samples/addons/kiali.yaml
```

Store the name of the sleep pod in an environment variable:

```sh
SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
```

Send requests to the productpage service at a 1-2 second interval:

```sh
while true; do kubectl exec $SLEEP_POD -it -- curl productpage:9080/productpage; sleep 1; done
```

Launch the Kiali dashboard:

```sh
istioctl dashboard kiali
```

Select the Graph option from the sidebar and select the default namespace.

screenshoot

Through the Display options, additional information can be overlaid on the graph.

screenshoot

## Security

### Mutual TLS

Istio provides mutual TLS to secure service-to-service communication. This ensures that all communication between services is encrypted and authenticated.

### Authorization Policies

Istio allows you to define authorization policies to control who can access your services. These policies can be based on service identity, request properties, and more.

### Secure Ingress and Egress

Istio provides secure ingress and egress gateways to control traffic entering and leaving the service mesh. This helps in enforcing security policies at the edge of the mesh.

### Security Best Practices

- Regularly update Istio to the latest version.
- Use strong authentication mechanisms.
- Monitor and audit your service mesh for any security vulnerabilities.
