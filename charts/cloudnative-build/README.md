# CloudNative Build

A Helm chart for deploying CloudNative Build tools using ArgoCD Applications.
This chart creates ArgoCD Application resources that manage the deployment of
Tekton, Shipwright, and Harbor for QuizApp developers.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- ArgoCD installed and running in your cluster
- ArgoCD CLI installed (optional, for manual operations)

## Installation

### Install ArgoCD

First, ensure ArgoCD is installed in your cluster:

```bash
# Install ArgoCD using Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argo-cd \
  --create-namespace \
  --set global.domain=argocd.example.com
```

### Install cloudnative-build

```bash
# Install the cloudnative-build chart
helm install cloudnative-build . \
  --namespace argo-cd \
  --create-namespace
```

## What This Chart Creates

This chart deploys ArgoCD Application resources that manage CloudNative Build tools:

1. `tekton`: Deploys Tekton Pipelines to the `tekton-pipelines` namespace
2. `shipwright`: Deploys Shipwright Build to the `shipwright-build` namespace
3. `harbor`: Deploys the Harbor container registry to the `harbor` namespace

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `argocd.namespace` | Namespace where ArgoCD Applications will be created | `argo-cd` |
| `argocd.project` | ArgoCD project name | `default` |
| `argocd.destination.server` | Destination server for all applications | `https://kubernetes.default.svc` |
| `argocd.syncPolicy.automated.prune` | Enable automatic pruning of resources | `true` |
| `argocd.syncPolicy.automated.selfHeal` | Enable automatic self-healing | `true` |
| `argocd.syncPolicy.syncOptions` | Additional sync options for applications | `["CreateNamespace=true", "PrunePropagationPolicy=foreground", "PruneLast=true", "ServerSideApply=true"]` |
| `argocd.finalizers` | Finalizers for the Application resource | `["resources-finalizer.argocd.argoproj.io"]` |
| `argocd.labels` | Additional labels to apply to applications | `{}` |
| `argocd.annotations` | Additional annotations to apply to applications | `{}` |
| `harbor.enabled` | Enables Harbor deployment | `true` |
| `harbor.namespace` | Namespace where Harbor is deployed | `harbor` |
| `harbor.targetRevision` | Version of the Harbor chart to deploy | `v1.18.0` |
| `harbor.tls.enabled` | Enable backend TLS for Harbor | `true` |
| `harbor.tls.secretName` | Reference to TLS secret for Harbor backend service | `harbor-tls` |
| `harbor.gateway.name` | Name of the gateway for Harbor ingress | `central-gateway` |
| `harbor.gateway.namespace` | Namespace of the gateway for Harbor ingress | `central-gateway` |
| `harbor.gateway.sectionName` | Section of the gateway for Harbor ingress | `https-wildcard` |
| `harbor.hostname` | Hostname for Harbor | `registry.k8s.local` |
| `tekton.enabled` | Enable Tekton Pipelines deployment | `true` |
| `tekton.namespace` | Namespace where Tekton will be deployed | `tekton-pipelines` |
| `tekton.targetRevision` | Git revision/branch for Tekton deployment | `tekton` |
| `shipwright.enabled` | Enable Shipwright Build deployment | `true` |
| `shipwright.namespace` | Namespace where Shipwright will be deployed | `shipwright-build` |
| `shipwright.targetRevision` | Git revision/branch for Shipwright deployment | `shipwright` |

## Usage

### ArgoCD Application Management

Once installed, the chart creates ArgoCD Application resources that automatically manage the deployment of CloudNative Build tools. You can monitor and manage these applications through the ArgoCD UI or CLI.

### Accessing ArgoCD UI

```bash
# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argo-cd 8080:443

# Get the admin password
kubectl -n argo-cd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Custom Configuration

You can customize the installation by providing your own values:

```bash
helm install cloudnative-build . \
  --namespace argo-cd \
  --set argocd.namespace=argo-cd \
  --set argocd.project=my-project \
  --set tekton.enabled=true \
  --set tekton.namespace=my-tekton-namespace \
  --set tekton.targetRevision=main
```

### Adding Labels and Annotations

You can add custom labels and annotations to all ArgoCD Applications:

```bash
helm install cloudnative-build . \
  --namespace argo-cd \
  --set argocd.labels.environment=production \
  --set argocd.labels.team=platform \
  --set argocd.annotations."argocd\.argoproj\.io/sync-wave"="1"
```

## Verification

After installation, verify that the ArgoCD Applications were created successfully:

```bash
# Check ArgoCD Applications
kubectl get applications -n argo-cd

# Check specific Tekton application
kubectl get application tekton -n argo-cd

# Check application status
kubectl describe application tekton -n argo-cd

# Check if Tekton is deployed
kubectl get pods -n tekton-pipelines

# Check if Shipwright is deployed
kubectl get pods -n shipwright-build

# Check if Harbor is deployed
kubectl get pods -n harbor
```

## Troubleshooting

### Application Not Syncing

If the ArgoCD Application is not syncing, check the application status:

```bash
# Check application status
kubectl describe application tekton -n argo-cd

# Check ArgoCD logs
kubectl logs -n argo-cd deployment/argocd-application-controller
```

### Wrong Namespace

If you installed the chart in the wrong namespace, uninstall and reinstall:

```bash
helm uninstall cloudnative-build -n wrong-namespace
helm install cloudnative-build . -n argo-cd
```

### Application Not Found

Ensure ArgoCD is running and the Application was created:

```bash
kubectl get applications -n argo-cd
```

## Security Considerations

- ArgoCD Applications have cluster-wide permissions. Ensure proper RBAC is configured.
- The chart uses Git repositories for source management. Ensure repository access is properly configured.
- Consider using ArgoCD Projects to limit application permissions and resources.

## Uninstalling

To uninstall the chart:

```bash
helm uninstall cloudnative-build -n argo-cd
```

**Note**: This will remove the ArgoCD Application resources, but will not affect the actual deployments managed by ArgoCD. The deployed applications will continue running until manually deleted through ArgoCD.

## Contributing

Contributions are welcome! Please see the main project repository for contribution guidelines.

## License

This chart is licensed under the MIT License. See the LICENSE file for details.

## Support

For issues and questions, please open an issue in the main project repository.
