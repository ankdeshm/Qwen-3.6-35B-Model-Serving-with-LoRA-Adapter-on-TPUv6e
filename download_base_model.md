# Downalod base model from Hugging Face


### Create qwen36-35b-model-downloader.yaml
```bash
nano qwen36-35b-model-downloader.yaml
```

### Paste this content

```bash

apiVersion: batch/v1
kind: Job
metadata:
  name: model-downloader
spec:
  ttlSecondsAfterFinished: 60
  template:
    metadata:
      annotations:
        gke-gcsfuse/volumes: "true"
        gke-gcsfuse/memory-limit: "0"
    spec:
      serviceAccountName: default
      restartPolicy: OnFailure
      containers:
      - name: downloader
        image: python:3.10-slim
        command: ["/bin/sh", "-c"]
        args:
        - |
          pip install -U "huggingface_hub[hf_transfer]" filelock
          export HF_HUB_ENABLE_HF_TRANSFER=1

          python -c '
          import filelock

          class DummyLock:
              def __init__(self, *args, **kwargs): pass
              def __enter__(self): return self
              def __exit__(self, *args): pass
              def acquire(self, *args, **kwargs): pass
              def release(self, *args, **kwargs): pass

          filelock.FileLock = DummyLock

          from huggingface_hub import snapshot_download
          snapshot_download(
              repo_id="Qwen/Qwen3.6-35B-A3B",
              local_dir="/models/qwen36-35b-weights",
              local_dir_use_symlinks=False
          )
          '
        env:
        - name: HF_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: hf_api_token
        volumeMounts:
        - name: model-weights
          mountPath: /models
      volumes:
      - name: model-weights
        csi:
          driver: gcsfuse.csi.storage.gke.io
```

```bash
kubectl apply -f qwen36-35b-model-downloader.yaml
```

