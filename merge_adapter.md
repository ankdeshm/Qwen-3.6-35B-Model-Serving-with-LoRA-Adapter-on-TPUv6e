# Merge Lora Adapter with Base Model

```bash
nano qwen-merge-job.yaml
```


```bash

apiVersion: batch/v1
kind: Job
metadata:
  name: qwen-merge-job
spec:
  backoffLimit: 0
  template:
    metadata:
      annotations:
        gke-gcsfuse/volumes: "true"
        autoscaling.gke.io/provisioning-request: "tpu-v6e-16-pr"
    spec:
      restartPolicy: Never
      serviceAccountName: default
      containers:
      - name: merge-worker
        image: vllm/vllm-tpu:nightly
        command: ["sh", "-c"]
        args:
        - |
          echo "Installing PEFT..."
          pip install peft==0.11.1
          
          echo "Writing merge script..."
          cat << 'EOF' > merge.py
          from transformers import AutoModelForCausalLM, AutoTokenizer
          from peft import PeftModel
          import torch
          import os
          
          base_model_path = "/models/qwen36-35b-weights"
          lora_path = "/models/qwen-lora-adapter"
          output_path = "/models/qwen36-35b-merged"
          
          print("Loading base model...")
          base_model = AutoModelForCausalLM.from_pretrained(
              base_model_path, 
              torch_dtype=torch.bfloat16, 
              device_map="cpu"
          )
          
          print("Loading tokenizer...")
          tokenizer = AutoTokenizer.from_pretrained(base_model_path)
          
          print("Loading LoRA adapter and merging...")
          model = PeftModel.from_pretrained(base_model, lora_path)
          merged_model = model.merge_and_unload()
          
          print(f"Saving merged model to {output_path}...")
          os.makedirs(output_path, exist_ok=True)
          merged_model.save_pretrained(output_path)
          tokenizer.save_pretrained(output_path)
          print("Merge complete!")
          EOF
          
          echo "Executing merge script..."
          python3 merge.py
        volumeMounts:
        - name: model-weights
          mountPath: /models
        resources:
          requests:
            memory: "250Gi"
      volumes:
      - name: model-weights
        csi:
          driver: gcsfuse.csi.storage.gke.io
          volumeAttributes:
            bucketName: ank-qwen-serving-314837540096
            mountOptions: "implicit-dirs"
      nodeSelector:
        cloud.google.com/gke-tpu-accelerator: tpu-v6e-slice
      tolerations:
      - key: "google.com/tpu"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "cloud.google.com/gke-queued"
        operator: "Exists"
        effect: "NoSchedule"

```

```bash
kubectl apply -f qwen-merge-job.yaml
```
