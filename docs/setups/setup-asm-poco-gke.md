# Install Google Service Mesh and Policy Controller on GKE

As prerequisites, you need to have these tools installed:
- [`gcloud`](https://cloud.google.com/sdk/docs/install)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl)

## Set current project

```
PROJECT_ID=FIXME
gcloud config set project ${PROJECT_ID}
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')
```

## Create Kubernetes cluster

Create a GKE cluster:
```bash
CLUSTER_NAME=asm-poco-demo
CLUSTER_ZONE=northamerica-northeast1

gcloud services enable container.googleapis.com

gcloud container clusters create ${CLUSTER_NAME} \
    --zone ${CLUSTER_ZONE} \
    --machine-type e2-standard-4 \
    --num-nodes 4 \
    --workload-pool ${PROJECT_ID}.svc.id.goog \
    --labels mesh_id=proj-${PROJECT_NUMBER}

gcloud services enable gkehub.googleapis.com

gcloud container fleet memberships register ${CLUSTER_NAME} \
    --gke-cluster ${CLUSTER_ZONE}/${CLUSTER_NAME} \
    --enable-workload-identity
```

## Enable Google Service Mesh

```bash
gcloud services enable mesh.googleapis.com

gcloud container fleet mesh enable

gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME}
```

## Enable Policy Controller

```bash
gcloud beta container fleet config-management enable

cat <<EOF > acm-config.yaml
applySpecVersion: 1
spec:
  policyController:
    enabled: true
    templateLibraryInstalled: false
    referentialRulesEnabled: true
    auditIntervalSeconds: 5
EOF
gcloud beta container fleet config-management apply \
    --membership ${CLUSTER_NAME} \
    --config acm-config.yaml
```

## (Optional) Enable Config Sync

```bash
cat <<EOF > acm-config.yaml
applySpecVersion: 1
spec:
  configSync:
    enabled: true
  policyController:
    enabled: true
    templateLibraryInstalled: false
    referentialRulesEnabled: true
EOF
gcloud beta container fleet config-management apply \
    --membership ${CLUSTER_NAME} \
    --config acm-config.yaml
```
