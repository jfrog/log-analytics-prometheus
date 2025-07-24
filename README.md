# Prometheus, Loki and Grafana Log Collection and Metrics for JFrog Artifactory and Xray

The JFrog Log Analytics and Metrics solution using Prometheus consists of three segments,

1. Prometheus - the component where metrics data gets ingested
2. Loki - the component where log data gets ingested
3. Grafana - the component where data visualization is achieved via prebuilt dashboards

## Pre-Requisites

1. A Kubernetes Cluster - Amazon EKS / Google GKE / Azure AKS / Docker Desktop / Minikube
   1. Recommended Kubernetes Version 1.25.2 and above
   2. For Google GKE, refer [GKE Guide](https://cloud.google.com/kubernetes-engine/docs/how-to)
   3. For Amazon EKS, refer [EKS Guide](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
   4. For Azure AKS, refer [AKS Guide](https://docs.microsoft.com/en-us/azure/aks/)
   5. For Docker Desktop and Kubernetes, refer [Docker Guide](https://docs.docker.com/desktop/kubernetes/)

2. `kubectl` configured to the Kubernetes cluster
   1. For Installation and usage refer [kubectl setup](https://kubernetes.io/docs/tasks/tools/#kubectl)

3. `helm` v3
   1. For Installation and usage refer [helm setup](https://helm.sh/docs/intro/install/)

4. Versions supported and Tested:
   1. Artifactory: 7.117.x
   2. Xray: 3.118.x
   3. Prometheus: 3.5.x
   4. Grafana: 12.0.x
   5. Loki: 3.5.x

## Read This Before Installing

### Important Note: This version replaces all previous implementations. This version is not an in-place upgrade to the existing solution from JFrog but is a full reinstall. Any dashboard customizations done on previous versions will need to be redone.

```
This guide assumes the implementer is performing new setup. Changes to handle install in an existing setup will be highlighted where applicable.
If Prometheus is already installed and configured, we recommend to have the existing Prometheus release name handy.
If Loki is already installed and configured, we recommend to have its service URL handy.
```

If Prometheus and Loki are already available you can skip the installation section and proceed to [Configuration Section](#Configuration).

> [!WARNING]
>
> The old docker registry `partnership-pts-observability.jfrog.io`, which contains older versions of this integration is now deprecated. We'll keep the existing docker images on this old registry until August 1st, 2024. After that date, this registry will no longer be available. Please `helm upgrade` your JFrog kubernetes deployment in order to pull images as specified on the above helm value files, from the new `releases-pts-observability-fluentd.jfrog.io` registry. Please do so in order to avoid `ImagePullBackOff` errors in your deployment once this registry is gone.

# Installation

## Installing Prometheus, Grafana and Loki

The Prometheus Community [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart allows the creation of Prometheus instances and includes Grafana.
The Grafana Community [grafana](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) helm chart allows the creation of Loki instances and includes Grafana which can link to prometheus.

Once the Pre-Requisites are met, to install Prometheus Kubernetes stack:

1. Create the namespaces required for the Kubernetes deployments
   1. We use the `jfrog` namespace for the JFrog applications
   2. We use the `monitoring` namespace for the observability tools

```shell
export JFROG_NAMESPACE=jfrog
kubectl create namespace ${JFROG_NAMESPACE}

export OBS_NAMESPACE=monitoring
kubectl create namespace ${OBS_NAMESPACE}
```

Note: The `monitoring` namespace is also used in the Loki configuration in [artifactory-valies.yaml](helm/artifactory-values.yaml) and [xray-values.yaml](helm/xray-values.yaml). If you decide to change it, make sure to update these files (the `LOKI_URL` variable).

2. Install Prometheus and Grafana

```shell
# Add the required Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```shell
# Install the kube-prometheus-stack chart
helm upgrade --install prometheus --values helm/prometheus-grafana-values.yaml prometheus-community/kube-prometheus-stack -n ${OBS_NAMESPACE}

# Might need to add --set prometheus.prometheusSpec.maximumStartupDurationSeconds=600 to avoid an error (bug?)
```

3. For Docker Desktop

   Run this additional command to correct the mount path propagation for prometheus node-exporter component.

   An error event will be appearing as follows "Error: failed to start container "node-exporter": Error response from daemon: path / is mounted on / but it is not a shared or slave mount"

```shell
kubectl patch ds prometheus-prometheus-node-exporter --type json -p '[{"op": "remove", "path" : "/spec/template/spec/containers/0/volumeMounts/2/mountPropagation"}]' -n ${OBS_NAMESPACE}
```

4. Install Loki

```shell
# Add the required Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```shell
# Install the Loki chart
helm upgrade --install loki --values helm/loki-values.yaml grafana/loki -n ${OBS_NAMESPACE}
```

## Install Artifactory with Open Metrics

### Artifactory
Installing Artifactory using the official [Helm Chart](https://github.com/jfrog/charts/tree/master/stable/artifactory)

1. Before starting the Artifactory installation generate a join and master keys

```shell
export JOIN_KEY=$(openssl rand -hex 32)
export MASTER_KEY=$(openssl rand -hex 32)
```

2. Install Artifactory (using the generated join and master keys)

```shell
# Install Artifactory
helm upgrade --install artifactory jfrog/artifactory \
     --set artifactory.masterKey=${MASTER_KEY} \
     --set artifactory.joinKey=${JOIN_KEY} \
     --set artifactory.metrics.enabled=true \
     -n ${JFROG_NAMESPACE}
```

:bulb: Open Metrics is disabled by default in Artifactory. You enable it by setting `artifactory.metrics.enabled=true`.

3. Follow the instructions how to get your new Artifactory URL from the helm install output

```shell
export SERVICE_IP=$(kubectl get svc --namespace ${JFROG_NAMESPACE} artifactory-artifactory-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# OR
export SERVICE_IP=$(kubectl get svc --namespace ${JFROG_NAMESPACE} artifactory-artifactory-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo ${SERVICE_IP}

# URL to access Artifactory
echo "http://${SERVICE_IP}/"
```

4. Browse to the URL above and login to Artifactory with the default credentials: `admin`/`password`
   1. Follow initial setup wizard
   2. You will need to enter a valid Artifactory license. If needed, get a free trial license from [here](https://jfrog.com/start-free/)

5. In the Artifactory UI, go to "Administration" -> "User Management" -> "Access Tokens" and generate an [admin access token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory). Using the generated token, create a Kubernetes generic secret for the token - using one of the following methods

```shell
kubectl create secret generic jfrog-admin-token --from-file=token=<path_to_token_file> -n ${JFROG_NAMESPACE}
```
OR
```shell
kubectl create secret generic jfrog-admin-token --from-literal=token=<JFROG_ADMIN_TOKEN> -n ${JFROG_NAMESPACE}
```

6. The PostgreSQL password is required for Artifactory upgrade. Run the following command to get the current PostgreSQL password
```shell
export POSTGRES_PASSWORD=$(kubectl get secret -n ${JFROG_NAMESPACE} artifactory-postgresql -o jsonpath="{.data.postgres-password}" | base64 --decode)
echo ${POSTGRES_PASSWORD}
```

7. Upgrade Artifactory with the custom values in [helm/artifactory-values.yaml](helm/artifactory-values.yaml) to create additional Kubernetes resources, which are required for the Prometheus service discovery process.

```shell
# Upgrade Artifactory
helm upgrade --install artifactory jfrog/artifactory \
     --set artifactory.joinKey=${JOIN_KEY} \
     --set databaseUpgradeReady=true --set postgresql.auth.password=${POSTGRES_PASSWORD} \
     -f helm/artifactory-values.yaml \
     -n ${JFROG_NAMESPACE}
```

This will complete the necessary configuration for Artifactory and expose new service monitors `servicemonitor-artifactory` and `servicemonitor-observability` to expose metrics to Prometheus

## Install Xray with Open Metrics

To configure and install Xray with Prometheus metrics being exposed use our file `helm/xray-values.yaml` to expose a metrics and new service monitor to Prometheus.

### Xray

1. Generate a master key for the Xray installation:

```shell
export XRAY_MASTER_KEY=$(openssl rand -hex 32)
```

2. Use the same `JOIN_KEY` from the Artifactory installation, in order to connect Xray to Artifactory. You'll also be using the `jfrog-admin-token` kubernetes secret, that was created earlier as part of Artifactory installation

```shell
# Getting the Artifactory URL
export JFROG_URL=$(kubectl get svc -n ${JFROG_NAMESPACE} artifactory-artifactory-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# OR
export JFROG_URL=$(kubectl get svc -n ${JFROG_NAMESPACE} artifactory-artifactory-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "http://${JFROG_URL}"

# Install xray
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://${JFROG_URL} \
     --set xray.masterKey=${XRAY_MASTER_KEY} \
     --set xray.joinKey=${JOIN_KEY} \
     -f helm/xray-values.yaml \
     -n ${JFROG_NAMESPACE}
```

# Configuration

## Access the Prometheus UI

Use `kubectl port-forward` as mentioned below in a separate terminal window

```shell
kubectl port-forward service/prometheus-operated 9090:9090 -n ${JFROG_NAMESPACE}
```

Go to the web UI of the Prometheus instance http://localhost:9090 and verify "Status -> Service Discovery", the list shows all the `serviceMonitor`s.

Search for `servicemonitor-artifactory` and `servicemonitor-xray` to confirm they are successfully picked up by Prometheus.

## Complete the Grafana Setup

Use `kubectl port-forward` as mentioned below in a separate terminal window

```shell
kubectl port-forward service/prometheus-grafana 3000:80 -n ${JFROG_NAMESPACE}
```

1. Open your Grafana on a browser at http://localhost:3000. Grafana default credentials are `admin/prom-operator` (set in [prometheus-grafana-values.yaml](helm/prometheus-grafana-values.yaml)).

2. Go to "Data sources" on the sidebar menu

3. Click `Add new data source`
   1. Add your Prometheus as datasources (if not already configured): Set "Prometheus server URL" to `http://prometheus-kube-prometheus-prometheus:9090/`
   2. Add your Loki as datasources: Set "URL" to `http://loki:3100`

4. When adding the `Loki` and `Prometheus` datasources, click `Save & Test` button at the bottom to validate connection to services is successful

## Artifactory and Xray Grafana Dashboards

Example dashboards are included in the [grafana](grafana) directory. These dashboards need to be imported to Grafana. These include:

- Artifactory Application Metrics (Open Metrics) Dashboard [Download Here](grafana/ArtifactoryMetrics.json)
- Xray Application Metrics (Open Metrics) Dashboard [Download Here](grafana/XrayMetrics.json)

Older (broken) dashboards
- Artifactory Metrics and Log Analytics Dashboard [Download Here](grafana/ArtifactoryLogAnalyticsAndSystemMetrics.json)
- Xray Metrics and Log Analytics Dashboard [Download Here](grafana/XrayLogAnalyticsAndSystemMetrics.json)

1. After downloading the dashboards go to "Dashboards" -> "New" -> "Import"

2. Pick `Upload dashboard JSON file` and upload Artifactory and Xray dashboards files that you downloaded in the previous step

## Artifactory and Xray Logs in Grafana

If you have Loki configured as a Grafana datasource, you will see a `Logs` link on the sidebar menu. Click it and you should see the Artifactory and Xray services with a snippet of their logs.

Click the `Show logs` on any of these services, and you can now see all the service (Artifactory or Xray) logs in Grafana and start searching and filtering through them.

## References

* [Grafana Dashboards](https://grafana.com/docs/grafana/latest/features/dashboard/dashboards/)
* [Grafana Queries](https://prometheus.io/docs/prometheus/latest/querying/basics/)
* [Loki Queries](https://grafana.com/docs/loki/latest/query/)
