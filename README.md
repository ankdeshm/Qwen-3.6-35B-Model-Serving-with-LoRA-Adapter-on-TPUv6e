# Qwen-3.6-35B Model Serving with LoRA Adapter on TPU v6e

This repository provides an end-to-end guide, infrastructure configuration, and deployment manifests for serving the **Qwen 3.6 35B** model with a custom **LoRA adapter** optimized on Google Kubernetes Engine (GKE) using **TPU v6e** accelerators. 

We benchmark and support two primary TPU hardware variants:
*   **Single-Host Variant (`v6e-8`):** Leverages a single TPU host containing 8 interconnected chips.
*   **Multi-Host Variant (`v6e-16`):** Leverages a multi-host environment across a 4x4 topology spanning 16 TPU chips.

---

## Deployment Workflow Overview

Deploying this highly optimized, long-context workload requires a structured sequence across three distinct phases: **Infrastructure Provisioning**, **Model Preparation & Optimization**, and **Engine Deployment**.


---

## 📋 Step-by-Step Setup Guide

### Phase 1: Infrastructure & Networking Setup
Before deploying the serving components, the fundamental GKE and VPC environment must be built out.

1.  **Configure Custom Networking**
    High-throughput TPU multi-host communication requires dedicated network setups to prevent inter-node bottlenecks. Set up the network stack by following:
    `create_custom_networking.md`
2.  **Spin Up Infrastructure & GKE Cluster**
    Deploy the core GKE cluster pinned to your custom VPC/subnetwork, retrieve credentials, inject your Hugging Face secret, and install the LeaderWorkerSet (LWS) operator via Helm. This file also provisions GCS Rapid Cache in the TPU zone and configures secure Google Cloud Workload Identity bindings for storage access control. Follow the setup via:
    `setup_infra.md`
3.  **Provision TPU Node Pools**
    Depending on whether you are targeting the single-host (`v6e-8`) or multi-host (`v6e-16`) deployment architecture, create the specialized TPU node pools by executing the configurations within:
    `provision_tpu.md`

### Phase 2: Model Acquisition & LoRA Merging
To guarantee low latency and avoid the execution overhead of fetching dynamic adapters during runtime, we pre-merge the LoRA weights directly into the base model's weight matrices.

1.  **Download Base Model Weights**
    Pull the native Qwen 3.6 35B model files down from Hugging Face directly into a designated Google Cloud Storage (GCS) bucket. *Note: A valid Hugging Face (`HF_TOKEN`) authorized for the repository access is required.* Follow the instructions in:
    `download_base_model.md`
2.  **Execute the LoRA Weight Merge**
    Run the offline merging script to blend the base model weights stored in your GCS bucket with your target LoRA adapter. The unified output is stored back into the bucket, mounted inside GKE pods seamlessly using **GCS FUSE**. Follow the steps in:
    `merge_adapter.md`

### Phase 3: GKE Service Deployment
Once the merged weights are ready in your bucket, apply the Kubernetes serving manifest matching your target tensor-parallelism (TP) and data-parallelism (DP) layout:

*   **For TPU v6e-8 (Tensor Parallelism = 8, Data Parallelism = 1):**

```bash
kubectl apply -f qwen-singlehost-tp8-dp1.yaml
```

*   **For TPU v6e-8 (Tensor Parallelism = 4, Data Parallelism = 2 via Process Isolation):**

```bash
kubectl apply -f qwen-singlehost-tp4-dp2.yaml
```

*   **For TPU v6e-16 Multi-Host (Tensor Parallelism = 16, Pipeline Parallelism = 1):**
  
```bash
kubectl apply -f qwen-multihost-tp16-pp1.yaml
```

---

## 📊 Benchmarking & Performance Results

We track automated concurrency sweeps (ranging from 8 to 64 concurrent streams) evaluating Throughput, Time-to-First-Token (TTFT), and Time-Per-Output-Token (TPOT). 

All raw JSON outputs generated from our sequential testing sweeps can be explored inside the [`/results`](./results) directory.
