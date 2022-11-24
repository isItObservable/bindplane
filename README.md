# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Episode : What is OpAmp & What is BindPlane 
This repository contains the files utilized during the tutorial presented in the dedicated IsItObservable episode related to Bindplane.
<p align="center"><img src="/image/bindplane.png" width="40%" alt="bindplane Logo" /></p>

What you will learn
* How to use the [BindPlane](https://github.com/observIQ/bindplane-op)

This repository showcase the usage of BindPlane  with :
* The Otel-demo
* The OpenTelemetry Operator
* Nginx ingress controller
* Prometheus
* Grafana
* Dynatrace
* KubeCost

We will send the Kubecost metrics and Traces to dynatrace using BindPlane.

## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm


## Deployment Steps in GCP

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
    cloudtrace.googleapis.com \
    clouddebugger.googleapis.com \
    cloudprofiler.googleapis.com \
    --project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=europe-west3-a
NAME=isitobservable-bindplane
gcloud container clusters create "${NAME}" \
 --zone ${ZONE} --machine-type=e2-standard-2 --num-nodes=4
```


## Getting started
### Dynatrace Tenant
#### 1. Dynatrace Tenant - start a trial
If you don't have any Dyntrace tenant , then i suggest to create a trial using the following link : [Dynatrace Trial](https://bit.ly/3KxWDvY)
Once you have your Tenant save the Dynatrace (including https) tenant URL in the variable `DT_TENANT_URL` (for example : https://dedededfrf.live.dynatrace.com)
```
DT_TENANT_URL=<YOUR TENANT URL>
```


#### 2. Create the Dynatrace API Tokens
The dynatrace operator will require to have one token:
* Token to ingest metrics and Traces


##### Token to ingest data
Create a Dynatrace token with the following scope:
* ingest metrics
* ingest events
* ingest OpenTelemetry traces
* ingest Logs
* Data ingest, e.g.: metrics and events
<p align="center"><img src="/image/data_ingest.png" width="40%" alt="data token" /></p>
Save the value of the token . We will use it later to store in a k8S secret

```
DATA_INGEST_TOKEN=<YOUR TOKEN VALUE>
```

### 3.Clone the Github Repository
```
https://github.com/isItObservable/bindplane
cd bindplane
```
### 4.Deploy most of the components
The application will deploy the otel demo v1.0.0
```
chmod 777 deployment.sh
./deployment.sh  --clustername ${NAME}
```


### 5.Configure KubeCost

#### Add the additional scraping config
We need to edit the Prometheus settings by adding the additional scrape configuration, edit Prometheus with the following command :
```
kubectl get Prometheus
```

here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```

We will need to add an extra property in the configuration object :
```
additionalScrapeConfigs:
  name: addtional-scrape-configs
  key: additionnalscrapeconfig.yaml
```

so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
#### Connect kubecost to prometheus
```
kubectl edit cm kubecost-cost-analyzer  -n kubecost
```
make sure all the configuration are correct :
```
apiVersion: v1
data:
kubecost-token: aGVucmlrLnJleGVkQGR5bmF0cmFjZS5jb20=xm343yadf98
prometheus-alertmanager-endpoint: http://prometheus-kube-prometheus-alertmanager.default.svc:9093
prometheus-server-endpoint: http://prometheus-kube-prometheus-prometheus.default.svc:9090
kind: ConfigMap
metadata:
annotations:
meta.helm.sh/release-name: kubecost
meta.helm.sh/release-namespace: kubecost
labels:
app: cost-analyzer
app.kubernetes.io/instance: kubecost
app.kubernetes.io/managed-by: Helm
app.kubernetes.io/name: cost-analyzer
helm.sh/chart: cost-analyzer-1.92.0
name: kubecost-cost-analyzer
namespace: kubecost
```
#### Connect kubecost to Grafana
```
kubectl edit cm nginx-conf -n kubecost
```
update the grafana upstream url :
```
upstream grafana {
server prometheus-grafana.default.svc;
}
```
#### Edit the Kubecost ingress rule
```
kubectl edit ingress kubecost-cost-analyzer  -n kubecost
```
make sure to add the following annotation : kubernetes.io/ingress.class: nginx
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
annotations:
ingress.kubernetes.io/backends: '{"k8s-be-30348--560d80e95126adbd":"UNHEALTHY","k8s-be-31223--560d80e95126adbd":"HEALTHY"}'
ingress.kubernetes.io/forwarding-rule: k8s2-fr-xw9dp7bo-kubecost-kubecost-cost-analyzer-5laj3bq5
ingress.kubernetes.io/target-proxy: k8s2-tp-xw9dp7bo-kubecost-kubecost-cost-analyzer-5laj3bq5
ingress.kubernetes.io/url-map: k8s2-um-xw9dp7bo-kubecost-kubecost-cost-analyzer-5laj3bq5
kubernetes.io/ingress.class: nginx
meta.helm.sh/release-name: kubecost
meta.helm.sh/release-namespace: kubecost
```

### 6. Deploy BindPlane

#### Create a secret with 
```
BINDPLANE_PASS=<YOUR PASSWORD>
BINDPLANE_CONFIG_SECRET_KEY=6b130308-9189-47cf-905a-b9abf4b6641e
BINDPLANE_CONFIG_SESSIONS_SECRET=b399bc86-db0d-4075-a477-3c3ce0d57e43
kubectl create ns bindplane
kubectl create secret generic bindplane --from-literal='BINDPLANE_PASS=$BINDPLANE_PASS' --from-literal="BINDPLANE_CONFIG_SECRET_KEY=$BINDPLANE_CONFIG_SECRET_KEY" --from-literal="BINDPLANE_CONFIG_SESSIONS_SECRET=$BINDPLANE_CONFIG_SESSIONS_SECRET" -n bindplane
```

#### Deploy Bindplane Server
```
kubectl apply -f blindplane/bindplane.yaml -n bindplane
```
#### Deploy the bindplane Collector
```
kubectl apply -f blindplane/daemonset.yaml -n bindplane
```

### 7. Deploy The application
```
kubectl create ns otel-demo
kubectl apply -f kubernetes-manifests/openTelemetry-sidecar.yaml -n otel-demo
kubectl apply -f kubernetes-manifests/K8sdemo.yaml -n otel-demo
```

### 8. Install the Blindplane cli
Follow the following [documentation](https://github.com/observIQ/bindplane-op/blob/main/docs/install.md#client) to install the bindplanectl on your local machine.

#### Configure the bindplane CLI
```
bindplanectl profile set observiq_tutorial --username admin --password $BINDPLANE_PASS --server-url http://blindplane.$IP.nip.io
```
#### Add a new custom destination in Bindplane
To send the Traces in Dynatrace, we can customize the Destination settings in bindplane.
`bindplane/otlp_dynatrace.yaml` is a new DestinationType that Bindplane can use .

```
apiVersion: bindplane.observiq.com/v1
kind: DestinationType
metadata:
  name: otlp_dynatrace
  displayName: OpenTelemetry Dyantrace (OTLP http)
  icon: /icons/destinations/dynatrace.svg
  description: Send traces to dynatrace endpoint.
spec:
  parameters:
    - name: dynatrace_endpoint
      label: Dynatrace url
      description: Dynatradce url ( example https://dded.live.dynatrace.com) .
      type: string
      default: ""
      required: true

    - name: apitoken
      label: Access Token
      description:  Access Token that is restricted to 'OpenTelemetry Trace inges' scope. Required if Endpoint is specified
      type: string
      required: true
      default: ""
      documentation:
        - text: Read more
          url: https://www.dynatrace.com/support/help/extend-dynatrace/opentelemetry/opentelemetry-traces/opentelemetry-trace-ingest-api


  traces:
    exporters: |
      - otlphttp:
          endpoint: {{ .dynatrace_endpoint }}
          headers:
            Authorization: "Api-Token {{ .apitoken }}"


    processors: |
      - batch:
          send_batch_max_size: 1000
          send_batch_size: 1000
          timeout: 30s
```
This destinationType object precise the parameter that would be use , and how to create the right exporter in our pipeline.
We can also add a predefined processor for this exporter.

Let's deploy this new destination :
```
bindplanectl apply -f bindplane/otlp_dynatrace.yaml
```


### 8. Configure our Pipeline in BindPlane
#### Create a new configuration
Let's create a new configuration ( collector pipeline in Bindplane) using :
- as receivers ( source ) :
    - Prometheus for metrics
    - Otlp for metrics and traces
    
- as exporters ( Destination ) :
    - the new destination : otp-dynatrace for traces
    - dynatrace-metrics for metrics

In the configuration you will need to precise the host for :
- Prometheus
    let's collect he metrics provided by kubecost : `kubecost-cost-analyzer.kubecost.svc:9003`
  
- Dynatrace-metrics :
    metric ingest endpoint : `$DT_TENANT_URL/api/v2/metrics/ingest` 
    Api token : `DATA_INGEST_TOKEN`
  
- otlp-dynatrace:
    Dynatrace url : `$DT_TENANT_URL `
    Access token : `DATA_INGEST_TOKEN`
  
#### Select your agent to deploy this configuration

In the agents section, press on the `+ button` and select the observiq-collector instances previously deployed
