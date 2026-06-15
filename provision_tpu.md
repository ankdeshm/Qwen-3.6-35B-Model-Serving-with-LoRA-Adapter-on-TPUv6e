# Create Flex-Start Node Pool

```bash
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)")
export ZONE=us-east5-a
export REGION=${ZONE%-*}
export CLUSTER_NAME=ank-tpu-multi-host-serveing
export GVNIC_NETWORK_PREFIX=ank-tpu-serving
export BUCKET_NAME=ank-qwen-serving-${PROJECT_NUMBER}
export NODE_POOL_NAME=ank-tpu-multi-host-pool #16 chip node pool
export MULTIHOST_COLLECTION_NAME="tpu-6-collection"
export HF_TOKEN=<YOUR_HF_TOKEN>
kubectl create secret generic hf-token --from-literal=HF_TOKEN=${HF_TOKEN}
```

### TPUv6e-8 nodepool

```bash
export NODE_POOL_NAME=ank-tpu-v6e8-pool

gcloud beta container node-pools create ${NODE_POOL_NAME} \
    --project=${PROJECT_ID} \
    --cluster=${CLUSTER_NAME} \
    --region=${REGION} \
    --node-locations=${ZONE} \
    --machine-type=ct6e-standard-8t \
    --num-nodes=0 \
    --flex-start \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=1 \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --reservation-affinity=none \
    --accelerator-network-profile=auto \
    --node-labels=cloud.google.com/gke-nodepool-group-name=${MULTIHOST_COLLECTION_NAME} \
    --node-labels=cloud.google.com/gke-workload-type=HIGH_AVAILABILITY \
    --node-labels=cloud.google.com/gke-networking-dra-driver=true
```

### TPUv6e-16 multi-host nodepool

```bash
export NODE_POOL_NAME=ank-tpu-multi-host-pool

gcloud beta container node-pools create ${NODE_POOL_NAME} \
    --project=${PROJECT_ID} \
    --cluster=${CLUSTER_NAME} \
    --region=${REGION} \
    --node-locations=${ZONE} \
    --machine-type=ct6e-standard-4t \
    --tpu-topology=4x4 \
    --num-nodes=0 \
    --flex-start \
    --enable-queued-provisioning \
    --enable-autoscaling \
    --min-nodes=0 \
    --max-nodes=4 \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --reservation-affinity=none \
    --accelerator-network-profile=auto \
    --node-labels=cloud.google.com/gke-nodepool-group-name=${MULTIHOST_COLLECTION_NAME} \
    --node-labels=cloud.google.com/gke-workload-type=HIGH_AVAILABILITY \
    --node-labels=cloud.google.com/gke-networking-dra-driver=true
```


# Create Provisioning Request

Required by v6e-16 config since multi-host config and needs all chips to be available at once

```bash
nano tpu-v6e-pr.yaml
```

```bash
apiVersion: v1
kind: PodTemplate
metadata:
  name: tpu-v6e-16-template
  namespace: default
template:
  spec:
    nodeSelector:
      cloud.google.com/gke-tpu-topology: 4x4
      cloud.google.com/gke-tpu-accelerator: tpu-v6e-slice
    tolerations:
    - key: "google.com/tpu"
      operator: "Exists"
      effect: "NoSchedule"
    - key: "cloud.google.com/gke-queued"
      operator: "Exists"
      effect: "NoSchedule"
    containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
      resources:
        requests:
          google.com/tpu: 4
          memory: "100Gi"
        limits:
          google.com/tpu: 4
          memory: "100Gi"
---
apiVersion: autoscaling.x-k8s.io/v1beta1
kind: ProvisioningRequest
metadata:
  name: tpu-v6e-16-pr
  namespace: default
spec:
  provisioningClassName: queued-provisioning.gke.io
  parameters:
    maxRunDurationSeconds: "86400"
  podSets:
  - count: 4
    podTemplateRef:
      name: tpu-v6e-16-template
```


```bash
kubectl apply -f tpu-v6e-pr.yaml
kubectl get ProvisioningRequest tpu-v6e-16-pr -w
kubectl describe ProvisioningRequest tpu-v6e-16-pr 
kubectl get nodes
```
