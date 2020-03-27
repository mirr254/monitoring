## Documentation on how to integrate datadog+prometheous with Gluu enterprise edition

### Prerequisites

- Datadog Agent 6.5.0+

## Instructions

### Create a namespace

We separate Gluu components from monitoring components/tools by creating a separate namespace for each monitoring tool. In this case since we have 2 monitoring tools, we wil have 3 namespaces - inclusive of Gluu Server namespace.   

For each of the monitoring tool namespace we must assign cluster reader permision to this namespace so that each tool has access to k8s APIs.

- Create a `clusturerole.yaml`. for every  Copy the contents 
- create the role `kubectl create -f clusterrole.yaml -n monitoring`

### Launch datadog agent in monitoring namespace
```
helm install --name datadog-agent-v1 \
   --set datadog.apiKey=<DataDog API Key> \
   --set datadog.apmEnabled=true \
   --set datadog.logsEnabled=true \
   stable/datadog
   - n datadog
```

### prometheus
- Create a namespace for prometheus
`kubectl create namespace prometheus`

- Use the files in [prometheus](/prometheus) to launch prometheus with datadog Autodiscovery enabled.   
`kubectl apply -f prometheus/ ` 

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
