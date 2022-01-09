# lsc-k8s-scaling

Purpose of this repository is a comparison of three kubernetes metrics storages - **InfluxDb**, **Prometheus** and **Graphite**.

**Requirements**
- minikube - `>= 1.24.0`
- kubectl
- helm

## setup - k8s cluster

Only for demonstration purposes we 

```console
$ minikube version      
minikube version: v1.24.0
commit: 76b94fb3c4e8ac5062daf70d60cf03ddcc0a741b

$ minikube start -p lsc
(...)
$ kubectl cluster-info
Kubernetes master is running at https://192.168.49.2:8443
KubeDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ minikube -p lsc ssh
Last login: Sun Jan  9 19:20:59 2022 from 192.168.49.1
docker@lsc:~$ sudo chmod 666 /var/run/docker.sock
```
# Grafana

```console
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm install grafana -n monitoring grafana/grafana
```

Fetch secret - admin password:

```console
$ kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

# InfluxDb

// TODO: Telegraf


// lineprotocol - https://docs.influxdata.com/influxdb/cloud/reference/syntax/line-protocol/


// HA kosztuje $$ https://docs.influxdata.com/influxdb/v1.8/guides/hardware_sizing/#single-node-or-cluster

//duże wymagania startowe https://docs.influxdata.com/influxdb/v1.8/guides/hardware_sizing/#influxdb-oss-guidelines

```console
$ helm repo add influxdata https://helm.influxdata.com
$ helm upgrade -i influxdb influxdata/influxdb
```
##

port-forward
```console
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

kubectl --namespace default port-forward $POD_NAME 3000
```

## Telegraf

Telegraf installation
```console
$ KUBERNETES_HOST=$(minikube -p lsc ip)
$ helm upgrade --install -n monitoring telegraf-ds \
--set daemonset.config.outputs.influxdb.urls={http://influxdb.monitoring:8086} \
--set daemonset.config.inputs.kubernetes.url=http://${KUBERNETES_HOST}:10255 \
influxdata/telegraf-ds
```

Metrics are being pulled from docker socket - telegraf is running as `DaemonSet`

```console
kubectl describe cm -n monitoring telegraf-ds | grep -C 1 docker

[[inputs.docker]]
endpoint = "unix:///var/run/docker.sock"
```

## verify if metrics are sunk into InfluxDb

from inside of InfluxDb container find if the **telegraf** database has been created

```console
$ influx
Connected to http://localhost:8086 version 1.8.6
InfluxDB shell version: 1.8.6
> SHOW databases
name: databases
name
----
_internal
telegraf
```

Example queries on metrics data:

```sql
$ influx
Connected to http://localhost:8086 version 1.8.6
InfluxDB shell version: 1.8.6
> SELECT usage_percent, container_name FROM docker_container_mem LIMIT 15;
name: docker_container_mem
time                usage_percent         container_name
----                -------------         --------------
1641756151000000000 0.0014785347305364344 k8s_POD_kube-proxy-rbx9v_kube-system_68308d73-1217-4bfb-be6d-e52b6e30a601_0
1641756152000000000 0.0007575963082087515 k8s_POD_coredns-78fcd69978-2jspd_kube-system_f59622d8-7008-4447-99f8-2c6ba56d0cb9_0
1641756152000000000 0.0005376489929223398 k8s_POD_etcd-lsc_kube-system_0718f35ce7473bbbfbc9b019ef460259_0
1641756152000000000 0.0008186927846771992 k8s_POD_influxdb-0_default_8cee725a-c6c2-45e4-bd91-d94f868c729a_0
1641756152000000000 0.0005376489929223398 k8s_POD_kube-apiserver-lsc_kube-system_f732a34dfa98aa302cd4bdff70682bd2_0
1641756152000000000 0.0005376489929223398 k8s_POD_kube-controller-manager-lsc_kube-system_6c4baaed1d9cc46dd4fadddb7663c040_0
1641756152000000000 0.0005376489929223398 k8s_POD_kube-scheduler-lsc_kube-system_5adae6cb6dff4ae67658d85a4f4ae4fe_0
1641756152000000000 0.0005376489929223398 k8s_POD_storage-provisioner_kube-system_c9e1ca46-335a-4093-bd65-4d4aaec9e639_0
1641756152000000000 0.0008186927846771992 k8s_POD_telegraf-ds-w7xwh_default_4009ed2d-43a5-46d6-ae48-f221224b5b0d_0
1641756152000000000 7.0657169117647065    k8s_coredns_coredns-78fcd69978-2jspd_kube-system_f59622d8-7008-4447-99f8-2c6ba56d0cb9_0
1641756152000000000 0.10777418449034175   k8s_etcd_etcd-lsc_kube-system_0718f35ce7473bbbfbc9b019ef460259_0
1641756152000000000 0.04622559409602753   k8s_influxdb_influxdb-0_default_8cee725a-c6c2-45e4-bd91-d94f868c729a_0
1641756152000000000 0.6919053767098765    k8s_kube-apiserver_kube-apiserver-lsc_kube-system_f732a34dfa98aa302cd4bdff70682bd2_0
1641756152000000000 0.1438211056067259    k8s_kube-controller-manager_kube-controller-manager-lsc_kube-system_6c4baaed1d9cc46dd4fadddb7663c040_0
1641756152000000000 0.032992097292961764  k8s_kube-proxy_kube-proxy-rbx9v_kube-system_68308d73-1217-4bfb-be6d-e52b6e30a601_0
> SELECT mean(usage_percent) FROM docker_container_mem WHERE container_name='k8s_influxdb_influxdb-0_default_8cee725a-c6c2-45e4-bd91-d94f868c729a_0' LIMIT 15;

name: docker_container_mem
time mean
---- ----
0    0.4698723762118421

```

Or to verify if the node is healthy

```console
$ apk add curl jq
$ curl -XGET "localhost:8086/health" | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   106  100   106    0     0  97069      0 --:--:-- --:--:-- --:--:--  103k
{
  "checks": [],
  "message": "ready for queries and writes",
  "name": "influxdb",
  "status": "pass",
  "version": "1.8.6"
}
```

## Setup Grafana dashboards 

```console
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 300
```

![](img/influxdb-data-source.png)

Import example Docker [dashboard](https://grafana.com/grafana/dashboards/1150)

![](img/grafana-dashboard.png)

![](img/influxdb-dashboards.png)


# Prometheus

# Graphite
