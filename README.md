# atarnet-navidrome

Helm chart and CI pipeline assets for running the [Navidrome](https://www.navidrome.org/) self‑hosted music server on Kubernetes. The repository contains everything needed to deploy the application, expose it through an ingress, provision persistent storage for metadata, and run a Jenkins pipeline that scans and deploys the chart.

## Repository Layout
- `charts/app/` – Helm chart for Navidrome (deployment, service, ingress, PVC, Helm test, and a pre-install patch job).
- `ci/kubernetes/trivy.yaml` – Kubernetes Job manifest used by the pipeline for Trivy image scans.
- `Jenkinsfile` – Declarative pipeline that installs tooling, validates cluster access, runs the Trivy scan, deploys the chart, and executes `helm test`.

## Prerequisites
- Kubernetes cluster with access to a storage class that supports `ReadWriteOnce`.
- Helm 3.x installed locally, or Jenkins access capable of running Helm commands.
- NGINX (or compatible) ingress controller for routing external traffic to Navidrome.
- Node(s) with access to the host path defined in `values.yaml` (default `/mnt/test/organized_music`) so Navidrome can read your music library.

## Configuration
Update `charts/app/values.yaml` or supply a custom values file when installing. Key settings:

| Value | Description | Default |
| --- | --- | --- |
| `namespace` | Kubernetes namespace to host the release. | `navidrome` |
| `name` | Release name used for resources. | `navidrome` |
| `image` | Navidrome container image tag. | `ghcr.io/navidrome/navidrome:latest` |
| `host` | Ingress host routed to the service. | `music.atarnet.org` |
| `storageClass` | Storage class bound to the Navidrome data PVC. | `microk8s-hostpath` |
| `musicHostPath` | Host path exposing your music library to the pod. | `/mnt/test/organized_music` |

## Deploy with Helm
```bash
# Optionally override settings
helm upgrade --install navidrome charts/app \
  --namespace navidrome \
  --create-namespace \
  --set image=ghcr.io/navidrome/navidrome:latest \
  -f my-values.yaml

# Validate release
helm test navidrome --namespace navidrome --logs
```

The deployment mounts your music library at `/music` and stores Navidrome metadata under `/data` backed by a PVC. The chart exposes the application through a ClusterIP service and an ingress bound to the configured host.

## Jenkins Pipeline
The included `Jenkinsfile` expects the following:
- A credential named `kubeconfigglobal` containing a kubeconfig file.
- Network access to download Helm, kubectl, and run Trivy scans inside the cluster.

Pipeline stages:
1. Checkout repository and place helper binaries under `$WORKSPACE/bin`.
2. Download kubectl and Helm (architecture aware).
3. Copy the kubeconfig from Jenkins credentials and verify cluster reachability.
4. Launch the `ci/kubernetes/trivy.yaml` job in-cluster to scan the Navidrome image.
5. Deploy the Helm chart (`helm upgrade --install`) and wait for the rollout.
6. Execute `helm test` to ensure the ingress/service respond.

## Notes
- The chart includes a pre-install hook job that patches existing resources with Helm ownership labels, easing transitions from manually created resources.
- The Helm test spins up a BusyBox pod that curls the service endpoint to verify reachability.

## Development
Common tasks while iterating:

```bash
# Render chart locally
helm template navidrome charts/app --namespace navidrome

# Lint chart
helm lint charts/app

# Dry-run upgrade
helm upgrade --install navidrome charts/app \
  --namespace navidrome --create-namespace --dry-run
```
