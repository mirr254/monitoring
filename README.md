## Documentation on how to integrate datadog+prometheous with Gluu enterprise edition

### Prerequisites

- Datadog Agent 6.5.0+

## Instructions

- Install Gluu server using one of the following
  - [Kustomize](https://github.com/GluuFederation/enterprise-edition/blob/4.1/pygluu/kubernetes/templates/README.md#install-gluu-using-pygluu-kubernetes-with-kustomize)
  - [Helm](https://github.com/GluuFederation/enterprise-edition/blob/4.1/pygluu/kubernetes/templates/README.md#install-gluu-using-helm)

### Create a namespace

We separate Gluu components from monitoring components/tools by creating a separate namespace for each monitoring tool. In this case since we have 2 monitoring tools, we wil have 3 namespaces - inclusive of Gluu Server namespace.   

For each of the monitoring tool namespace we must assign cluster reader permision to this namespace so that each tool has access to k8s APIs. These files are included in every tools' directory.

## Using kubernetes commands to set up

### Datadog
- Configure RBAC permissions for the datadog agent in `datadog` namespace.
  `kubectl create namespace datadog`

  `kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrole.yaml" -ns datadog `
  `kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/serviceaccount.yaml" -ns datadog `
  `kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrolebinding.yaml" -ns datadog `

- Create a `Secret` containing datadog API key. It will be used in datadog agent daemonset.
 `kubectl create secret generic datadog-secret --from-literal api-key="<API-KEY>" ` !!!NOTE: Don't change the secret name. UNLESS one wishes to change it also in the `DaemonSet`.

- Create a datadog agent with custom metrics and APM logs collection enabled.
 `kubectl apply -f datadog/ `

### prometheus
- Create a namespace for prometheus
  `kubectl create namespace prometheus`

- Use the files in [prometheus](/prometheus) to launch prometheus with datadog Autodiscovery enabled.   
  `kubectl apply -f prometheus/ ` 

- For datadog and prometheus integration, prometheus deployment has the following annotations.
  ```
        annotations:
            ad.datadoghq.com/prometheus.check_names: |
              ["openmetrics"]
            ad.datadoghq.com/prometheus.init_configs: |
              [{}]
            ad.datadoghq.com/prometheus.instances: |
              [
                {
                  "prometheus_url": "http://%%host%%:%%port%%/metrics",
                  "namespace": "monitoring",
                  "metrics": [ {"promhttp_metric_handler_requests_total": "prometheus.handler.requests.total"}]
                }
              ]
  ```

  This allows datadog autodiscovery feature and gets all the metrics that prometheus collects from the cluster.

### kube-state metrics
- Kube state metrics service exposes all the metrics on /metrics URI. Prometheus can scrape all the metrics exposed by kube state metrics. 
- This will be created in the `kube-system` namespace.
  `kubectl apply -f kube-state-metrics/`

- Add the following configuration as part of prometheus job configuration.
  ```
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']
  ```

!!!NOTE: This part has been included in by default in [proth-cm](/prometheus/proth-cm.yaml). If it is not being used, it should be removed. If not it will cause health check error in prometheus targets.

## Using helm

### Prerequisites
- Helm 2.10+
- Tiller

### Datadog
- Helm2 creates a namespace to use automatically when one defines the namespace to deploy in.

- install datadog
  ```
  helm install --name datadog-agent-v1 \
      --set datadog.apiKey=<DataDog API Key> \
      --set datadog.apmEnabled=true \
      --set datadog.logsEnabled=true \
      stable/datadog
      --namespace datadog
 
  ```

### prometheus

- Provided here is a [values' file](/helm/prometheus-values.yaml) that is to be used to override the default installation params for prometheus helm chart. 

- The custom `values.yaml` file above has the annotations needed to integrate datadog with prometheus.

- Install prometheus
  ` helm install --name prometheus --namespace prometheus -f helm/prometheus-values.yaml stable/prometheus `

- One can access the prometheus UI via port-forwarding
  `export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")`
  `kubectl --namespace prometheus port-forward $POD_NAME 9090 `

### Kube-state Metrics for more verbose metrics

- install the chart. kube-state metrics installs its resources in `kube-system` namespace so no need to define the namespace
  `helm2 install --name kube-state stable/kube-state-metrics`

- Add the kube-state metrics job in prometheus serverFiles.
  ```
        - job_name: 'kube-state-metrics'
          static_configs:
            - targets: ['kube-state-kube-state-metrics.default.svc.cluster.local:8080']
  ```

That's it. All the metrics will be exported to datadog `metrics` -> `summary`