# Create Custom Networking
Multi-host TPU workloads require specific network configurations, including higher MTU sizes for efficient accelerator communication. Create a custom VPC network for your cluster.

## Create the VPC network with a large MTU (8896):

```bash
gcloud compute --project=${PROJECT_ID} \
    networks create ${GVNIC_NETWORK_PREFIX}-main \
    --subnet-mode=custom \
    --mtu=8896
```


## Create the subnet for the cluster:

```bash
gcloud compute --project=${PROJECT_ID} \
    networks subnets create ${GVNIC_NETWORK_PREFIX}-tpu \
    --network=${GVNIC_NETWORK_PREFIX}-main \
    --region=${REGION} \
    --range=192.168.100.0/24
```


## Create firewall rules allowing internal traffic to enable workers to communicate:

```bash
gcloud compute --project=${PROJECT_ID} firewall-rules create ${GVNIC_NETWORK_PREFIX}-allow-internal \
    --network=${GVNIC_NETWORK_PREFIX}-main \
    --allow=all \
    --source-ranges=172.16.0.0/12,192.168.0.0/16,10.0.0.0/8 \
    --description="Allow all internal traffic within the network."
```
