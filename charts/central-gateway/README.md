# Central Gateway

A Helm chart for deploying a central gateway with PKI (Public Key Infrastructure) and TLS termination using cert-manager. This chart creates a self-signed root certificate authority, an intermediate CA, and a Gateway API resource for centralized traffic management with automatic TLS certificate provisioning.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- cert-manager v1.19.0+ installed in your cluster
- Gateway API CRDs installed in your cluster

## Installation

### Install cert-manager

First, ensure cert-manager is installed in your cluster using the OCI chart:

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.0
```

### Install central-gateway

```bash
# Install in the cert-manager namespace
helm install central-gateway . \
  --namespace central-gateway \
  --create-namespace
```

## What This Chart Creates

This chart deploys a complete PKI infrastructure and central gateway:

1. **Root Issuer** (`central-gateway-root`): A self-signed root certificate authority
2. **CA Certificate** (`central-gateway-ca`): An intermediate CA certificate signed by the root
3. **CA Issuer** (`central-gateway-ca`): A ClusterIssuer that can issue certificates using the CA
4. **Gateway** (`central-gateway`): A Gateway API resource for centralized traffic management with TLS termination

## Configuration

The following table lists the configurable parameters and their default values:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ca.name` | Name of the CA certificate | `k8s-local-ca` |
| `ca.commonName` | Common name for the CA certificate | `ca.k8s.local` |
| `ca.secretName` | Name of the secret storing the CA certificate | `k8s-local-ca` |
| `ca.privateKey.algorithm` | Private key algorithm | `ECDSA` |
| `ca.privateKey.size` | Private key size | `256` |
| `rootIssuer.name` | Name of the root ClusterIssuer | `k8s-local-root` |
| `caIssuer.name` | Name of the CA ClusterIssuer | `k8s-local-ca` |
| `global.labels` | Additional labels to apply to all resources | `{}` |
| `global.annotations` | Additional annotations to apply to all resources | `{}` |

## Usage

### Issuing Certificates

Once installed, you can use the `central-gateway-ca` ClusterIssuer to issue certificates for your applications, and the Gateway will handle TLS termination:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-cert
  namespace: my-namespace
spec:
  secretName: my-app-tls
  issuerRef:
    name: central-gateway-ca
    kind: ClusterIssuer
  dnsNames:
  - my-app.example.com
  - my-app.local
```

### Using Gateway API

The Gateway provides the following listeners for traffic management:

- **http-wildcard**: listens to HTTP traffic for all subdomains (`*.{{ .Values.gateway.hostname }}`)
- **https-wildcard**: listens to HTTPS traffic for all subdomains (`*.{{ .Values.gateway.hostname }}`)

You can use HTTPRoute resources to route traffic to your applications:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
  - name: central-gateway
    namespace: cert-manager
    sectionName: https-wildcard
  hostnames:
  - app.k8s.local
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 8080
```

### Custom Configuration

You can customize the installation by providing your own values:

```bash
helm install central-gateway . \
  --namespace central-gateway \
  --set ca.commonName=my-ca.local \
  --set ca.privateKey.algorithm=RSA \
  --set ca.privateKey.size=2048 \
  --set gateway.hostname=myapp.local
```

### Adding Labels and Annotations

You can add custom labels and annotations to all resources:

```bash
helm install central-gateway . \
  --namespace cert-manager \
  --set global.labels.environment=production
```

## Verification

After installation, verify that the resources were created successfully:

```bash
# Check ClusterIssuers
kubectl get clusterissuers

# Check Certificate
kubectl get certificate -n cert-manager

# Check Certificate status
kubectl describe certificate central-gateway-ca -n cert-manager

# Check the CA secret
kubectl get secret central-gateway-ca -n cert-manager

# Check Gateway
kubectl get gateway central-gateway -n cert-manager
```

## Uninstalling

To uninstall the chart:

```bash
helm uninstall central-gateway -n central-gateway
```

**Note**: This will remove the CA certificate and ClusterIssuer, but will not affect certificates already issued by the CA.

## Contributing

Contributions are welcome! Please see the main project repository for contribution guidelines.

## License

This chart is licensed under the MIT License. See the LICENSE file for details.

## Support

For issues and questions, please open an issue in the main project repository.
