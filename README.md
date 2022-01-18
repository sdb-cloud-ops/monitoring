# Prometheus on GCP

> Note we are using the [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus) stack. Please refer to this doc for reference.

### Install dependencies
```
jsonnet-bundler
```
```
jsonnet
```
```
go
go install github.com/brancz/gojsontoyaml@latest
go install github.com/google/go-jsonnet/cmd/jsonnet@latest
```
```
wget
```

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

### Access Grafana locally
```
kubectl --namespace monitoring port-forward svc/grafana 3000
```

#### Reference

> To expose behind ingress:
> https://github.com/prometheus-operator/kube-prometheus/blob/main/docs/customizations/exposing-prometheus-alertmanager-grafana-ingress.md
