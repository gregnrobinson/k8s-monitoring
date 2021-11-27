# InfluxDB2 & Telegraf Quickstart

All code was sourced from [`https://github.com/influxdata/helm-charts.git`](https://github.com/influxdata/helm-charts.git). I had to get a local copy of the influxdb2 chart because they have not updated the ingress.yaml configuration for `networking.k8s.io/v1`. I included that in the `./influxdb/templates` directory and generated an Ingress resource that works fine with the deployment.

## InfluxDB

To deploy influx DB 2 run...

```bash
helm upgrade --install influxdb -n monitoring ./influxdb -f influxdb/values.yaml
```

Consider changing the following sections in `values.yaml`...

```yaml
# I need arm64 but you can reference https://hub.docker.com/_/influxdb for supported architectures.
image:
  repository: arm64v8/influxdb
  tag: 2.1.1-alpine
  pullPolicy: IfNotPresent

# I leave the pass empty so it generates automatically.
adminUser:
  organization: "influxdata"
  bucket: "default"
  user: "greg.robinson"
  retention_policy: "0s"
  ## Leave empty to generate a random password and token.
  ## Or fill any of these values to use fixed values.
  password: ""
  token: ""

# I enable persistance and use my nfs dynamic provisioner to allocate storage based on storageClass name.
persistence:
  enabled: true
  storageClass: "nfs-client"
  accessMode: ReadWriteOnce
  size: 50Gi
  mountPath: /var/lib/influxdb2

# I always use LoadBalancer for service types if they off a UI that needs to be accessed. I use metallb for load balancing on servers. 
service:
  type: LoadBalancer
  port: 8086
  targetPort: 8086
  annotations: {}
  labels: {}
  portName: http

# I use ingress for anything I want to access publically. Attach a cert that is generated by Cloudflare. Nginx handles inbound routing to services. End result: https://influxdb.phronesis.cloud
ingress:
  enabled: true
  # For Kubernetes >= 1.18 you should specify the ingress-controller via the field ingressClassName
  # See https://kubernetes.io/blog/2020/04/02/improvements-to-the-ingress-api-in-kubernetes-1.18/#specifying-the-class-of-an-ingress
  ingressClassName: "nginx"
  tls: true
  secretName: wildcard-phronesis-cert
  hostname: influxdb.phronesis.cloud
  annotations: {}
  path: /
```

## Telegraf

### Prerequisites

Create a kubernetes secret called `tokens.yaml` that will hold all sensitive information. This file if ignore by git and it will prevent any secrets being exposed in the values file.

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: telegraf-tokens
type: Opaque
stringData:
  # Required for Telegraf to send data to InfluxDB
  INFLUX_HOST: https://influxdb.phronesis.cloud 
  INFLUX_ORG: <https://docs.influxdata.com/influxdb/v2.1/organizations/view-orgs/>
  INFLUX_TOKEN: <https://docs.influxdata.com/influxdb/cloud/security/tokens/create-token/>
  INFLUX_BUCKET: <https://docs.influxdata.com/influxdb/v2.0/organizations/buckets/>

  # Other Tokens for Plugins
  # https://docs.influxdata.com/telegraf/v1.20/plugins/
  KUBE_BEARER: <ADD_TOKENS_THAT_OTHER_PLUGINS_USE>
EOF
```

### Modify `values.yaml` overrides

Once again, consider changing the following sections in `values.yaml`...

```yaml
# Very happy influxdata has ARM images for all their tools :)
image:
  repo: "arm64v8/telegraf"
  tag: "1.20.4"
  pullPolicy: IfNotPresent

# Leaving this as ClusterIP as Telegraf only talks upstream. In other words, it only sends data outbound to InfluxDB. No need to have a LoadBalancer for Telegraf.
service:
  enabled: true
  type: ClusterIP
  annotations: {}

```

The main thing in `values.yaml` that needs changing is the telegraf config. This is the meat and potatoes of the configuration and will be the configuration that gets the most attention. The plugins that define the data Telegraf will collect is all shown below. There are hundreds of different plugins that can be declared here. Some require additional config, some don't. For example. the temp

- More info: [https://docs.influxdata.com/influxdb/v2.1/write-data/no-code/use-telegraf/manual-config/](ttps://docs.influxdata.com/influxdb/v2.1/write-data/no-code/use-telegraf/manual-config/)
- All Plugins: [https://github.com/influxdata/telegraf/tree/master/plugins](https://github.com/influxdata/telegraf/tree/master/plugins)
- Temp Plugin: [https://github.com/influxdata/telegraf/tree/master/plugins/inputs/temp](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/temp)

The one thing that took me eons to figure out is you define the variables from `tokens.yaml` directly in the `values.yaml` under `config:`. The data is parsed into a `ConfigMap` which eventually is stored as a file on the container so the values will substitute.

```yaml
## Exposed telegraf configuration
## For full list of possible values see `/docs/all-config-values.yaml` and `/docs/all-config-values.toml`
## ref: https://docs.influxdata.com/telegraf/v1.1/administration/configuration/
config:
  agent:
    interval: "60s"
    round_interval: true
    metric_batch_size: 1000
    metric_buffer_limit: 10000
    collection_jitter: "0s"
    flush_interval: "10s"
    flush_jitter: "0s"
    precision: ""
    debug: false
    quiet: false
    logfile: ""
    hostname: ""
    omit_hostname: false
  processors:
    - enum:
        mapping:
          field: "status"
          dest: "status_code"
          value_mappings:
            healthy: 1
            problem: 2
            critical: 3
  outputs:
    - influxdb_v2:
        urls:
          - "https://influxdb.phronesis.cloud"
        bucket: $INFLUX_BUCKET
        token: $INFLUX_TOKEN
        organization: phronesis-influxdb
  inputs:
    - statsd:
        service_address: ":8125"
        percentiles:
          - 50
          - 95
          - 99
        metric_separator: "_"
        allowed_pending_messages: 10000
        percentile_limit: 1000
    - temp:
    - diskio:
    - disk:
    - consul:
        address: "http://consul.phronesis.cloud"
    - internet_speed:
        enable_file_download: false
  health:
    enabled: false
  collect_memstats: false
```

### Install

```bash
helm repo add influxdata https://helm.influxdata.com/

helm upgrade --install telegraf influxdata/telegraf -f telegraf/values.yaml --force -n monitoring --force
```

## Troubleshooting

To open a shell session in the container running Telegraf run the following:

```bash
kubectl exec -i -t --namespace monitoring $(kubectl get pods --namespace monitoring -l app.kubernetes.io/name=telegraf -o jsonpath='{.items[0].metadata.name}') /bin/sh
```

To view the logs for a Telegraf pod, run the following:

```bash
kubectl logs -f --namespace monitoring $(kubectl get pods --namespace monitoring -l app.kubernetes.io/name=telegraf -o jsonpath='{ .items[0].metadata.name }')
```

### Uninstall

```bash
helm uninstall my-release
```
