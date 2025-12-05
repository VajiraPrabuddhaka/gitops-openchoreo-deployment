# OpenChoreo Platform GitOps Repository

GitOps repository for deploying OpenChoreo platform to k3d using Flux CD.

## Components

- **Control Plane**: Core OpenChoreo control plane with Backstage UI
- **Data Plane**: Runtime environment for applications
- **Build Plane**: CI/CD with Argo Workflows and container registry

## Prerequisites

1. k3d cluster created with port mappings:

    > [!IMPORTANT]
    > If you're using Colima, set the `K3D_FIX_DNS=0` environment variable when creating clusters.
    > See [k3d-io/k3d#1449](https://github.com/k3d-io/k3d/issues/1449) for more details.
    > Example: `export K3D_FIX_DNS=0`
   
    ```bash
    curl -s https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/k3d/single-cluster/config.yaml | k3d cluster create --config=-
    ```

2. kubectl

## Installation

```bash
# Install Flux components
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml

# Wait for Flux to be ready
kubectl wait --for=condition=ready --timeout=5m pod -l app=source-controller -n flux-system
kubectl wait --for=condition=ready --timeout=5m pod -l app=kustomize-controller -n flux-system
kubectl wait --for=condition=ready --timeout=5m pod -l app=helm-controller -n flux-system

# Apply the manifests (from your local git clone)
kubectl apply -k clusters/k3d-openchoreo/
```

**Note**: To apply changes after editing values files, run `kubectl apply -k clusters/k3d-openchoreo/` again.

## Verify Deployment

```bash
# Check HelmReleases
kubectl get helmreleases -A

# Check pods
kubectl get pods -n openchoreo-control-plane
kubectl get pods -n openchoreo-data-plane
kubectl get pods -n openchoreo-build-plane

# Check ConfigMaps were generated from values files
kubectl get configmap openchoreo-control-plane-values -n openchoreo-control-plane
kubectl get configmap openchoreo-data-plane-values -n openchoreo-data-plane
kubectl get configmap openchoreo-build-plane-values -n openchoreo-build-plane
```

## Access

- Backstage UI: http://openchoreo.localhost:8080
- API: http://api.openchoreo.localhost:8080
- Identity Service: http://thunder.openchoreo.localhost:8080
- Argo Workflows: http://localhost:10081

## Post-Installation

After all components are deployed, you may need to register the data plane and build plane:

```bash
# Register data plane
curl -s https://raw.githubusercontent.com/openchoreo/openchoreo/release-v0.6/install/add-data-plane.sh | bash -s -- --control-plane-context k3d-openchoreo --enable-agent --agent-ca-namespace openchoreo-control-plane

# Register build plane
curl -s https://raw.githubusercontent.com/openchoreo/openchoreo/release-v0.6/install/add-build-plane.sh | bash -s -- --control-plane-context k3d-openchoreo
```

## Customization

The Helm values are in separate YAML files for easy customization:

- **values-cp.yaml**: Control plane configuration (ingress hosts, cluster gateway, etc.)
- **values-dp.yaml**: Data plane configuration (gateway ports, cluster agent, etc.)
- **values-bp.yaml**: Build plane configuration (Argo Workflows, registry, etc.)

**To customize**: Simply edit the values file you need and reapply:
1. Edit the values file (e.g., `values-cp.yaml`)
2. Run `kubectl apply -k clusters/k3d-openchoreo/`
3. Kustomize will regenerate ConfigMaps from the updated values
4. Flux will trigger HelmRelease reconciliation with the new values

No need to edit multiple files - just update the values YAML and apply!

## Repository Structure

```
gitops-openchoreo-deployment/
├── README.md
└── clusters/
    └── k3d-openchoreo/
        ├── kustomization.yaml                 # Kustomize config (generates ConfigMaps)
        ├── openchoreo-control-plane.yaml      # Control plane HelmRelease
        ├── openchoreo-data-plane.yaml         # Data plane HelmRelease
        ├── openchoreo-build-plane.yaml        # Build plane HelmRelease
        ├── values-cp.yaml                     # Control plane values
        ├── values-dp.yaml                     # Data plane values
        └── values-bp.yaml                     # Build plane values
```

**How it works**: The `kustomization.yaml` automatically generates ConfigMaps from the `values-*.yaml` files. Each HelmRelease references its ConfigMap via `valuesFrom`.

## Troubleshooting

```bash
# Check HelmRelease status
kubectl describe helmrelease openchoreo-control-plane -n openchoreo-control-plane

# Check Helm controller logs
kubectl logs -n flux-system -l app=helm-controller --tail=100

# Check if ConfigMap exists and has correct data
kubectl get configmap openchoreo-control-plane-values -n openchoreo-control-plane -o yaml

# Manually reapply after making changes
kubectl apply -k clusters/k3d-openchoreo/

# Delete and recreate a HelmRelease to force reinstall
kubectl delete helmrelease openchoreo-control-plane -n openchoreo-control-plane
kubectl apply -k clusters/k3d-openchoreo/
```