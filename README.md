# Prometheus + Loki + Grafana based Log Analytics and Metrics for JFrog Artifactory, Xray

The JFrog Log Analytics and Metrics solution using Prometheus consists of three segments,

1. Prometheus - the component where metrics data gets ingested
2. Loki - the component where log data gets ingested
3. Grafana - the component where data visualization is achieved via prebuilt dashboards

## Prerequisites
1. Working and configured Kubernetes Cluster - Amazon EKS / Google GKE / Azure AKS / Docker Desktop / Minikube
   * Recommended Kubernetes Version 1.25.2 and above
   * For Google GKE, refer [GKE Guide](https://cloud.google.com/kubernetes-engine/docs/how-to)
   * For Amazon EKS, refer [EKS Guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
   * For Azure AKS, refer [AKS Guide](https://docs.microsoft.com/en-us/azure/aks/)
   * For Docker Desktop and Kubernetes, refer [DOCKER Guide](https://docs.docker.com/desktop/kubernetes/)
2. 'kubectl' utility on the workstation which is capable of connecting to the Kubernetes cluster
   * For Installation and usage refer [KUBECTL Guide](https://kubernetes.io/docs/tasks/tools/)
3. HELM v3 Installed
   * For Installation and usage refer [HELM Guide](https://helm.sh/docs/intro/install/)
4. Versions supported and Tested:

   * Jfrog Platform: 10.9.2
   * Artifactory : 7.46.10
   * Xray : 3.59.4
   * Prometheus:2.39.1
   * Grafana:9.2.3
   * Loki: 2.6.1


## Read me before installing
** Important Note**: This version replaces all previous implementations. This version is not an in-place upgrade to the existing solution from JFrog but is a full reinstall. Any dashboard customizations done on previous versions will need to be redone after this install.
```html
This guide assumes the implementer is performing new setup, Changes to handle install in an existing setup will be highlighted where applicable.
    if prometheus is already installed and configured, we recommend to have the existing prometheus release name handy.
    If Loki is already installed and configured, we recommend to have its service URL handy.


```
If prometheus and loki are already available you can skip the installation section and proceed to [Configuration Section](#Configuration).


# Installation

## Installing Prometheus, Loki and Grafana on Kubernetes
The Prometheus Community [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart allows the creation of Prometheus instances and includes Grafana.
The Grafana Community [grafana](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart allows the creation of Loki instances and includes Grafana which can link to prometheus.

Once the Pre-Requisites are met, to install Prometheus Kubernetes stack:

#### Create the Namespace required for Prometheus Stack deployment

```
kubectl create namespace jfrog-plg
```
We will use jfrog-plg as the namespace throughout this document.
#### Install the Prometheus chart:

Add the required Helm Repositories:
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Install the chart:
```shell
helm install "prometheus" prometheus-community/kube-prometheus-stack -n jfrog-plg
```
```yaml
* "prometheus" here is the value that needs to be used against the value for "release_name" in the configuration section
```
For Docker Desktop, run this additional command to correct the mount path propagation for prometheus node-exporter component,
An error event will be appearing as follows "Error: failed to start container "node-exporter": Error response from daemon: path / is mounted on / but it is not a shared or slave mount"
```shell
kubectl patch ds prometheus-prometheus-node-exporter --type "json" -p '[{"op": "remove", "path" : "/spec/template/spec/containers/0/volumeMounts/2/mountPropagation"}]' -n jfrog-plg
```

#### Install the Loki chart:

Add the required Helm Repositories:
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
```
helm repo add loki https://grafana.github.io/loki/charts
helm repo update
```
Install the chart:
```shell
helm upgrade --install "loki" grafana/loki-stack -n jfrog-plg
```
```yaml
* "loki" will be the service name, the url to access loki as a datasource can be visualised as http://<service_name>.<namespace>:<port>
      ex: http://loki.jfrog-plg:3100 will be the "loki_url" value

* the --version v2.6.1 is the most recent loki version at the time of writing the document
      if there is a need to deploy this exact version, then add "--version v2.6.1" to the command line.

```

# Configuration

The configuration which is needed to be put before the JFrog products installation can take place are listed below,

From any of the value files for applying charts i.e in `helm/jfrog-platform-values.yaml`, `helm/artifactory-values.yaml`, `helm/artifactory-ha-values.yaml` or `helm/xray-values.yaml`
download and provide the following values for `global.prometheus.loki_url` and `global.prometheus.release_name` as per the installation if prometheus and loki are already installed

```yaml
global:
   jfrog:
      observability:
         branch: master
   prometheus:
      loki_url: http:\/\/loki.jfrog-plg:3100
      release_name: prometheus
```


## JFrog Platform + Metrics via Helm ⎈

Ensure Jfrog repo is added to helm. 

```
helm repo add jfrog https://charts.jfrog.io
helm repo update
```

To configure and install JFrog Platform with Prometheus metrics being exposed use our file `helm/jfrog-platform-values.yaml` to expose a metrics and new service monitor to Prometheus.

JFrog Platform ⎈:
```text
helm upgrade --install jfrog-platform jfrog/jfrog-platform \
       -f helm/jfrog-platform-values.yaml \
       -n jfrog-plg
```
**If you are installing in the same cluster with the deprecated solution, Use the same namespace as the previous one instead of jfrog-plg above.**

## Artifactory / Artifactory HA + Metrics via Helm ⎈

For configuring and installing Artifactory Pro/Pro-x use the `artifactory-values.yaml` file.

For configuring and installing Enterprise/Ent+ use the `artifactory-ha-values.yaml` file.

You can apply them to your helm install examples below:

Artifactory ⎈:
```text
helm upgrade --install artifactory jfrog/artifactory \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/artifactory-values.yaml \
       -n jfrog-plg
```
**If you are installing in the same cluster with the deprecated solution, Use the same namespace as the previous one instead of jfrog-plg above.**

Artifactory-HA ⎈:
```text
helm upgrade --install artifactory-ha jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/artifactory-ha-values.yaml \
       -n jfrog-plg
```
**If you are installing in the same cluster with the deprecated solution, Use the same namespace as the previous one instead of jfrog-plg above.**
Note the above examples are only references you will need additional parameters to configure TLS, binary blob storage, or other common Artifactory features.

This will complete the necessary configuration for Artifactory and expose a new service monitor `servicemonitor-artifactory` to expose metrics to Prometheus.

## Xray + Metrics via Helm ⎈

To configure and install Xray with Prometheus metrics being exposed use our file `helm/xray-values.yaml` to expose a metrics and new service monitor to Prometheus.

Xray ⎈:
```text
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/xray-values.yaml \
       -n jfrog-plg
```
**If you are installing in the same cluster with the deprecated solution, Use the same namespace as the previous one instead of jfrog-plg above.**
## Grafana Dashboard
Example dashboards are included in the [grafana directory](grafana). These dashboards needs to be imported to the grafana. These include:


- Artifactory Metrics and Log Analytics Dashboard [Download Here](grafana/ArtifactoryLogAnalyticsAndSystemMetrics.json)
- Xray Metrics and Log Analytics Dashboard [Download Here](grafana/XrayLogAnalyticsAndSystemMetrics.json)

## Assess the setup for working

Use 'kubectl port forwards' as mentioned in two terminal windows
```
   kubectl port-forward service/prometheus-operated 9090:9090 -n jfrog-plg
   kubectl port-forward service/prometheus-grafana 3000:80 -n jfrog-plg
```

1. Go to the web UI of the Prometheus instance "http://localhost:9090" and verify "Status -> Service Discovery", the list shows the new ServiceMonitor for Artifactory or Xray or Both.



![targets](images/ServiceDiscovery.jpeg)
__
2. Go to Grafana "http://localhost:3000" to add your Prometheus instance and Loki Instance as a datasource.
   While specifying datasource url for Loki and Prometheus,please test and confirm that the connection is successful.

For Loki, specify url value as http://loki:3100

For Prometheus specify url value as http://prometheus-kube-prometheus-prometheus.jfrog-plg:9090/

You can get the url content from the command ```kubectl get svc -n jfrog-plg```

```
   Default credentials (UNAME / PASSWD) for Prometheus grafana is ->  "admin" / "prom-operator"
```

![datasource](images/DataSourceAddition-01.jpeg)
![datasource](images/DataSourceAddition-02.jpeg)

3. Finally in the Grafana, import the Dashboards and select the appropriate sources.


![dashboards](images/DashboardAddition-01.jpeg)
![dashboards](images/DashboardAddition-02.jpeg)
![dashboards](images/DashboardAddition-03.jpeg)

## References
* [Grafana Dashboards](https://grafana.com/docs/grafana/latest/features/dashboard/dashboards/)
* [Grafana Queries](https://prometheus.io/docs/prometheus/latest/querying/basics/)
* [Loki Queries](https://grafana.com/docs/loki/latest/logql/)
