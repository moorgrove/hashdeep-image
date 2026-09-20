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

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hashdeep-<pvc-name>
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: hashdeep
          image: ghcr.io/moorgrove/hashdeep-image:latest
          args: ["-r", "-j", "0", "-o", "f", "/data"]
          volumeMounts:
            - name: data
              mountPath: /data
              readOnly: true
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <pvc-name>
            readOnly: true
```

Retrieve the result to your desktop:

```bash
kubectl wait --for=condition=complete job/hashdeep-<pvc-name> --timeout=1800s
POD=$(kubectl get pods -l job-name=hashdeep-<pvc-name> -o jsonpath='{.items[0].metadata.name}')
kubectl logs "$POD" > hashdeep_<pvc-name>.txt
kubectl delete job hashdeep-<pvc-name>
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
