# hashdeep-image

A minimal Docker image with [hashdeep](https://github.com/jessek/hashdeep) preinstalled, built on `debian:bookworm-slim`.

Used to run recursive hash comparisons against data in a Kubernetes cluster (e.g. against PVCs) as part of a file-integrity check when migrating data.

## Image

Automatically published to GitHub Container Registry via GitHub Actions on push to `main`:

```
ghcr.io/moorgrove/hashdeep-image:latest
```

Multi-arch: `linux/amd64` and `linux/arm64`.

## Usage

The image uses `hashdeep` as its entrypoint, so any standard hashdeep flags can be passed as `args`.

### Example: Kubernetes Job against a PVC
hashdeep-job.tmpl.yaml
```apiVersion: batch/v1
kind: Job
metadata:
  name: hashdeep-${PVC_NAME}
  namespace: ${NAMESPACE}
spec:
  backoffLimit: 0
  activeDeadlineSeconds: 86400
  template:
    spec:
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: hashdeep
        image: ghcr.io/moorgrove/hashdeep-image:latest
        args: ["-r", "-j", "0", "-o", "f", "/data"]
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: data
          mountPath: /data
          readOnly: true
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: ${PVC_NAME}
          readOnly: true
```

Retrieve the result to your desktop:
hashdeep-script.sh
```#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${NAMESPACE:-default}"
PVC_LIST=("$@")   # t.ex. ./run_hashdeep.sh pvc-app1 pvc-app2 pvc-app3

for PVC_NAME in "${PVC_LIST[@]}"; do
  export PVC_NAME NAMESPACE
  echo "== Kör hashdeep mot PVC: $PVC_NAME =="

  if kubectl get job "hashdeep-${PVC_NAME}" -n "$NAMESPACE" >/dev/null 2>&1; then
    echo "!! Job hashdeep-${PVC_NAME} finns redan i namespace $NAMESPACE. Hoppar över."
    echo "   Ta bort den manuellt om du vill köra om: kubectl delete job hashdeep-${PVC_NAME} -n $NAMESPACE"
    continue
  fi

  envsubst < hashdeep-job.tmpl.yaml | kubectl apply -f -
  kubectl wait --for=condition=complete "job/hashdeep-${PVC_NAME}" -n "$NAMESPACE" --timeout=86400s

  POD=$(kubectl get pods -n "$NAMESPACE" -l job-name="hashdeep-${PVC_NAME}" -o jsonpath='{.items[0].metadata.name}')
  kubectl logs "$POD" -n "$NAMESPACE" > "hashdeep_${PVC_NAME}.txt"

  kubectl delete job "hashdeep-${PVC_NAME}" -n "$NAMESPACE"
  echo "-> Klart: hashdeep_${PVC_NAME}.txt"
done
```

### Run script
```
NAMESPACE=mitt-namespace ./hashdeep-script.sh pvc-app1 pvc-app2 pvc-app3
```

## Building locally

```bash
docker build -t hashdeep-image .
docker run --rm -v /path/to/data:/data:ro hashdeep-image -r -j 0 -o f /data
```

## Background

Built to verify that files migrated from source to various PVCs in the cluster have transferred correctly.

## Pinning the hashdeep version

The Dockerfile currently uses whatever hashdeep version ships with `debian:bookworm-slim` at build time. If you want to guarantee an identical output format across runs over time, pin the version in the Dockerfile, e.g.:

```dockerfile
RUN apt-get install -y --no-install-recommends hashdeep=4.4-2
```
