---
title: Builtin Prometheus | Stash
description: Monitor Stash using official Prometheus server
menu:
  product_stash_0.7.0:
    identifier: monitoring-builtin
    name: Builtin Prometheus
    parent: monitoring
    weight: 20
product_name: stash
menu_name: product_stash_0.7.0
---

# Monitoring Stash with builtin Prometheus

This tutorial will show you how to configure [official Prometheus](https://github.com/prometheus/prometheus) server to monitor Stash backup and recovery operations as well as Stash operator.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://github.com/kubernetes/minikube).

## Enable Monitoring in Stash

Enable Prometheus monitoring using `builtin` agent while installing Stash. To know details about how to enable monitoring see [here](/docs/guides/monitoring/overview.md#how-to-enable-monitoring).

Here, we are going to enable monitoring for both `backup & recovery` and `opeartor` metrics.

```console
$ curl -fsSL https://raw.githubusercontent.com/appscode/stash/0.7.0/hack/deploy/stash.sh \
    | bash -s -- --monitoring-agent=builtin --monitoring-backup=true --monitoring-operator=true
```

This will add some annotation to `stash-operator` service. Prometheus server will scrap metrics using those annotations. Let's check which annotations are added to the service,

```yaml
$ kubectl get service -n kube-system stash-operator -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"stash"},"name":"stash-operator","namespace":"kube-system"},"spec":{"ports":[{"name":"admission","port":443,"targetPort":8443},{"name":"pushgateway","port":56789,"targetPort":56789}],"selector":{"app":"stash"}}}
    prometheus.io/operator_path: /metrics
    prometheus.io/operator_port: "8443"
    prometheus.io/operator_scheme: https
    prometheus.io/pushgateway_path: /metrics
    prometheus.io/pushgateway_port: "56789"
    prometheus.io/pushgateway_scheme: http
    prometheus.io/scrap: "true"
  creationTimestamp: 2018-11-07T04:10:26Z
  labels:
    app: stash
  name: stash-operator
  namespace: kube-system
  resourceVersion: "1649"
  selfLink: /api/v1/namespaces/kube-system/services/stash-operator
  uid: 0e73664a-e243-11e8-a768-080027767ca3
spec:
  clusterIP: 10.105.200.228
  ports:
  - name: admission
    port: 443
    protocol: TCP
    targetPort: 8443
  - name: pushgateway
    port: 56789
    protocol: TCP
    targetPort: 56789
  selector:
    app: stash
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

Here, `prometheus.io/scrap: "true"` annotation indicates that Prometheus should scrap metrics for this service.

These three annotations are backup and recovery metrics specific annotation. They provides information about `pushgateway` endpoints which provides those metrics.

```ini
prometheus.io/pushgateway_path: /metrics
prometheus.io/pushgateway_port: "56789"
prometheus.io/pushgateway_scheme: http
```

Below, three annotations are operator metrics specific annotation. They provides information about how to collect those metrics from `admission` endpoint.

```ini
prometheus.io/operator_path: /metrics
prometheus.io/operator_port: "8443"
prometheus.io/operator_scheme: https
```

Now, we are ready to configure our Prometheus server to scrap those metrics.

## Deploy Prometheus Server

We have deployed Stash in `kube-system` namespace. Stash exports operator metrics in TLS secure `admission` endpoint. So, Prometheus server need to provide certificate while scrapping metrics from this endpoint. Stash creates a secret named `stash-apiserver-certs`  with this certificate in the same namespace as Stash operator. We have to mount this secret in Prometheus deployment. So, we have to deploy Prometheus in `kube-system` namespace.

>If you want to deploy Prometheus in a different namespace, you have to make a copy of this secret to that namespace.

**Create RBAC:**

If you are using a RBAC enabled cluster, you have to give necessary RBAC permissions for Prometheus. Let's create necessary RBAC stuffs for Prometheus,

```console
$ kubectl apply -f ./docs/examples/monitoring/builtin/prom-rbac.yaml
clusterrole.rbac.authorization.k8s.io/stash-prometheus-server created
serviceaccount/stash-prometheus-server created
clusterrolebinding.rbac.authorization.k8s.io/stash-prometheus-server created
```

**Create ConfigMap:**

Now, create a ConfigMap with necessary scrapping configuration. Bellow, the YAML of ConfigMap that we are going to create in this tutorial.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: stash-prometheus-server-conf
  labels:
    name: stash-prometheus-server-conf
  namespace: kube-system
data:
  prometheus.yml: |-
    global:
      scrape_interval: 30s
      scrape_timeout: 10s
      evaluation_interval: 30s
    scrape_configs:
    - job_name: stash-pushgateway
      scrape_interval: 30s
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: http
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrap]
        regex: true
        action: keep
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        regex: pushgateway
        action: keep
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_pushgateway_path]
        regex: (.+)
        target_label: __metrics_path__
        action: replace
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_pushgateway_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_pushgateway_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - source_labels: [__meta_kubernetes_namespace]
        separator: ;
        regex: (.*)
        target_label: namespace
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_service_name]
        separator: ;
        regex: (.*)
        target_label: service
        replacement: $1
        action: replace
    - job_name: stash-operator
      scrape_interval: 30s
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: https
      kubernetes_sd_configs:
      - role: endpoints
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      tls_config:
        ca_file: /etc/prometheus/secret/stash-apiserver-cert/tls.crt
        server_name: stash-operator.kube-system.svc
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrap]
        regex: true
        action: keep
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        regex: admission
        action: keep
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_operator_path]
        regex: (.+)
        target_label: __metrics_path__
        action: replace
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_operator_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_operator_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - source_labels: [__meta_kubernetes_namespace]
        separator: ;
        regex: (.*)
        target_label: namespace
        replacement: $1
        action: replace
      - source_labels: [__meta_kubernetes_service_name]
        separator: ;
        regex: (.*)
        target_label: service
        replacement: $1
        action: replace
```

Here, we have two scrapping job. One is `stash-pushgateway` that scraps backup and recovery metrics and another is `stash-operator` which scraps operator metrics.

Look at the `tls_config` field of `stash-operator` job. We have provided certificate file through `ca_file` field. This certificate comes from `stash-apiserver-cert` that we are going to mount in Prometheus deployment. Here, `server_name` is used to verify hostname. In our case, the certificate is valid for hostname `server` and `stash-operator.kube-system.svc`.

Also note that, we have provided a bearer-token file through `bearer_token_file` field. This file is token for `stash-prometheus-server` serviceaccount that we have created while creating RBAC stuffs. This is required for authorizing Prometheus to Stash API Server.

Let's create the ConfigMap we have shown above,

```console
$ kubectl apply -f curl -fsSL https://raw.githubusercontent.com/appscode/stash/0.7.0/docs/examples/monitoring/builtin/prom-config.yaml
configmap/stash-prometheus-server-conf created
```

**Deploy Prometheus:**

Now, we are ready to deploy Prometheus server. YAML for the deployment that we are going to create for Prometheus is shown below.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stash-prometheus-server
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stash-prometheus-server
  template:
    metadata:
      labels:
        app: stash-prometheus-server
    spec:
      serviceAccountName: stash-prometheus-server
      containers:
      - name: prometheus
        image: prom/prometheus:v2.4.3
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/"
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prometheus-config-volume
          mountPath: /etc/prometheus/
        - name: prometheus-storage-volume
          mountPath: /prometheus/
        - name: stash-apiserver-cert
          mountPath: /etc/prometheus/secret/stash-apiserver-cert
      volumes:
      - name: prometheus-config-volume
        configMap:
          defaultMode: 420
          name: stash-prometheus-server-conf
      - name: prometheus-storage-volume
        emptyDir: {}
      - name: stash-apiserver-cert
        secret:
          defaultMode: 420
          secretName: stash-apiserver-cert
```

Notice that, we have mounted `stash-apiserver-cert` secret as a volume at `/etc/prometheus/secret/stash-apiserver-cert` directory.

Now, let's create the deployment,

```console
$ kubectl apply -f curl -fsSL https://raw.githubusercontent.com/appscode/stash/0.7.0/docs/examples/monitoring/builtin/prom-deployment.yaml
deployment.apps/stash-prometheus-server created
```


### Verify Monitoring Metrics

Prometheus server is running on port `9090`. We will use [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to access Prometheus dashboard. Run following command on a separate terminal,

```console
$ kubectl port-forward -n kube-system stash-prometheus-server-9ddbf79b6-8l6hk 9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

Now, we can access the dashboard at `localhost:9090`. Open `localhost:9090` in your browser. You should see `pushgateway` and `admission` endpoints of `stash-operator` service as targets.

<p align="center">
  <img alt="Prometheus Target"  src="/docs/images/monitoring/prom-builtin-target.png", style="padding:10px">
</p>

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl delete clusterrole stash-prometheus-server
kubectl delete clusterrolebinding stash-prometheus-server
kubectl delete serviceaccount/stash-prometheus-server -n kube-system
kubectl delete configmap/stash-prometheus-server-conf -n kube-system
kubectl delete deployment stash-prometheus-server -n kube-system
```

To uninstall Stash follow this [guide](/docs/setup/uninstall.md).

## Next Steps

- Learn how monitoring in Stash works from [here](/docs/guides/monitoring/overview.md).
- Learn how to monitor Stash using CoreOS Prometheus operator from [here](/docs/guides/monitoring/coreos.md).
- Learn how to use Grafana dashboard to visualize monitoring data from [here](/docs/guides/monitoring/grafana.md).