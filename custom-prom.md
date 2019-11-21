# Custom Prometheus

Integrating Kubecost with an existing Prometheus installation can be nuanced. We recommend first installing Kubecost with a bundled Prometheus ([instructions](http://kubecost.com/install)) as a dry run before integrating with your own Prometheus. We also recommend getting in touch (team@kubecost.com) for assistance. 

**Note:** integrating with an existing Prometheus is officially supported under all Kubecost paid plans. 

__Required Steps__

1. Copy [values.yaml](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml) and update the following parameters:
  
   `promtheus.fqdn` to match your local Prometheus with this format `http://<prometheus-server-service-name>.<prometheus-server-namespace>.svc.cluster.local`  
   `prometheus.enabled` set to `false`  
  
   Pass this updated file to the Kubecost helm install command with `--values values.yaml` 

2. <a name="scrape-configs"></a>Have your Prometheus scrape the cost-model `/metrics` endpoint. These metrics are needed for reporting accurate pricing data. Here is an example scrape config:

```
- job_name: kubecost
      honor_labels: true
      scrape_interval: 1m
      scrape_timeout: 10s
      metrics_path: /metrics
      scheme: http
      dns_sd_configs:
      - names:
        - kubecost-cost-analyzer.<namespace-of-your-kubecost>
        type: 'A'
        port: 9003
```  

This config needs to be added under `extraScrapeConfigs` in Prometheus configuration. [Example](https://github.com/kubecost/cost-analyzer-helm-chart/blob/0758d5df54d8963390ca506ad6e58c597b666ef8/cost-analyzer/values.yaml#L74)

You can confirm that this job is successfully running with the Targets view in Prometheus. 

![Prometheus Targets](/prom-targets.png)

<a name="recording-rules"></a>
__Recording Rules__  
<br/>
Kubecost uses Prometheus [recording rules](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml#L145) to enable certain product features and to help improve product performance. These are recommended additions, especially for medium and large-sized clusters using their own Prometheus installation.


<a name="remote-write"></a>
__Remote Write__
<br/>
Kubecost stores the Prometheus Data in the Postgres Database, so the Remote Write Configs must be configured like in this example:

```
  serverFiles:
    prometheus.yml:
        remote_write:
          - url: "http://pgprometheus-adapter:9201/write"
            write_relabel_configs:
              - source_labels: [__name__]
                regex: 'container_.*_allocation|container_.*_allocation_bytes|.*_hourly_cost|kube_pod_container_resource_requests_memory_bytes|container_memory_working_set_bytes|kube_pod_container_resource_requests_cpu_cores|kube_pod_container_resource_requests|pod_pvc_allocation|kube_namespace_labels|kube_pod_labels'
                action: keep
            queue_config:
              max_samples_per_send: 1000
```

<a name="cluster-id"></a>
__Cluster Id__
<br/>
The ClusterId for Remote Write must be set in Prometheus:

```
  server:
    global:
      external_labels:
        cluster_id: <yourCluster> # Each cluster should have a unique ID
```



 
<a name="troubleshoot"></a>__Troubleshooting Issues__

Common issues include the following: 

* Wrong Prometheus FQDN: evidenced by the following pod error message `No valid prometheus config file at  ...`. We recommend running `curl <your_prometheus_url>/api/v1/status/config` from a pod in the cluster to confirm that your [Prometheus config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration-file) is returned. If not, this is an indication that an incorrect Prometheus Url has been provided. If a config file is returned, then the Kubecost pod likely has it's access restricted by a cluster policy, service mesh, etc. 

* Prometheus throttling -- ensure Prometheus isn't being CPU throttled due to a low resource request.

* Required dependancy versions (node-exporter - v0.16, kube-state-metrics - v1.3, cadvisor)

* Missing scrape configs -- visit Prometheus Target page (screenshot above)

You can visit Settings in Kubecost to see basic diagnostic information on these Prometheus metrics:

![Prometheus status diagnostic](/prom-status.png)
