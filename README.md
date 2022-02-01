# Monitoring your Kubernetes stack on GKE

## Install Prometheus
> Note we are using the [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) stack. Please refer to this doc for reference.

### Install dependencies
> On MacOS you can use `brew` to install these packages

1. [jsonnet-bundler](https://github.com/jsonnet-bundler/jsonnet-bundler)

2. [jsonnet](https://github.com/google/jsonnet)

3. [go](https://go.dev/doc/install)
```
  go install github.com/brancz/gojsontoyaml@latest
  go install github.com/google/go-jsonnet/cmd/jsonnet@latest
```
4. wget


### Create the install directory
```
mkdir kube-prometheus
```
```
cd kube-prometheus
```

### Initialize jb and install kube-prometheus
```
jb init
```
```
jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@main
```

### Pull the example.jsonnet and build.sh files
```
wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/example.jsonnet -O example.jsonnet
```
```
wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/main/build.sh -O build.sh
```

### Update jb
```
jb update
```

### Make build.sh executable
```
chmod +x build.sh
```

> Note: If you need to update GOPATH in the build.sh file edit line as below:
> 
> ```jsonnet -J vendor -m manifests "${1-example.jsonnet}" | xargs -I{} sh -c 'cat {} | $(go env GOPATH)/bin/gojsontoyaml > {}.yaml; rm -f {}' -- {}```
>

### Build the customization file

```
vim memsql.jsonnet
```
```
local kp = (import 'kube-prometheus/main.libsonnet') + {
  values+:: {
    common+: {
      namespace: 'monitoring',
    },
    prometheus+:: {
      namespaces+: ['singlestore'],
    },
  },
  memsqlMetrics: {
    serviceMonitorMyNamespace: {
      apiVersion: 'monitoring.coreos.com/v1',
      kind: 'ServiceMonitor',
      metadata: {
        name: 'memsql-metrics',
        namespace: 'singlestore',
      },
      spec: {
        endpoints: [
          {
            path: '/metrics',
            targetPort: 'metrics',
          },
        ],
        jobLabel: 'app.kubernetes.io/name',
        selector: {
          matchLabels: {
            'app.kubernetes.io/name': 'memsql-cluster',
            'app.kubernetes.io/component': 'cluster',
          },
        },
      },
    },
  },
  memsqlClusterMetrics: {
    serviceMonitorMyNamespace: {
      apiVersion: 'monitoring.coreos.com/v1',
      kind: 'ServiceMonitor',
      metadata: {
        name: 'memsql-cluster-metrics',
        namespace: 'singlestore',
      },
      spec: {
        endpoints: [
          {
            path: '/cluster-metrics',
            targetPort: 'metrics',
          },
        ],
        jobLabel: 'app.kubernetes.io/name',
        selector: {
          matchLabels: {
            'app.kubernetes.io/name': 'memsql-cluster',
            'app.kubernetes.io/component': 'master',
          },
        },
      },
    },
  },
};

{ 'setup/0namespace-namespace': kp.kubePrometheus.namespace } +
{
  ['setup/prometheus-operator-' + name]: kp.prometheusOperator[name]
  for name in std.filter((function(name) name != 'serviceMonitor' && name != 'prometheusRule'), std.objectFields(kp.prometheusOperator))
} +
// serviceMonitor and prometheusRule are separated so that they can be created after the CRDs are ready
{ 'prometheus-operator-serviceMonitor': kp.prometheusOperator.serviceMonitor } +
{ 'prometheus-operator-prometheusRule': kp.prometheusOperator.prometheusRule } +
{ 'kube-prometheus-prometheusRule': kp.kubePrometheus.prometheusRule } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['blackbox-exporter-' + name]: kp.blackboxExporter[name] for name in std.objectFields(kp.blackboxExporter) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['kubernetes-' + name]: kp.kubernetesControlPlane[name] for name in std.objectFields(kp.kubernetesControlPlane) }
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['prometheus-adapter-' + name]: kp.prometheusAdapter[name] for name in std.objectFields(kp.prometheusAdapter) } +
{ ['memsql-metrics-' + name]: kp.memsqlMetrics[name] for name in std.objectFields(kp.memsqlMetrics) } +
{ ['memsql-cluster-metrics-' + name]: kp.memsqlClusterMetrics[name] for name in std.objectFields(kp.memsqlClusterMetrics) }
```
> Note: Only change the namespace inside of this file where it says `singlestore` to the namespace that you have SingleStore deployed.


### Apply the build to the manifest files
```
./build.sh memsql.jsonnet
```

### Create the kubernetes objects
```
kubectl apply --server-side -f manifests/setup
```
```
kubectl apply -f manifests/
```

### Access Prometheus locally
```
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
```

## Add Loki + FluentBit with a persistant volume claim
Prereqs: Install [Helm](https://helm.sh/docs/intro/install/)

```
helm repo add grafana https://grafana.github.io/helm-charts
```
```
helm repo update
```
```
helm upgrade --install loki --namespace=monitoring grafana/loki-stack --set fluent-bit.enabled=true,promtail.enabled=false,loki.persistence.enabled=true,loki.persistence.storageClassName=standard,loki.persistence.size=5Gi
```
> Note: you can change the size of the volume claim in the last command

## Add the Loki data source to Grafana

### Access Grafana locally
```
kubectl --namespace monitoring port-forward svc/grafana 3000
``` 
### Go to Configuration > Data Sources
<img width="252" alt="Screen Shot 2022-02-01 at 2 35 32 PM" src="https://user-images.githubusercontent.com/16610646/152038532-d4a3b68a-da39-4b94-b45c-a31401386b93.png">

### Click add data source and then type in 'Loki'
<img width="190" alt="Screen Shot 2022-02-01 at 2 36 44 PM" src="https://user-images.githubusercontent.com/16610646/152038758-e6f8d6fc-d802-4a22-a069-16c5dc95d63e.png">

### Type in the Loki URL
<img width="690" alt="Screen Shot 2022-02-01 at 2 37 45 PM" src="https://user-images.githubusercontent.com/16610646/152038840-e74cef93-b044-4edd-bca4-8dcc443f9787.png">

### Click 'Save & Test' and you should see the message below
<img width="570" alt="Screen Shot 2022-02-01 at 2 38 21 PM" src="https://user-images.githubusercontent.com/16610646/152038985-3d2c9fcf-92cd-4867-9e64-ec516af68d2e.png">

### Now you can Explore SingleStore logs through Grafana
<img width="237" alt="Screen Shot 2022-02-01 at 2 42 01 PM" src="https://user-images.githubusercontent.com/16610646/152039628-dfbc60e1-5add-4d1c-95d4-fff0139572cf.png">

### Expand the Log Browser in order to have a helpful UI
<img width="128" alt="Screen Shot 2022-02-01 at 2 42 27 PM" src="https://user-images.githubusercontent.com/16610646/152039715-46109465-a4a0-403a-922a-af8e363202bd.png">

### Access SingleStore logs
<img width="1368" alt="Screen Shot 2022-02-01 at 2 42 46 PM" src="https://user-images.githubusercontent.com/16610646/152039764-446028e1-9fb8-4445-ab33-0d646031d550.png">

> Kube-prometheus has also added 24 dashboards to Grafana to get you started with dashboard monitoring

### Click on Dashboards > Browse
<img width="247" alt="Screen Shot 2022-02-01 at 2 45 32 PM" src="https://user-images.githubusercontent.com/16610646/152040633-dfc416c5-ae0a-4036-b62e-b43243c590f3.png">

### Expand the Default folder
<img width="1373" alt="Screen Shot 2022-02-01 at 2 45 53 PM" src="https://user-images.githubusercontent.com/16610646/152040686-5a7c0dea-419a-45d9-9eab-fb5bc31b805b.png">

### Click on a dashboard to try it out! 

> Now we have installed a monitoring stack consisting of Prometheus, Grafana, Loki, and FluentBit
