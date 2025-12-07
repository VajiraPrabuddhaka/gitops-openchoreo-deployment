# OpenChoreo Platform GitOps Repository

GitOps repository for deploying OpenChoreo platform to k3d using Flux CD.

## Components

- **Cert-Manager**: Certificate management for Kubernetes (dependency)
- **Control Plane**: Core OpenChoreo control plane with Backstage UI
- **Data Planes**: Runtime environments for applications (non-prod and prod)

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

## Quick Start (GitOps)

**Step 1: Install Flux** (if not already installed)

```bash
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml

# Wait for Flux to be ready
kubectl wait --for=condition=ready --timeout=5m pod -l app=source-controller -n flux-system
kubectl wait --for=condition=ready --timeout=5m pod -l app=kustomize-controller -n flux-system
kubectl wait --for=condition=ready --timeout=5m pod -l app=helm-controller -n flux-system
```

**Step 2: Apply GitOps bootstrap resources**

```bash
kubectl apply -f https://raw.githubusercontent.com/VajiraPrabuddhaka/gitops-openchoreo-deployment/main/bootstrap/gitrepository.yaml
kubectl apply -f https://raw.githubusercontent.com/VajiraPrabuddhaka/gitops-openchoreo-deployment/main/bootstrap/kustomization-k3d-openchoreo.yaml
```

That's it! Flux will now automatically sync and deploy all OpenChoreo components from this repository.

### Alternative: Manual Installation

If you prefer to apply manifests directly without GitOps sync:

```bash
# Clone the repository
git clone https://github.com/openchoreo/gitops-openchoreo-deployment.git
cd gitops-openchoreo-deployment

# Apply the manifests
kubectl apply -k clusters/k3d-openchoreo/
```

**Note**: With manual installation, you need to run `kubectl apply -k clusters/k3d-openchoreo/` again after any changes.

## Verify Deployment

```bash
# Check HelmReleases
kubectl get helmreleases -A

# Check pods
kubectl get pods -n cert-manager
kubectl get pods -n openchoreo-control-plane
kubectl get pods -n non-prod-dataplane
kubectl get pods -n prod-dataplane

# Check ConfigMaps were generated from values files
kubectl get configmap cert-manager-values -n cert-manager
kubectl get configmap openchoreo-control-plane-values -n openchoreo-control-plane
kubectl get configmap non-prod-dataplane-values -n non-prod-dataplane
kubectl get configmap prod-dataplane-values -n prod-dataplane
```

## Access

- Backstage UI: http://openchoreo.localhost:8080
- API: http://api.openchoreo.localhost:8080
- Identity Service: http://thunder.openchoreo.localhost:8080

## Post-Installation

After all components are deployed, you may need to register the data planes:

```bash
# Register data plane
curl -s https://raw.githubusercontent.com/openchoreo/openchoreo/main/install/add-data-plane.sh | bash -s -- --control-plane-context k3d-openchoreo --enable-agent --agent-ca-namespace openchoreo-control-plane
```

## Customization

The Helm values are in separate YAML files for easy customization:

- **values-cp.yaml**: Control plane configuration (ingress hosts, cluster gateway, etc.)
- **values-non-prod-dp.yaml**: Non-prod data plane configuration (gateway ports, cluster agent, etc.)
- **values-prod-dp.yaml**: Prod data plane configuration (gateway ports, cluster agent, etc.)

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
├── bootstrap/                                   # GitOps bootstrap resources
│   ├── gitrepository.yaml                       # Flux GitRepository (points to this repo)
│   └── kustomization-k3d-openchoreo.yaml        # Flux Kustomization (deploys cluster config)
└── clusters/
    └── k3d-openchoreo/
        ├── kustomization.yaml                   # Kustomize config (generates ConfigMaps)
        ├── openchoreo-helm-repo.yaml            # OpenChoreo Helm repository
        ├── cert-manager-repo.yaml               # Cert-manager Helm repository
        ├── cert-manager.yaml                    # Cert-manager HelmRelease
        ├── openchoreo-control-plane.yaml        # Control plane HelmRelease
        ├── non-prod-dataplane.yaml              # Non-prod data plane HelmRelease
        ├── prod-dataplane.yaml                  # Prod data plane HelmRelease
        ├── values-cert-manager.yaml             # Cert-manager values
        ├── values-cp.yaml                       # Control plane values
        ├── values-non-prod-dp.yaml              # Non-prod data plane values
        └── values-prod-dp.yaml                  # Prod data plane values
```

**How it works**:
1. The bootstrap resources create a Flux GitRepository and Kustomization that point to this repo
2. Flux automatically syncs and applies the `clusters/k3d-openchoreo/` path
3. The `kustomization.yaml` generates ConfigMaps from the `values-*.yaml` files
4. Each HelmRelease references its ConfigMap via `valuesFrom`

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