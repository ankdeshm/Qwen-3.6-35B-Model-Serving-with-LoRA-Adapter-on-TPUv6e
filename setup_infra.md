# 🏗️ End-to-End Infrastructure Provisioning Workflow

Before deploying the Qwen 3.6 35B model workload, the cluster infrastructure and security perimeter must be fully established. This document guides you through the complete, end-to-end environment setup required for high-performance TPU serving. You will begin by creating a custom VPC network and subnetwork tailored for optimal inter-node TPU communication, followed by spinning up the GKE cluster control plane mapped explicitly to this networking stack. Once the cluster is live, the workflow handles pulling cluster credentials, injecting your Hugging Face API token as a Kubernetes secret, and installing the LeaderWorkerSet (LWS) orchestration operator via Helm. Finally, the setup optimizes and secures data access by provisioning GCS Rapid Cache within the local TPU zone for ultra-low latency weight loading, and binding a dedicated Google Cloud IAM Service Account to the Kubernetes Service Account using Workload Identity to enforce strict, least-privilege storage bucket access.

## 1. Set Environment Variables

```bash
export PROJECT_ID=<YOUR_PROJECT_ID>
gcloud config set project $PROJECT_ID

export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
export ZONE="<YOUR_ZONE>" # e.g., us-east5-a
export REGION=${ZONE%-*}
export CLUSTER_NAME="qwen-serving-cluster"
export GVNIC_NETWORK_PREFIX="qwen-serving"
export BUCKET_NAME="inf-demo-model-storage-${PROJECT_NUMBER}"
export MULTIHOST_COLLECTION_NAME="tpu-6-collection"
export HF_TOKEN="<YOUR_HUGGING_FACE_TOKEN>" # Token with access to Qwen model if restricted
```


## 2. Create Custom Networking
Multi-host TPU workloads require specific network configurations, including higher MTU sizes for efficient accelerator communication. Create a custom VPC network for your cluster.

### Create the VPC network with a large MTU (8896):

```bash
gcloud compute --project=${PROJECT_ID} \
    networks create ${GVNIC_NETWORK_PREFIX}-main \
    --subnet-mode=custom \
    --mtu=8896a
```


### Create the subnet for the cluster:

```bash
gcloud compute --project=${PROJECT_ID} \
    networks subnets create ${GVNIC_NETWORK_PREFIX}-tpu \
    --network=${GVNIC_NETWORK_PREFIX}-main \
    --region=${REGION} \
    --range=192.168.100.0/24
```


### Create firewall rules allowing internal traffic to enable workers to communicate:

```bash
gcloud compute --project=${PROJECT_ID} firewall-rules create ${GVNIC_NETWORK_PREFIX}-allow-internal \
    --network=${GVNIC_NETWORK_PREFIX}-main \
    --allow=all \
    --source-ranges=172.16.0.0/12,192.168.0.0/16,10.0.0.0/8 \
    --description="Allow all internal traffic within the network."
```


## 3. Create GKE Cluster
Create a Standard GKE cluster setup configured to support GCS Fuse mounts and Ray Operator workloads.

### Create the cluster:

```bash
gcloud container clusters create ${CLUSTER_NAME} \
    --project=${PROJECT_ID} \
    --location=${REGION} \
    --release-channel=rapid \
    --machine-type=e2-standard-4 \
    --network=${GVNIC_NETWORK_PREFIX}-main \
    --subnetwork=${GVNIC_NETWORK_PREFIX}-tpu \
    --num-nodes=1 \
    --gateway-api=standard \
    --enable-managed-prometheus \
    --enable-dataplane-v2 \
    --enable-dataplane-v2-metrics \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --addons=GcsFuseCsiDriver,RayOperator \
    --enable-ip-alias
```

### Retrieve Cluster Credentials:

```bash
gcloud container clusters get-credentials ${CLUSTER_NAME} --region=${REGION}
```

### Create Hugging Face Secret: Save your token securely for container access downloads:

```bash
kubectl create secret generic hf-secret \
    --from-literal=hf_api_token=${HF_TOKEN} \
    --dry-run=client -o yaml | kubectl apply -f -
```

### Install LeaderWorkerSet (LWS) via Helm. LWS manages groups of pods that must be scheduled together:

```bash
helm install lws oci://registry.k8s.io/lws/charts/lws \
    --version=0.7.0 \
    --namespace lws-system \
    --create-namespace \
    --wait
```

## 4. Enable GCS Rapid Cache

To speed up reading dozens of GBs of weights from Cloud Storage during serving, create a GCS bucket and enable GCS Rapid Cache in your zone.

### Create the bucket:

```bash
gcloud storage buckets create gs://$BUCKET_NAME \
    --location=$REGION \
    --uniform-bucket-level-access
```

### Initialize Rapid Cache in your TPU zone:

```bash
gcloud storage buckets anywhere-caches create gs://$BUCKET_NAME $ZONE \
    --ttl=1d \
    --admission-policy=ADMIT_ON_FIRST_MISS
```

## 5. IAM & Permissions

Configure identity links to securely mount the weight bucket into your GKE pods without embedding long-lived keys.

### Create a dedicated IAM Service Account:

```bash
gcloud iam service-accounts create tpu-reader-sa
```

### Grant Bucket Read permissions:

```bash
gcloud storage buckets add-iam-policy-binding gs://${BUCKET_NAME} \
    --member="serviceAccount:tpu-reader-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
```

### Create Workload Identity Binding for the default namespace Kubernetes Service Account:

```bash
gcloud iam service-accounts add-iam-policy-binding tpu-reader-sa@${PROJECT_ID}.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[default/default]"
```

### Annotate the Kubernetes SA:

```bash
kubectl annotate serviceaccount default iam.gke.io/gcp-service-account=tpu-reader-sa@${PROJECT_ID}.iam.gserviceaccount.com
```
