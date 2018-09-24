# Monitoring stack on Kubernetes.

This documentation aim to document Monitoring in a kubernetes environment.

**Prometheus Server & Alertmanager**

Prometheus is an open-source monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.

**Grafana**

Grafana is an open-source system for querying, analysing and visualizing metrics.


**Nginx Ingress Controller**

Deploying Nginx Ingress Controller allows us to provide TLS termination to our
services and to provide basic authentication to Prometheus Expression browser/API.

We have two networking configuration possible:

* __NodePort__: Our services will be publicly exposed on each node of the cluster, 
including master nodes, at port 30080 for HTTP and 30443 for HTTPS.
* __ClusterIP with external IP(s)__: Our services will be exposed on specific nodes
of the cluster, at port 80 for HTTP and 443 for HTTPS.


## Prerequisites

1. Monitoring namespace

We will deploy our monitoring stack in its own namespace:

```console
$ kubectl create namespace monitoring
```

2. Create DNS entries

In this example, we will use the a worker node with IP 10.84.152.113 to expose
our services.

```
monitoring.example.com                      IN  A       10.84.152.113
prometheus.example.com                      IN  CNAME   monitoring.example.com
prometheus-alertmanager.example.com         IN  CNAME   monitoring.example.com
grafana.example.com                         IN  CNAME   monitoring.example.com
```

Or add this entry to /etc/hosts

```
10.84.152.113 prometheus.example.com prometheus-alertmanager.example.com grafana.example.com
```

3. Create certificates

SSL certificates are required for the domains prometheus.example.com, prometheus-alertmanager.example.com and grafana.example.com


## Nginx Ingress Controller

1. Choose a network configuration

**NodePort**

Create a configuration file *nginx-ingress-config-values.yaml*

```yaml
# Enable the creation of pod security policy
podSecurityPolicy:
  enabled: true

# Create a specific service account
serviceAccount:
  create: true
  name: nginx-ingress

# Publish services on port HTTP/30080
# Publish services on port HTTPS/30443
# These services are exposed on each node
controller:
  service:
    type: NodePort
    nodePorts:
      http: 30080
      https: 30443
```


**ClusterIP with external IP(s)**

Create a configuration file *nginx-ingress-config-values.yaml*

```yaml
# Enable the creation of pod security policy
podSecurityPolicy:
  enabled: true

# Create a specific service account
serviceAccount:
  create: true
  name: nginx-ingress

# Publish services on port HTTP/80
# Publish services on port HTTPS/443
# These services are exposed on the node with IP 10.84.152.113
controller:
  service:
    externalIPs:
      - 10.84.152.113
```

2. Deploy

Deploy the upstream helm chart and pass our configuration values file.

```console
$ helm install --name nginx-ingress stable/nginx-ingress \
  --namespace monitoring \
  --values nginx-ingress-config-values.yaml
```

=> we need two pods running

```console
$ kubectl -n monitoring get po
NAME                                             READY     STATUS    RESTARTS   AGE
nginx-ingress-controller-74cffccfc-p8xbb         1/1       Running   0          4s
nginx-ingress-default-backend-6b9b546dc8-mfkjk   1/1       Running   0          4s
```


## TLS

The next step explains how to create a self-signed certificate if you don't have 
a Certificate Authority, this is not recommended as this is subject to MITM attacks.
When deploying for production, get the certificates from your company CA and skip this step.

* self-signed certificate generation (optional)

This certificate uses Subject Alternative Names so it can be used for Prometheus
and Grafana.

* Create a file *openssl.conf* with the appropriate values (optional)

```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
default_md = sha256
default_bits = 4096
prompt=no

[req_distinguished_name]
C = CZ
ST = CZ
L = Prague
O = example
OU = monitoring
CN = example.com
emailAddress = admin@example.com

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = prometheus.example.com
DNS.2 = prometheus-alertmanager.example.com
DNS.3 = grafana.example.com
```

* Generate certificate (optional)

```console
$ openssl req -x509 -nodes -days 365 -newkey rsa:4096 -keyout ./monitoring.key -out ./monitoring.crt -config ./openssl.conf -extensions 'v3_req'
```

1. Create the TLS secret in Kubernetes

If you use one certificate per service, do not forget to create a secret for each
certificate and to change the secret name in the TLS ingress configuration of the service.

```
$ kubectl create -n monitoring secret tls monitoring-tls  \
  --key  ./monitoring.key \
  --cert ./monitoring.crt
```


## Prometheus

Deploying Prometheus Pushgateway is out of the scope of this document.

1. Manage permissions

Prometheus node-exporter is in charge of getting the host metrics, to do so, it needs access
to /proc or /sys path on the host. This is achieve with the use of HostPath in the pod specs,
however using HostPath is forbidden by default in CaaSP so we need to assign the privileged PodSecurityPolicy to the node-exporter ServiceAccount, this ServiceAccount will be in 
charge of creating the pods.

```console
$ kubectl create rolebinding node-exporter-psp-privileged \
  --namespace monitoring \
  --clusterrole=suse:caasp:psp:privileged \
  --serviceaccount=monitoring:prometheus-node-exporter
```

2. Deploy

We need to configure the storage for our deployment.

Choose among the options and uncomment the line in the config file.

**Mandatory for production: Persistent Storage**

* Use an existing PersistentVolumeClaim
* Use a StorageClass **(prefered)**


* Create a file *prometheus-config-values.yaml* with the appropriate values.

```yaml
# Alertmanager configuration
alertmanager:
  enabled: true
  ingress:
    enabled: true
    hosts:
    -  prometheus-alertmanager.example.com
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: prometheus-basic-auth
      nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    tls:
      - hosts:
        - prometheus-alertmanager.example.com
        secretName: monitoring-tls
  persistentVolume:
    enabled: true
    ## Use a StorageClass
    storageClass: my-storage-class
    ## Create a PersistentVolumeClaim of 2Gi
    size: 2Gi
    ## Use an existing PersistentVolumeClaim (my-pvc)
    #existingClaim: my-pvc

## AlertManager is configured through alertmanager.yml. This file and any others
## listed in alertmanagerFiles will be mounted into the alertmanager pod.
## See configuration options https://prometheus.io/docs/alerting/configuration/
#alertmanagerFiles:
#  alertmanager.yml:

# Create a specific service account
serviceAccounts:
  nodeExporter:
    name: prometheus-node-exporter

# Allow scheduling of node-exporter on master nodes
nodeExporter:
  hostNetwork: false
  hostPID: false
  tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule

# Disable Pushgateway
pushgateway:
  enabled: false

# Prometheus configuration
server:
  ingress:
    enabled: true
    hosts:
    - prometheus.example.com
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: prometheus-basic-auth
      nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    tls:
      - hosts:
        - prometheus.example.com
        secretName: monitoring-tls
  persistentVolume:
    enabled: true
    ## Use a StorageClass
    storageClass: my-storage-class
    ## Create a PersistentVolumeClaim of 8Gi
    size: 8Gi
    ## Use an existing PersistentVolumeClaim (my-pvc)
    #existingClaim: my-pvc

## Prometheus is configured through prometheus.yml. This file and any others
## listed in serverFiles will be mounted into the server pod.
## See configuration options 
## https://prometheus.io/docs/prometheus/latest/configuration/configuration/
#serverFiles:
#  prometheus.yml:
```

* Deploy the upstream helm chart and pass our configuration values file.

```console
$ helm install --name prometheus stable/prometheus \
  --namespace monitoring \
  --values prometheus-config-values.yaml
```

=> We need these pods running (3 node-exporter pods because we have 3 nodes)

```console
$ kubectl -n monitoring get po | grep prometheus
NAME                                             READY     STATUS    RESTARTS   AGE
prometheus-alertmanager-5487596d54-kcdd6         2/2       Running   0          2m
prometheus-kube-state-metrics-566669df8c-krblx   1/1       Running   0          2m
prometheus-node-exporter-jnc5w                   1/1       Running   0          2m
prometheus-node-exporter-qfwp9                   1/1       Running   0          2m
prometheus-node-exporter-sc4ls                   1/1       Running   0          2m
prometheus-server-6488f6c4cd-5n9w8               2/2       Running   0          2m
```


3. Configure Authentication

We need to create a basic-auth secret so the Nginx Ingress Controller can perform
authentication.

* Install htpasswd

```console
$ sudo zypper in apache2-utils
```

* Create secret file *auth*

```console
htpasswd -c auth admin
New password: 
Re-type new password: 
Adding password for user admin
```

* Create secret in Kubernetes

```console
$ kubectl create secret generic -n monitoring prometheus-basic-auth --from-file=auth
```

It is very important that the file name is **auth** so the key in the secret will be name auth
because the ingress controller will look for that key.


=> At this stage, the Prometheus Expression browser/API is accessible, depending on
you network configuration at https://prometheus.example.com or
https://prometheus.example.com:30443



## Grafana

1. Configure provisoning

Starting from Grafana 5.0, it is possible to dynamically provision the data sources 
and dashbords via files. In Kubernetes, these files are provided via the utilization
of ConfigMap, editing a ConfigMap will result by the modification of the configuration 
without having to delete/recreate the pod.

* Create the default datasource configuration file *grafana-datasources.yaml* which point to our Prometheus server 

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: grafana-datasources
  namespace: monitoring
  labels:
     grafana_datasource: "1"
data:
  datasource.yaml: |-
    apiVersion: 1
    deleteDatasources:
      - name: Prometheus
        orgId: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local:80
      access: proxy
      orgId: 1
      isDefault: true
```

* Create the ConfigMap in Kubernetes

```console
$ kubectl create -f grafana-datasources.yaml
```

* This repo provides a demo dashboard to visualize the resources of the 
Kubernetes nodes leveraging Prometheus node-exporter metrics only. (optional)

```console
$ cat grafana-dashboards-caasp* | kubectl apply -f -
```

2. Deploy stack

We need to configure the storage for our deployment.

Choose among the options and uncomment the line in the config file.

**Mandatory for production: Persistent Storage**

* Use an existing PersistentVolumeClaim
* Use a StorageClass **(prefered)**


* Create a file *grafana-config-values.yaml* with the appropriate values.

```yaml
# Configure admin password
adminPassword: 3357b57b-25a1-4780-ac78-8b77fd7ed9e1

# Ingress configuration
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - grafana.example.com
  tls:
    - hosts:
      - grafana.example.com
      secretName: monitoring-tls

# Configure persistent storage
persistence:
  enabled: true
  accessModes: 
    - ReadWriteOnce
  ## Use a StorageClass
  storageClassName: my-storage-class
  ## Create a PersistentVolumeClaim of 10Gi
  size: 10Gi
  ## Use an existing PersistentVolumeClaim (my-pvc)
  #existingClaim: my-pvc

# Enable sidecar for provisioning
sidecar:
  datasources:
    enabled: true
    label: grafana_datasource
  dashboards:
    enabled: true
    label: grafana_dashboard
```

* Deploy the upstream helm chart and pass our configuration values file.

```console
$ helm install --name grafana stable/grafana \
  --namespace monitoring \
  --values grafana-config-values.yaml
```

=> We need these pods running (3 node-exporter pods because we have 3 nodes)

```console
$ kubectl -n monitoring get po | grep grafana
NAME                                             READY     STATUS    RESTARTS   AGE
grafana-dbf7ddb7d-fxg6d                          3/3       Running   0          2m
```


=> At this stage, Grafana is accessible, depending on
you network configuration at https://grafana.example.com or
https://grafana.example.com:30443
 
 


