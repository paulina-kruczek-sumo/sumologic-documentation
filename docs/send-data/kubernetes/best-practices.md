---
id: best-practices
title: Best Practices and Advanced Configuration
sidebar_label: Best Practices
description: This page provides info about advanced configuration and best practices.
---

## Overriding chart resource names with `fullnameOverride`

You can use the `fullnameOverride` properties of this chart and of its subcharts to override the created resource names.

See the chart's [README](https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/main/deploy/helm/sumologic/README.md) for all the available `fullnameOverride` properties.

Here's an example of using the `fullnameOverride` properties with the components that are enabled by default:

```yaml
fullnameOverride: sl

kube-prometheus-stack:
  fullnameOverride: kps

  kube-state-metrics:
    fullnameOverride: ksm

  prometheus-node-exporter:
    fullnameOverride: pne
```

After installing the chart, the resources in the cluster will have names similar to the following:

```console
$ kubectl -n <namespace> get pods
NAME                                     READY   STATUS    RESTARTS   AGE
kps-operator-6fdb4b955-zqsdh             1/1     Running   0          3m4s
ksm-779757976f-4d262                     1/1     Running   0          3m4s
pne-hfk88                                1/1     Running   0          3m4s
prometheus-kps-prometheus-0              2/2     Running   0          3m4s
sl-otelcol-events-0                      1/1     Running   0          3m4s
sl-otelcol-instrumentation-0             1/1     Running   0          3m4s
sl-otelcol-instrumentation-1             1/1     Running   0          3m4s
sl-otelcol-instrumentation-2             1/1     Running   0          3m4s
sl-otelcol-logs-0                        1/1     Running   0          3m4s
sl-otelcol-logs-1                        1/1     Running   0          3m4s
sl-otelcol-logs-2                        1/1     Running   0          3m4s
sl-otelcol-logs-collector-24z56          1/1     Running   0          3m4s
sl-otelcol-metrics-0                     1/1     Running   0          3m4s
sl-otelcol-metrics-1                     1/1     Running   0          3m4s
sl-otelcol-metrics-2                     1/1     Running   0          3m4s
sl-remote-write-proxy-7f79766ff7-pvchj   1/1     Running   0          3m4s
sl-remote-write-proxy-7f79766ff7-qkk8n   1/1     Running   0          3m4s
sl-remote-write-proxy-7f79766ff7-vsjbj   1/1     Running   0          3m4s
sl-traces-gateway-cdc7d9bbc-wq9sl        1/1     Running   0          3m4s
sl-traces-sampler-94656c48d-cslsb        1/1     Running   0          3m4s
```

:::note

When changing the `fullnameOverride` property for an already installed chart with the `helm upgrade` command, you need to restart the Prometheus pods for the changed names of the Otelcol pods to be picked up:

```sh
helm -n <namespace> upgrade <release_name> sumologic/sumologic --values changed-fullnameoverride.yaml
kubectl -n <namespace> rollout restart statefulset <prometheus_statefulset_name>
```

:::

## OpenTelemetry Collector Autoscaling

Autoscaling for logs metadata, metrics metadata, metrics collector, otelcol instrumentation, and traces gateway is enabled by default.

Disabling autoscaling for all components can be done by overriding the `sumologic.autoscaling.enabled` parameter.

```yaml
sumologic:
  autoscaling:
    enabled: false
```

Furthermore, you can disable autoscaling for individual components. Keep in mind that the `autoscaling.enabled` parameter always takes precedence over the global `sumologic.autoscaling.enabled` parameter. Use the following configuration to accomplish this:

- Disable autoscaling for log metadata enrichment
  ```yaml
  metadata:
    logs:
      autoscaling:
        enabled: false
  ```
- Disable autoscaling for metrics metadata enrichment
  ```yaml
  metadata:
    metrics:
      autoscaling:
        enabled: false
  ```
- Disable autoscaling for metrics collector
  ```yaml
  sumologic:
    metrics:
      collector:
        otelcol:
          autoscaling:
            enabled: false
  ```
- Disable autoscaling for otelcol instrumentation
  ```yaml
  otelcolInstrumentation:
    autoscaling:
      enabled: false
  ```
- Disable autoscaling for traces gateway
  ```yaml
  tracesGateway:
    autoscaling:
      enabled: false
  ```

It's also possible to adjust other autoscaling configuration options, like the maximum number of replicas or the average CPU utilization. Refer to the [chart readme][chart_readme] and [default values.yaml][values.yaml] for details.

If metrics-server is not already installed in your cluster, it is required to enable metrics-server.

```yaml
## Configure metrics-server
## ref: https://github.com/bitnami/charts/tree/master/bitnami/metrics-server/values.yaml
metrics-server:
  enabled: true
```

### Cleaning unused PVCs

When autoscaling is enabled, new Persistent Volume Claims (PVCs) will be created for new pods. However, Kubernetes doesn't support removing unused PVCs after downscaling a StatefulSet yet, which means that they will be reused when the StatefulSet gets upscaled again. This creates problems in EKS when new instance of the pod is created in a different availability zone than the initial one, as PVCs cannot be mounted across different AZs.

To solve this problem, `pvcCleaner` can be used to remove unused PVCs:

- For log metadata enrichment
  ```yaml
  pvcCleaner:
    logs:
      enabled: true
  ```
- For metrics metadata enrichment
  ```yaml
  pvcCleaner:
    metrics:
      enabled: true
  ```
  This will create `cronJobs` that will run the [pvcCleaner script][pvcCleaner-script] regularly. The schedule is by default set to 15 minutes, but it can be overridden:
  ```yaml
  pvcCleaner:
    job:
      schedule: "*/10 * * * *"
  ```
  The schedule is written in [Cron] format.

:::note
It is recommended to enable `pvcCleaner` only for the types of telemetry which are autoscaled to avoid creating unnecessary `cronJobs`.
:::

[pvcCleaner-script]: https://github.com/SumoLogic/sumologic-kubernetes-tools/blob/v2.15.0/src/commands/pvc-cleaner
[Cron]: https://en.wikipedia.org/wiki/Cron

## OpenTelemetry Collector Persistent Buffer

The logs and metrics metadata enrichment pods store data in a persistent file-based buffer to prevent data loss during restarts. This feature is enabled by default and can be disabled using the `metadata.persistence.enabled: false` property.

The persistence of the event collection pods is controlled by another property: `sumologic.events.persistence.enabled` and is also enabled by default.

The persistent storage is implemented from the Kubernetes perspective by adding a `volumeClaimTemplate` to the statefulset, which in effect creates a [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) resource for each pod of the statefulset.

For logs and metrics, use the following properties to configure the PVC templates:

- `metadata.persistence.accessMode: ReadWriteOnce` defines the [access mode](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) used in the `PersistentVolumeClaim` resources.
- `metadata.persistence.pvcLabels: {}` allows to set additional labels on the `PersistentVolumeClaim` resources
- `metadata.persistence.size: 10Gi` defines the requested volume size
- `metadata.persistence.storageClass: null` defines the [storage class](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1) name to use in the volume claim template. This property is not set by default, which means that the `storageClassName` will not be set in the `PersistentVolumeClaim` resources and the default storage class will be used (assuming the [DefaultStorageClass](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) admission controller is enabled on the Kubernetes API server).

For events, use the following properties:

- `sumologic.events.persistence.persistentVolume.accessMode: ReadWriteOnce`
- `sumologic.events.persistence.persistentVolume.pvcLabels: {}`
- `sumologic.events.persistence.persistentVolume.storageClass: null`
- `sumologic.events.persistence.size: 10Gi`

## Collect logs from additional files on the Node

To collect logs from additional files on Node, it is necessary to:

- mount log file to make it accessible by log collector pod
- create an additional pipeline in both the log collector and metadata enrichment service to transfer your data
- open a new port for this pipeline to enable sending data from the log collector to the metadata enrichment service

The following configuration can be used for these purposes:

```yaml
otellogs:
  config:
    merge:
      receivers:
        filelog/extrafiles:
          include: [/var/log/extrafiles/extrafile.log]
      exporters:
        otlphttp/extrafiles:
          endpoint: http://${LOGS_METADATA_SVC}.${NAMESPACE}.svc.cluster.local.:4319
      service:
        pipelines:
          logs/extrafiles:
            receivers: [filelog/extrafiles]
            exporters: [otlphttp/extrafiles]
  daemonset:
    extraVolumes:
      - name: extrafiles-mount
        hostPath:
          path: PATH_TO_YOUR_LOG_FILE
          type: File
    extraVolumeMounts:
      - name: extrafiles-mount
        mountPath: /var/log/extrafiles/extrafile.log
        readOnly: true

metadata:
  logs:
    config:
      merge:
        receivers:
          otlp/extrafiles:
            protocols:
              http:
                endpoint: 0.0.0.0:4319
        service:
          pipelines:
            logs/extrafiles:
              receivers: [otlp/extrafiles]
              processors:
                - memory_limiter
                - batch
              exporters: [sumologic/containers]
    statefulset:
      extraPorts:
        - name: otlphttp2
          containerPort: 4319
          protocol: TCP
```

In the example above, two internally defined processors were used in metadata pipeline:
[batch](https://github.com/open-telemetry/opentelemetry-collector/tree/v0.90.1/processor/batchprocessor) and [memory limiter](https://github.com/open-telemetry/opentelemetry-collector/tree/v0.90.1/processor/memorylimiterprocessor). If you need to change the parameters of these processors in any way, you can define your own and use them in this pipeline.

## Removing attributes from systemd logs

If you want to remove some attributes from Systemd logs, like for example `PRIORITY` and `SYSLOG_FACILITY`, you can do it the following way:

```yaml
sumologic:
  logs:
    systemd:
      otelcol:
        extraProcessors:
          - transform/cleanup_systemd:
              log_statements:
                - context: log
                  statements:
                    - delete_key(body, "PRIORITY")
                    - delete_key(body, "SYSLOG_FACILITY")
```

## Modify the Log Level for Falco

To modify the default log level for Falco, edit the following section in the `user-values.yaml` file. Available log levels can be found in Falco's documentation here: https://falco.org/docs/configuration/.

```yaml
falco:
  ## Set the enabled flag to false to disable falco.
  enabled: true
  falco:
    json_output: true
    log_level: debug
```

## Overriding metadata using annotations

You can use [Kubernetes annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) to override some metadata and settings per pod.

- `sumologic.com/sourceCategory` overrides the value of the `sumologic.logs.container.sourceCategory` property
- `sumologic.com/sourceCategoryPrefix` overrides the value of the `sumologic.logs.container.sourceCategoryPrefix` property
- `sumologic.com/sourceCategoryReplaceDash` overrides the value of the `sumologic.logs.container.sourceCategoryReplaceDash` property
- `sumologic.com/sourceName` overrides the value of the `sumologic.logs.container.sourceName` property

For example:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    app: mywebsite
  template:
    metadata:
      name: nginx
      labels:
        app: mywebsite
      annotations:
        sumologic.com/format: "text"
        sumologic.com/sourceCategory: "mywebsite/nginx"
        sumologic.com/sourceName: "mywebsite_nginx"
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

### Overriding source category with pod annotations

The following example shows how to customize the source category for data from a specific deployment. The resulting value of `_sourceCategory` field will be `my-component`:

```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: my-component-deployment
spec:
  replicas: 1
  selector:
    app: my-component
  template:
    metadata:
      annotations:
        sumologic.com/sourceCategory: "my-component"
        sumologic.com/sourceCategoryPrefix: ""
        sumologic.com/sourceCategoryReplaceDash: "-"
      labels:
        app: my-component
      name: my-component
    spec:
      containers:
        - name: my-component
          image: my-image
```

The `sumologic.com/sourceCategory` annotation defines the source category for the data.

The empty `sumologic.com/sourceCategoryPrefix` annotation removes the default prefix added to the source category.

The `sumologic.com/sourceCategoryReplaceDash` annotation with value `-` prevents the dash in the source category from being replaced with another character.

### Excluding data using annotations

You can use the `sumologic.com/exclude` [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) to exclude data from Sumo. This data is sent to the metadata enrichment service, but not to Sumo.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    app: mywebsite
  template:
    metadata:
      name: nginx
      labels:
        app: mywebsite
      annotations:
        sumologic.com/exclude: "true"
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

#### Including subsets of excluded data

If you excluded a whole namespace, but still need one or few pods to be still included for shipping to Sumo, you can use the `sumologic.com/include` annotation to include it. It takes precedence over the exclusion described above.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    app: mywebsite
  template:
    metadata:
      name: nginx
      labels:
        app: mywebsite
      annotations:
        sumologic.com/format: "text"
        sumologic.com/sourceCategory: "mywebsite/nginx"
        sumologic.com/sourceName: "mywebsite_nginx"
        sumologic.com/include: "true"
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

## Templating Kubernetes metadata

The following Kubernetes metadata is available for string templating:

| String template  | Description                                             |
| :--------------- | :------------------------------------------------------ |
| `%{namespace}`   | Namespace name                                          |
| `%{pod}`         | Full pod name (e.g., `travel-products-4136654265-zpovl`) |
| `%{pod_name}`    | Friendly pod name (e.g., `travel-products`)              |
| `%{pod_id}`      | The pod's uid (a UUID)                                  |
| `%{container}`   | Container name                                          |
| `%{source_host}` | Host                                                    |
| `%{label:foo}`   | The value of label `foo`                                |

### Missing labels

Unlike the other templates, labels are not guaranteed to exist, so missing labels interpolate as `"undefined"`.

For example, if you have only the label `app: travel` but you define `SOURCE_NAME="%{label:app}@%{label:version}"`, the source name will appear as `travel@undefined`.

## Disable logs, metrics, or falco

If you want to disable the collection of logs, metrics, or falco, make the below changes respectively in the `user-values.yaml` file and run the `helm upgrade` command.

| parameter                   | value | function                   |
| :--------------------------- | :----- | :-------------------------- |
| `sumologic.logs.enabled`    | false | disable logs collection    |
| `sumologic.metrics.enabled` | false | disable metrics collection |
| `falco.enabled`             | false | disable falco              |

## Changing scrape interval for Prometheus

Default scrapeInterval for collection is `30s`. This is the recommended value which ensures that all of Sumo Logic dashboards are filled up with proper data.

To change it, you can use following configuration:

```yaml
kube-prometheus-stack: # For user-values.yaml
  prometheus:
    prometheusSpec:
      scrapeInterval: "1m"
```

## Get logs not available on stdout

When logs from a pod are not available on stdout, [Tailing Sidecar Operator](https://github.com/SumoLogic/tailing-sidecar) can help with collecting them using standard logging pipeline. To tail logs using Tailing Sidecar Operator the file with those logs needs to be accessible through a volume mounted to sidecar container.

Providing that the file with logs is accessible through volume, to enable tailing of logs using Tailing Sidecar Operator:

- Enable Tailing Sidecar Operator by modifying `user-values.yaml`:

  ```yaml
  tailing-sidecar-operator:
    enabled: true
  ```

- Add annotation to pod from which you want to tail logs in the following format:

  ```yaml
  metadata:
    annotations:
      tailing-sidecar: <sidecar-name-0>:<volume-name-0>:<path-to-tail-0>;<sidecar-name-1>:<volume-name-1>:<path-to-tail-1>
  ```

Example of using Tailing Sidecar Operator is described in the [blog post](https://www.sumologic.com/blog/tailing-sidecar-operator/).

## Using custom Kubernetes API server address

In order to change API server address, the following configurations can be used.

```yaml
metadata:
  logs:
    statefulset:
      extraEnvVars:
        - name: KUBERNETES_SERVICE_HOST
          value: my-custom-k8s.api
        - name: KUBERNETES_SERVICE_PORT
          value: "12345"
  metrics:
    statefulset:
      extraEnvVars:
        - name: KUBERNETES_SERVICE_HOST
          value: my-custom-k8s.api
        - name: KUBERNETES_SERVICE_PORT
          value: "12345"
```

## OpenTelemetry Collector queueing and batching

OpenTelemetry comes with several parameters related to queue management.

For [batch processor][batch_processor]:

- `send_batch_size` defines the number of items (logs, metrics, traces) in one batch before it's sent further down the pipeline.
- `timeout` defines time after which the batch is sent regardless of the size (can be lower than `send_batch_size`).
- `send_batch_max_size` is an upper limit of the batch size.

_We could say that `send_batch_size` is a soft limit and `send_batch_max_size` is a hard limit of the batch size._

In Helm Chart's default configuration, `send_batch_max_size` is set to `2 * send_batch_size`. It is necessary to consider changing the value of `send_batch_max_size` whenever `send_batch_size` is changed. More information can be found in [sumologic-otel-collector's documentation][sumo-otelcol-batching-doc].

For [sumologic exporter][sumologic_exporter]:

- `max_request_body_size` defines maximum size of requests to sumologic before compression.
- `timeout` defines connection timeout. It is recommended to adjust this value in relation to `max_request_body_size`.

- `sending_queue.num_consumers` is the number of consumers that dequeue batches. It translates to maximum number of parallel connections to the sumologic backend.
- `sending_queue.queue_size` is capacity of the queue in terms of batches (batches can vary between `1` and `send_batch_max_size`)

:::note
As the effective value of `sending_queue.queue_size` depends on current traffic, there is no way to figure out optimal PVC size in relation to `sending_queue.queue_size`. Due to this, we recommend setting `sending_queue.queue_size` to high value in order to use maximum resources of PVC.

The above, in connection with PVC monitoring, can lead to constant alerts (e.g., [KubePersistentVolumeFillingUp][filling_up_alert]), because once filled in PVC never reduces its fill.
:::

[batch_processor]: https://github.com/open-telemetry/opentelemetry-collector/tree/v0.47.0/processor/batchprocessor#batch-processor
[sumologic_exporter]: https://github.com/SumoLogic/sumologic-otel-collector/tree/v0.50.0-sumo-0/pkg/exporter/sumologicexporter#sumo-logic-exporter
[filling_up_alert]: https://runbooks.prometheus-operator.dev/runbooks/kubernetes/kubepersistentvolumefillingup/
[sumo-otelcol-batching-doc]: ../../opentelemetry-collector/data-source-configurations/additional-configurations-reference/#using-batch-processor-to-batch-data

### Compaction

The OpenTelemetry Collector doesn't have a compaction mechanism. Local storage can only grow - it can reuse disk space that has already been allocated, but not free it. This leads to a situation where the database file can grow a lot (due to a spike in data traffic) but after some time only small piece of the file will be used for data storage (until next spike).

### Examples

Here are some useful examples and calculations for queue and batch parameters.

For the calculations below we made an assumption that a single metric data point is around 1 kilobyte in size, including metadata. This assumption is based on the average data we ingest. Persistent storage doesn't compress data so we assume that single metric data point takes 1 kilobyte on disk as well.

`number_of_instances` represents number of `sumologic-otelcol-metrics` instances.

#### Outage with huge metrics spike

Let's consider a huge metrics spike in your network while connection to the Sumologic is down. Huge load means that batch processor is going to push batches due to `send_batch_max_size` instead of `timeout`. The reliability of the system can be calculated using the following formulas:

- If limited by queue_size: `number_of_instances*send_batch_max_size*sending_queue.queue_size/load_in_DPM` minutes.
- If limited by PVC size: `number_of_instances*PVC_size/(1KB*load_in_DPM)` minutes.

#### Outage with low DPM load

Let's consider a low but constant load in your network while connection to the Sumologic is down. Low load means that batch processor is going to push batches due to `timeout` instead of `send_batch_max_size`. The reliability of the system can be calculated using the following formulas:

- If limited by queue_size: `number_of_instances*timeout[min]*sending_queue.queue_size/load_in_DPM` minutes.
- If limited by PVC size: `number_of_instances*PVC_size/(1KB*load_in_DPM)` minutes.

#### Example configuration

Here is an example configuration to change `send_batch_size`, `send_batch_max_size`, and `timeout` for logs metadata otelcol, metrics metadata otelcol and logs collector otelcol.

```yaml
metadata:
  logs:
    config:
      merge:
        processors:
          batch:
            timeout: 10s
            send_batch_size: 1_024
            send_batch_max_size: 2_048
  metrics:
    config:
      merge:
        processors:
          batch:
            timeout: 10s
            send_batch_size: 1_024
            send_batch_max_size: 2_048

otellogs:
  config:
    merge:
      processors:
        batch:
          timeout: 10s
          send_batch_size: 1_024
          send_batch_max_size: 2_048
```

Below there is example configuration to change `sending_queue` for metrics metadata otelcol, logs metadata otelcol and logs collector otelcol

```yaml
metadata:
  logs:
    config:
      merge:
        exporters:
          sumologic/containers:
            sending_queue:
              queue_size: 25000
          sumologic/systemd:
            sending_queue:
              queue_size: 25000

  metrics:
    config:
      merge:
        exporters:
          sumologic/apiserver:
            sending_queue:
              queue_size: 25000
          sumologic/control_plane:
            sending_queue:
              queue_size: 25000
          sumologic/controller:
            sending_queue:
              queue_size: 25000
          sumologic/default:
            sending_queue:
              queue_size: 25000
          sumologic/kubelet:
            sending_queue:
              queue_size: 25000
          sumologic/node:
            sending_queue:
              queue_size: 25000
          sumologic/scheduler:
            sending_queue:
              queue_size: 25000
          sumologic/state:
            sending_queue:
              queue_size: 25000

otellogs:
  config:
    merge:
      exporters:
        otlphttp:
          sending_queue:
            queue_size: 20
```

## Assigning Pod to particular Node

### Using NodeSelectors

Kubernetes offers a feature of assigning specific pod to node. Such kind of control is sometimes useful, whenever you want to ensure that pod will end up on specific node according your requirements like operating system or connected devices.

#### Binding pods to linux nodes

Using this feature we can bind them to linux nodes. In order to do that `nodeSelector` has to be used. Node selector can be changed via additional parameter in `user-values.yaml`, see an example for PVC Cleaner below:

```yaml
pvcCleaner:
  job:
    nodeSelector:
      kubernetes.io/os: linux
```

You can also specify a global nodeSelector option that will be used for all components except for the subcharts. The option for a component has a higher priority than the global one.

To specify the global option, use key `sumologic.nodeSelector`:

```yaml
sumologic:
  nodeSelector:
    kubernetes.io/os: linux

metadata:
  metrics:
    statefulset:
      nodeSelector:
        kubernetes.io/os: metricOS
```

The above config will set:

- `nodeSelector.kubernetes.io/os` to `metricOS` for metadata metrics statefulset
- nothing for subcharts
- `nodeSelector.kubernetes.io/os` to `linux` for other components

See [the section about global customizations](#node-selectors) for information about setting nodeSelectors for subchart resources.

#### Setting different resources on different nodes for logs collector

All of the log collector Pods have the same resource settings, being members of the same DaemonSet. However, sometimes it's necessary to vary this based on the Node - for example when some large Nodes can contain applications generating a lot of log data.

It's possible to set different resource quotas for different Nodes based on labels, resulting in a different log collector DaemonSet for these Nodes.

Let's consider the following example.

You have node group with common label `workingGroup: IntenseLogGeneration` and for that specific group of nodes, you would like to set different resources than for rest of the cluster.

We assume that you only want to set `requests.cpu` to `2` and `limits.cpu` to `10` and have the same memory values like main DaemonSet.

```yaml
otellogs:
  additionalDaemonSets:
    ## intense will be suffix for daemonset for easier recognition
    intense:
      nodeSelector:
        ## we are using nodeSelector to select only nodes with `workingGroup` label set to `IntenseLogGeneration`
        workingGroup: IntenseLogGeneration
      resources:
        requests:
          cpu: 1
        limits:
          cpu: 10
  daemonset:
    # For main daemonset, we need to set nodeAffinity to not schedule on nodes with `workingGroup` label set to `IntenseLogGeneration`
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: workingGroup
                  operator: NotIn
                  values:
                    - IntenseLogGeneration
```

[chart_readme]: https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/main/deploy/helm/sumologic/README.md
[values.yaml]: https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/main/deploy/helm/sumologic/values.yaml

## Keeping Source Category for metrics

In order to keep Source Category for metrics, the following configuration can be applied:

```yaml
metadata:
  metrics:
    config:
      merge:
        processors:
          resource/delete_source_metadata:
            attributes:
              ## Do not remove _sourceCategory
              # - key: _sourceCategory
              #   action: delete
              - key: _sourceHost
                action: delete
              - key: _sourceName
                action: delete
```

## Using newer Kube Prometheus Stack

Due to breaking changes, we do not support the latest [Kube Prometheus Stack][kube-prometheus-stack]. We are aware that it can be a major issue, so this section describes how to install newer Kube Prometheus Stack to work with our collection.

1. Prepare `values.yaml` for Kube Prometheus Stack.
   - copy content of `kube-prometheus-stack` from [values.yaml][values.yaml], e.g., for Sumologic Kubernetes Collection v3.9 you need to copy [these][kube-prometheus-stack-3.9] lines
   - remove configuration for images for Kube Prometheus Stack to use newer versions, e.g., for Sumologic Kubernetes Collection v3.9 you need to remove [tag][kube-state-metrics-tag] for kube-state-metrics
   - add your custom configuration
1. Upgrade sumologic chart without Kube Prometheus Stack by adding the following configuration to your `values.yaml`:
   ```yaml
   kube-prometheus-stack:
     enabled: false
   ```
1. Verify changes in newer Kube Prometheus Stack which you want to install, you can find information about changes [here][kube-prometheus-stack-upgrade].
1. Install CRD for Kube Prometheus Stack to do this please see [upgrade instruction][kube-prometheus-stack-upgrade] for Kube Prometheus Stack, for example to upgrade from 40.x to 41.x:
   ```bash
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagerconfigs.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_alertmanagers.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_podmonitors.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_probes.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_prometheusrules.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_servicemonitors.yaml
   kubectl apply --server-side -f
   https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.60.1/example/prometheus-operator-crd/monitoring.coreos.com_thanosrulers.yaml
   ```
1. Install Kube Prometheus Stack in the same namespace in which collection has been installed:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update

   helm install kube-prometheus prometheus-community/kube-prometheus-stack \
     --namespace <NAMESPACE> \
     --version <VERSION> \
     -f values-prometheus.yaml
   ```

[kube-prometheus-stack]: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
[kube-prometheus-stack-3.9]: https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/3d5bf2855f0a5254007b7dfffa64b320cefadf53/deploy/helm/sumologic/values.yaml#L1418-L3462
[kube-state-metrics-tag]: https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/3d5bf2855f0a5254007b7dfffa64b320cefadf53/deploy/helm/sumologic/values.yaml#L1941
[kube-prometheus-stack-upgrade]: https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#upgrading-chart

## Lowering default ingest

By default, the Helm Chart collects some data necessary for Sumo Logic dashboards to work. If these dashboards are not used, we suggest to disable collection of this data in order to lower the ingest.

In order to learn more about filtering out data please see [the doc about filtering data](https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/main/docs/filtering.md).

## Global customizations

Some common Kubernetes resource attributes can be set globally for all resources managed by the Helm Chart. Due to Helm's limitations, they need to be separately set for subchart resources. This section lists the relevant attributes and configuration values.

:::note
Each subchart can be independently configured. To make sure you don't miss anything you want to change, make sure to check their documentation. Links to documentations of subcharts can be found [here][subcharts-docs].
:::

[subcharts-docs]: https://github.com/SumoLogic/sumologic-kubernetes-collection/blob/main/deploy/helm/sumologic/README.md#configuration

### Node selectors

These options allow you to set node selectors for whole subcharts. The values under these keys must be a map of node selectors. [More information about node selectors can be found here][k8s-node-selector].

| subchart                 | key                                                            |
| :------------------------ | :-------------------------------------------------------------- |
| `sumologic`              | `sumologic.nodeSelector`                                       |
| `kube-prometheus-stack`  | `kube-prometheus-stack.prometheus-node-exporter.nodeSelector`  |
| `kube-state-metrics`     | `kube-prometheus-stack.kube-state-metrics.nodeSelector`        |
| `prometheus`             | `kube-prometheus-stack.prometheus.prometheusSpec.nodeSelector` |
| `opentelemetry-operator` | `opentelemetry-operator.nodeSelector`                          |
| `falco`                  | `falco.nodeSelector`                                           |
| `telegraf-operator`      | `telegraf-operator.nodeSelector`                               |

In the main `sumologic` chart, value specified in `sumologic.nodeSelector` can be overridden for the following pods:

| component                | key                                                |
| :------------------------ | :-------------------------------------------------- |
| `remoteWriteProxy`       | `sumologic.metrics.remoteWriteProxy.nodeSelector`  |
| `otelcolInstrumentation` | `otelcolInstrumentation.statefulset.nodeSelector`  |
| `tracesGateway`          | `tracesGateway.deployment.nodeSelector`            |
| `otellogs`               | `otellogs.daemonset.nodeSelector`                  |
| `setupJob`               | `sumologic.setup.job.nodeSelector`                 |
| `tracesSampler`          | `tracesSampler.deployment.nodeSelector`            |
| `metadata`               | `metadata.metrics.statefulset.nodeSelector`        |
| `metadata`               | `metadata.logs.statefulset.nodeSelector`           |
| `pvcCleaner`             | `pvcCleaner.job.nodeSelector`                      |
| `metrics otelcol`        | `sumologic.metrics.collector.otelcol.nodeSelector` |

[k8s-node-selector]: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector

### Tolerations

These options allow you to set tolerations for whole subcharts. The values under these keys must be a list of tolerations. [More information about node selectors can be found here][k8s-tolerations].

| subchart                 | key                                                           |
| :------------------------ | :------------------------------------------------------------- |
| `sumologic`              | `sumologic.tolerations`                                       |
| `kube-prometheus-stack`  | `kube-prometheus-stack.prometheus-node-exporter.tolerations`  |
| `kube-state-metrics`     | `kube-prometheus-stack.kube-state-metrics.tolerations`        |
| `prometheus`             | `kube-prometheus-stack.prometheus.prometheusSpec.tolerations` |
| `opentelemetry-operator` | `opentelemetry-operator.tolerations`                          |
| `falco`                  | `falco.tolerations`                                           |
| `telegraf-operator`      | `telegraf-operator.tolerations`                               |

In the main `sumologic` chart, value specified in `sumologic.tolerations` can be overridden for the following pods:

| component                | key                                               |
| :------------------------ | :------------------------------------------------- |
| `remoteWriteProxy`       | `sumologic.metrics.remoteWriteProxy.tolerations`  |
| `otelcolInstrumentation` | `otelcolInstrumentation.statefulset.tolerations`  |
| `tracesGateway`          | `tracesGateway.deployment.tolerations`            |
| `otellogs`               | `otellogs.daemonset.tolerations`                  |
| `setupJob`               | `sumologic.setup.job.tolerations`                 |
| `tracesSampler`          | `tracesSampler.deployment.tolerations`            |
| `metadata`               | `metadata.metrics.statefulset.tolerations`        |
| `metadata`               | `metadata.logs.statefulset.tolerations`           |
| `pvcCleaner`             | `pvcCleaner.job.tolerations`                      |
| `metrics otelcol`        | `sumologic.metrics.collector.otelcol.tolerations` |

[k8s-tolerations]: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

### Affinity

These options allow you to set affinity for whole subcharts. The values under these keys must be a map of affinities and anti-affinities. [More information about affinity and anti-affinity can be found here][k8s-affinity].

| subchart                 | key                                                        |
| :------------------------ | :---------------------------------------------------------- |
| `sumologic`              | `sumologic.affinity`                                       |
| `kube-prometheus-stack`  | `kube-prometheus-stack.prometheus-node-exporter.affinity`  |
| `kube-state-metrics`     | `kube-prometheus-stack.kube-state-metrics.affinity`        |
| `prometheus`             | `kube-prometheus-stack.prometheus.prometheusSpec.affinity` |
| `opentelemetry-operator` | `opentelemetry-operator.affinity`                          |
| `falco`                  | `falco.affinity`                                           |
| `telegraf-operator`      | `telegraf-operator.affinity`                               |

In the main `sumologic` chart, value specified in `sumologic.affinity` can be overridden for the following pods:

| component                | key                                            |
| :------------------------ | :---------------------------------------------- |
| `remoteWriteProxy`       | `sumologic.metrics.remoteWriteProxy.affinity`  |
| `otelcolInstrumentation` | `otelcolInstrumentation.statefulset.affinity`  |
| `tracesGateway`          | `tracesGateway.deployment.affinity`            |
| `otellogs`               | `otellogs.daemonset.affinity`                  |
| `setupJob`               | `sumologic.setup.job.affinity`                 |
| `tracesSampler`          | `tracesSampler.deployment.affinity`            |
| `metadata`               | `metadata.metrics.statefulset.affinity`        |
| `metadata`               | `metadata.logs.statefulset.affinity`           |
| `pvcCleaner`             | `pvcCleaner.job.affinity`                      |
| `metrics otelcol`        | `sumologic.metrics.collector.otelcol.affinity` |

[k8s-affinity]: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity

### Pod labels

These options allow you to set pod labels for whole subcharts. The values under these keys must be a map of pod labels. [More information about labels can be found here][k8s-labels].

| subchart     | key                           |
| :------------- | :--------------------------------- |
| `sumologic`              | `sumologic.podLabels`                                                |
| `kube-prometheus-stack`  | `kube-prometheus-stack.prometheus-node-exporter.podLabels`           |
| `prometheus`             | `kube-prometheus-stack.prometheus.prometheusSpec.podMetadata.labels` |
| `opentelemetry-operator` | `opentelemetry-operator.manager.podLabels`                           |
| `falco`                  | `falco.podLabels`                                                    |
| `metrics-server`         | `metrics-server.podLabels`                                           |

In the main `sumologic` chart, you can specify additional pod labels for these pods:

| component                | key                                             |
| :------------------------ | :----------------------------------------------- |
| `remoteWriteProxy`       | `sumologic.metrics.remoteWriteProxy.podLabels`  |
| `otelcolInstrumentation` | `otelcolInstrumentation.statefulset.podLabels`  |
| `tracesGateway`          | `tracesGateway.deployment.podLabels`            |
| `otellogs`               | `otellogs.daemonset.podLabels`                  |
| `setupJob`               | `sumologic.setup.job.podLabels`                 |
| `tracesSampler`          | `tracesSampler.deployment.podLabels`            |
| `metadata (all)`         | `metadata.podLabels`                            |
| `metadata (metrics)`     | `metadata.metrics.statefulset.podLabels`        |
| `metadata (logs)`        | `metadata.logs.statefulset.podLabels`           |
| `pvcCleaner`             | `pvcCleaner.job.podLabels`                      |
| `metrics otelcol`        | `sumologic.metrics.collector.otelcol.podLabels` |

[k8s-labels]: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

### Pod annotations

These options allow you to set pod annotations for whole subcharts. The values under these keys must be a map of pod annotations. [More information about annotations can be found here][k8s-annotations].

| subchart                 | key                                                                       |
| :------------------------ | :------------------------------------------------------------------------- |
| `sumologic`              | `sumologic.podAnnotations`                                                |
| `kube-prometheus-stack`  | `kube-prometheus-stack.prometheus-node-exporter.podAnnotations`           |
| `kube-state-metrics`     | `kube-prometheus-stack.kube-state-metrics.podAnnotations`                 |
| `prometheus`             | `kube-prometheus-stack.prometheus.prometheusSpec.podMetadata.annotations` |
| `opentelemetry-operator` | `opentelemetry-operator.manager.podAnnotations`                           |
| `falco`                  | `falco.podAnnotations`                                                    |
| `metrics-server`         | `metrics-server.podAnnotations`                                           |

In the main `sumologic` chart, you can specify additional pod annotations for these pods:

| component                | key                                                  |
| :------------------------ | :---------------------------------------------------- |
| `remoteWriteProxy`       | `sumologic.metrics.remoteWriteProxy.podAnnotations`  |
| `otelcolInstrumentation` | `otelcolInstrumentation.statefulset.podAnnotations`  |
| `tracesGateway`          | `tracesGateway.deployment.podAnnotations`            |
| `otellogs`               | `otellogs.daemonset.podAnnotations`                  |
| `setupJob`               | `sumologic.setup.job.podAnnotations`                 |
| `tracesSampler`          | `tracesSampler.deployment.podAnnotations`            |
| `metadata (all)`         | `metadata.podAnnotations`                            |
| `metadata (metrics)`     | `metadata.metrics.statefulset.podAnnotations`        |
| `metadata (logs)`        | `metadata.logs.statefulset.podAnnotations`           |
| `pvcCleaner`             | `pvcCleaner.job.podAnnotations`                      |
| `metrics otelcol`        | `sumologic.metrics.collector.otelcol.podAnnotations` |

[k8s-annotations]: https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/

### Images

Every image configuration is a map that consists of two elements: `repository` and `tag`. Images defined in this helm chart also have `pullPolicy` key. For subcharts, use the following keys to override main images:

| subchart                 | key                                                     |
| :------------------------ | :------------------------------------------------------- |
| `kube-prometheus-stack`  | `kube-prometheus-stack.prometheus-node-exporter.image`  |
| `kube-state-metrics`     | `kube-prometheus-stack.kube-state-metrics.image`        |
| `prometheus`             | `kube-prometheus-stack.prometheus.prometheusSpec.image` |
| `opentelemetry-operator` | `opentelemetry-operator.manager.image`                  |
| `falco`                  | `falco.image`                                           |
| `metrics-server`         | `metrics-server.image`                                  |
| `telegraf-operator`      | `telegraf-operator.image`                               |

In the main `sumologic` chart, you can specify additional pod annotations for these pods:

| component                | key                                         |
| :------------------------ | :------------------------------------------- |
| `remoteWriteProxy`       | `sumologic.metrics.remoteWriteProxy.image`  |
| `otelcolInstrumentation` | `otelcolInstrumentation.statefulset.image`  |
| `tracesGateway`          | `tracesGateway.deployment.image`            |
| `otellogs`               | `otellogs.daemonset.image`                  |
| `setupJob`               | `sumologic.setup.job.image`                 |
| `tracesSampler`          | `tracesSampler.deployment.image`            |
| `metadata (all)`         | `metadata.image`                            |
| `pvcCleaner`             | `pvcCleaner.job.image`                      |
| `metrics otelcol`        | `sumologic.metrics.collector.otelcol.image` |
| `otelevents`             | `otelevents.image`                          |

You can also set a default image for every OpenTelemetry collector instance. To do so, use key `sumologic.otelcolImage`:

```yaml
sumologic:
  otelcolImage:
    repository: "public.ecr.aws/sumologic/sumologic-otel-collector"
    tag: "0.90.1-sumo-0"

    ## Add a -fips suffix to all image tags. With default tags, this results in FIPS-compliant otel images.
    ## See https://github.com/SumoLogic/sumologic-otel-collector/blob/main/docs/fips.md for more information.
    addFipsSuffix: false
```
