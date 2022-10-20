# Install Istio and Gatekeeper on GKE

As prerequisites, you need to have these tools installed:
- [`gcloud`](https://cloud.google.com/sdk/docs/install)
- [`istioctl`](https://istio.io/latest/docs/setup/install/istioctl/)
- [`helm`](https://helm.sh/docs/intro/install/)

## Set current project

```
PROJECT_ID=FIXME
gcloud config set project ${PROJECT_ID}
```

## Create Kubernetes cluster

Create a GKE cluster:
```bash
CLUSTER_NAME=istio-gatekeeper-demo
CLUSTER_ZONE=northamerica-northeast1

gcloud services enable container.googleapis.com

gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_ZONE} \
    --machine-type e2-standard-4 \
    --num-nodes 4
```

## Install Istio

Install Istio in your cluster:
```bash
istioctl install \
    --set profile=minimal \
    -y
```

## Install Gatekeeper

Install Gatekeeper in your cluster:
```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper \
    --name-template=gatekeeper \
    --namespace gatekeeper-system \
    --create-namespace \
    --set auditInterval=5
```

## (Optional) Install Config Sync

_Coming, stay tuned!_
