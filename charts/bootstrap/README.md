# Bootstrap Helm Chart

This Helm chart implements the ArgoCD "App of Apps" pattern for deploying Gateway API, cert-manager, and NGINX Gateway Fabric in the correct order.

## Overview

The bootstrap chart creates ArgoCD Application resources that manage three critical infrastructure components:

- **Gateway API** (sync-wave: 1) - Kubernetes Gateway API CRDs
- **cert-manager** (sync-wave: 2) - Certificate management with Gateway API support
- **NGINX Gateway Fabric** (sync-wave: 2) - NGINX Gateway implementation

This provides:

- Proper deployment order using sync waves
- Centralized management of infrastructure components
- GitOps workflow for infrastructure as code
- Integration with Gateway API ecosystem

## Prerequisites

- ArgoCD installed in your cluster
- Helm 3.x
- Kubernetes cluster with ArgoCD CRDs available
- Access to OCI registries (Quay.io, GitHub Container Registry)

## Installation

1. **Clone or download this chart**

2. **Customize the values.yaml file** if needed:
   ```yaml
   spec:
     destination:
       server: https://your-cluster.example.com
   
   certManager:
     targetRevision: v1.19.0
   
   nginxGatewayFabric:
     targetRevision: 2.1.4
   ```

3. **Install the chart**:
   ```bash
   helm install bootstrap ./charts/bootstrap -n argocd
   ```

## Configuration

### Global Configuration

The `global` section provides default settings:

```yaml
global:
  namespace: argocd          # Namespace for ArgoCD Applications
  project: default           # ArgoCD project name
  destination:
    server: https://kubernetes.default.svc  # Target cluster for all applications
  syncPolicy:
    automated:
      prune: true           # Automatically prune resources
      selfHeal: true        # Automatically heal drift
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ServerSideApply=true
```

**Sync Options Explained:**
- `CreateNamespace=true`: Automatically create namespaces if they don't exist
- `PrunePropagationPolicy=foreground`: Prune resources in the correct order
- `PruneLast=true`: Prune resources after creating new ones
- `ServerSideApply=true`: Use Kubernetes server-side apply for better conflict resolution

### Application Configuration

Each application can be enabled/disabled and configured independently:

#### Gateway API

Gateway API is always enabled with configurable version and path:

```yaml
gatewayApi:
  targetRevision: v1.4.0           # Gateway API version
  path: config/crd                 # Path to CRD manifests
```

Fixed values:
- Application name: `gateway-api`
- Sync wave: "1" (deploys first)
- Repository: `github.com/kubernetes-sigs/gateway-api`
- Destination server: Uses global setting

Note: Ensure the Gateway API version is compatible with your cert-manager and NGINX Gateway Fabric versions.

#### cert-manager

cert-manager can be enabled/disabled and configured with version and Helm values:

```yaml
certManager:
  enabled: true                    # Enable/disable cert-manager
  targetRevision: v1.19.0         # cert-manager chart version
  values:                         # Helm values for cert-manager
    crds:
      enabled: true
    config:
      enableGatewayAPI: true
```

Fixed values:
- Application name: `cert-manager`
- Sync wave: "2" (deploys after Gateway API)
- Repository: `oci://quay.io/jetstack/charts/cert-manager`
- Path: `.`
- Destination namespace: `cert-manager`
- Destination server: Uses global setting

#### NGINX Gateway Fabric

NGINX Gateway Fabric can be enabled/disabled and configured with namespace, version, and Helm values:

```yaml
nginxGatewayFabric:
  enabled: true                    # Enable/disable NGINX Gateway Fabric
  namespace: nginx-gateway-fabric  # Target namespace for deployment
  targetRevision: 2.1.4           # Chart version
  values:                         # Helm values for NGINX Gateway Fabric
    crds:
      enabled: true
    config:
      enableGatewayAPI: true
```

Fixed values:
- Application name: `nginx-gateway-fabric`
- Sync wave: "2" (deploys after Gateway API)
- Repository: `oci://ghcr.io/nginx/charts/nginx-gateway-fabric`
- Path: `.`
- Destination server: Uses global setting

### Sync Waves

The applications are deployed in the correct order using ArgoCD sync waves:

1. **Wave 1**: Gateway API CRDs (prerequisite for other components)
2. **Wave 2**: cert-manager and NGINX Gateway Fabric (can deploy in parallel)

## Usage Examples

### 1. Basic Installation

Deploy all components with default settings:

```bash
helm install bootstrap ./charts/bootstrap -n argocd
```

### 2. Disable Specific Components

Deploy only Gateway API and cert-manager (disable NGINX Gateway Fabric):

```yaml
# values-minimal.yaml
nginxGatewayFabric:
  enabled: false
```

Deploy with:
```bash
helm install bootstrap ./charts/bootstrap -f values-minimal.yaml -n argocd
```

Note: Gateway API is always deployed as it's required for the other components.

### 3. Custom Versions

Deploy specific versions of components:

```yaml
# values-custom.yaml
gatewayApi:
  targetRevision: v1.3.0          # Use older Gateway API version
  path: config/crd

certManager:
  targetRevision: v1.18.0

nginxGatewayFabric:
  targetRevision: 2.0.0
```

### 4. Multi-cluster Deployment

Deploy to different clusters by setting the global destination server:

```yaml
# values-cluster1.yaml
global:
  destination:
    server: https://cluster1.example.com

# values-cluster2.yaml
global:
  destination:
    server: https://cluster2.example.com
```

## Post-Installation

### Verify Installation

1. **Check ArgoCD Applications**:
   ```bash
   kubectl get applications -n argocd
   ```

2. **Verify Gateway API CRDs**:
   ```bash
   kubectl get crd | grep gateway.networking.k8s.io
   ```

3. **Check cert-manager**:
   ```bash
   kubectl get pods -n cert-manager
   kubectl get crd | grep cert-manager
   ```

4. **Check NGINX Gateway Fabric**:
   ```bash
   kubectl get pods -n nginx-gateway-fabric
   ```

### Create Gateway Resources

After installation, you can create Gateway resources:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
```

## Troubleshooting

### Check Application Status

```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
```

### View Application Logs

```bash
argocd app get <app-name>
argocd app logs <app-name>
```

### Sync Applications Manually

```bash
argocd app sync gateway-api
argocd app sync cert-manager
argocd app sync nginx-gateway-fabric
```

### Check Component Logs

```bash
# cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# NGINX Gateway Fabric logs
kubectl logs -n nginx-gateway-fabric deployment/nginx-gateway-fabric
```

## Upgrading

To upgrade components to newer versions:

1. **Update targetRevision** in values.yaml:
   ```yaml
   certManager:
     targetRevision: v1.20.0
   
   nginxGatewayFabric:
     targetRevision: 2.2.0
   ```

2. **Upgrade the Helm release**:
   ```bash
   helm upgrade bootstrap ./charts/bootstrap -n argocd
   ```

3. **ArgoCD will automatically sync** the new versions

## Architecture

The chart deploys the following architecture:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Gateway API   │    │   cert-manager  │    │ NGINX Gateway   │
│     CRDs        │    │                 │    │    Fabric       │
│  (sync-wave: 1) │    │  (sync-wave: 2) │    │  (sync-wave: 2) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   ArgoCD        │
                    │   (GitOps)      │
                    └─────────────────┘
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with your ArgoCD setup
5. Submit a pull request

## License

This chart is licensed under the same license as the parent project.
