# Istio and Cloud Service Mesh on Google Kubernetes Engine (GKE)

Istio is an open-source service mesh that provides a uniform way to secure, connect, and monitor microservices. It helps manage the complexity of microservices deployments by providing behavioral insights and operational control over the service mesh.

This project demonstrates two approaches to implementing a service mesh on GKE:

1. **Cloud Service Mesh (ASM)** – A managed Istio-based solution provided by Google Cloud.
2. **Open Source Istio** – A raw installation of the open-source Istio tool, with full manual control over configuration and observability.

The project includes screenshots, manifests, and instructions for observability tools (Prometheus, Grafana, Jaeger, Kiali), distributed tracing, and more.

## Part 1: Cloud Service Mesh (ASM)

This section shows how to use **Google Cloud’s managed Istio offering** — Cloud Service Mesh (ASM) — to install a secure and observable service mesh on GKE.

### What’s Covered

- Creating a GKE cluster for ASM
- Installing and configuring ASM with recommended settings
- Deploying sample application
- Enabling external access via **Istio Ingress Gateway**
- Using the **Google Cloud Console** to:
  - Explore service-to-service topology
  - View metrics and logs via the **Service Mesh tab**
  - Expand/collapse Bookinfo microservices relationships
- Captured screenshots of:
  - **List View** in Service Mesh tab
 
![list_view_slo](https://github.com/user-attachments/assets/ba17eb78-1978-4db7-acb6-1113027b86bf)

 - **Topology View** with connected Pods
  
![topology_view](https://github.com/user-attachments/assets/47268163-28af-4d00-a6fb-45280824d177)



## Part 2: Raw Istio (Open Source)

This section documents the installation and configuration of open-source Istio from scratch, without Google’s managed ASM solution.

### You will find:

- IstioOperator deployment via script
- Automatic sidecar injection configuration
- Deploy the sample application
- Prometheus and Grafana setup for metrics
- Envoy dashboard access
- Distributed tracing using Jaeger
- Kiali console integration

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

![gke_created](https://github.com/user-attachments/assets/bc328c84-acd7-42b0-a735-81a1591e8839)

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

![istio_installed](https://github.com/user-attachments/assets/2604e825-2145-4b18-88d7-b7a459f26411)

### Deploy the IstioOperator resource

```sh
istioctl install -f demo-profile.yaml
```

![IstioOperator_resource](https://github.com/user-attachments/assets/fce27f8a-745e-4508-9f75-ffe205e13c15)

Alternatively, use the `--set` flag:

```sh
istioctl install --set profile=demo
```

### Enable automatic sidecar injection

```sh
kubectl label namespace default istio-injection=enabled
```

To check that the namespace is labeled, run:

```sh
kubectl get namespace -L istio-injection
```

![auto_sidecar_inject](https://github.com/user-attachments/assets/f828fdeb-ff5c-477d-9f54-f6dd4aed9287)

### Check Istio system pods

```sh
kubectl get po -n istio-system
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

![deploy_apps](https://github.com/user-attachments/assets/b4d79b14-4d51-48e3-8d68-f106efb1a03f)

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

![call_service](https://github.com/user-attachments/assets/8f12a61a-01ce-443f-afd5-9eab6da1ba16)

### Query the scrape endpoint

```sh
kubectl exec $PRODUCTPAGE_POD -c istio-proxy -- curl localhost:15090/stats/prometheus
```

### Access the Envoy dashboard

```sh
istioctl dashboard envoy deploy/productpage-v1.default
```

![envoy_dashboard](https://github.com/user-attachments/assets/c4d66797-4221-4d0e-8e54-6b8ce34cc0a4)

Prometheus stats tab

![stats_prometheus](https://github.com/user-attachments/assets/0bf2a7b7-66b4-4fd7-afe5-10b17ac89960)

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

![prometheus_dashboard](https://github.com/user-attachments/assets/daf2136b-ad41-4faf-82a0-d24670af52a5)

### Prometheus Queries

To find out the subset of requests that returned an HTTP 200 response code:

```sh
istio_requests_total{response_code="200"}
```

To locate the call from the sleep pod to the productpage service:

```sh
istio_requests_total{source_app="sleep",destination_app="productpage"}
```

![istio_requests_total](https://github.com/user-attachments/assets/640d1cad-263c-46c2-91d8-4969e75415cb)

To query the rate of incoming requests over the last 5 minutes:

```sh
rate(istio_requests_total{destination_app="productpage"}[5m])
```

![istio_rate_of_incoming_requests](https://github.com/user-attachments/assets/02522f83-24e4-4f76-803d-20cbf24183bb)

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

![grafana_six_dashboards](https://github.com/user-attachments/assets/f2e41511-7f8e-430d-bc38-f1a94309e59e)

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

![grafana_istio_mesh_dashboard](https://github.com/user-attachments/assets/a3f7c72f-948f-40d2-8f95-d4d57dd899a4)

### Istio Service Dashboard

Visit the Istio Service Dashboard, select the service named `productpage.default.svc.cluster.local`, and expand the General panel.

![service_workloads](https://github.com/user-attachments/assets/70fa5531-6804-405a-af3b-8f96ffea60a9)

![client_workloads](https://github.com/user-attachments/assets/13b2c031-f507-4952-9fae-07783177b20f)

![general](https://github.com/user-attachments/assets/96ea83dc-07f5-4f81-902f-d068112d8eb2)

### Istio Workload Dashboard

Navigate to the Workload Dashboard, select the `reviews-v3` workload, and inspect its vitals.

![inbound_workloads](https://github.com/user-attachments/assets/ed2eca75-483a-4b10-bfc6-7ba067704372)

![outbound_workloads](https://github.com/user-attachments/assets/c69a7192-364b-4ba4-b1a0-1b05e3100841)

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

![kiali_service_rating](https://github.com/user-attachments/assets/f9db1d0d-0316-4dbc-a8a9-a94e5a52995a)

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
