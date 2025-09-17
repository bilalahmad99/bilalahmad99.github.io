---
layout: default
---

## GitOps with Argo CD vs Flux

A practical comparison of the two leading GitOps tools for Kubernetes deployments, focusing on reconciliation models, drift detection, multi-cluster strategies, and secrets handling.

### What is GitOps?

GitOps is a methodology where Git serves as the single source of truth for declarative infrastructure and applications. Changes are made through Git commits, and an automated process ensures the actual state matches the desired state.

### Argo CD vs Flux: Quick Comparison

| Feature | Argo CD | Flux |
|---------|---------|------|
| **Architecture** | Pull-based, agent in cluster | Pull-based, operator in cluster |
| **UI** | Rich web interface | CLI-focused, basic UI |
| **Multi-cluster** | Native support | Native support |
| **Helm Support** | Excellent | Excellent |
| **Kustomize** | Native | Native |
| **Learning Curve** | Moderate | Steep |
| **Resource Usage** | Higher | Lower |

### Reconciliation Models

#### Argo CD
- **Pull-based**: Argo CD polls Git repositories for changes
- **Manual sync**: Changes require explicit sync (unless auto-sync enabled)
- **Sync waves**: Supports ordered deployment with sync waves
- **Sync policies**: Fine-grained control over sync behavior

```yaml
# Argo CD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

#### Flux
- **Pull-based**: Flux operator watches Git repositories
- **Automatic sync**: Changes are automatically applied
- **Reconciliation loops**: Continuous reconciliation every 5 minutes (configurable)
- **GitOps Toolkit**: Modular components for different Git sources

```yaml
# Flux GitRepository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: my-app
  path: "./k8s"
  prune: true
  validation: client
```

### Drift Detection

#### Argo CD
- **Live state monitoring**: Continuously compares desired vs actual state
- **Health checks**: Built-in health assessment for applications
- **Status indicators**: Clear visual indicators in UI
- **OutOfSync detection**: Identifies when cluster state differs from Git

#### Flux
- **Reconciliation loops**: Regular checks every 5 minutes
- **Status reporting**: Reports drift through Kubernetes events
- **Health assessment**: Basic health checking capabilities
- **Alerting**: Integrates with Prometheus for drift notifications

### Multi-cluster Strategies

#### Argo CD
- **ApplicationSets**: Deploy same app to multiple clusters
- **Cluster management**: Centralized cluster registration
- **RBAC**: Per-cluster access control
- **Secrets management**: Cluster-specific secrets

```yaml
# Argo CD ApplicationSet for multi-cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-multicluster
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  template:
    metadata:
      name: '{{name}}-my-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp
        targetRevision: HEAD
        path: k8s
      destination:
        server: '{{server}}'
        namespace: production
```

#### Flux
- **Flux Multi-tenancy**: Isolate workloads per tenant
- **Cluster API**: Native Kubernetes cluster management
- **GitOps Toolkit**: Modular approach for different cluster types
- **Fleet management**: Deploy to multiple clusters from single repo

### Secrets Handling

#### Argo CD
- **External secrets**: Integrates with external secret managers
- **Sealed Secrets**: Support for Bitnami Sealed Secrets
- **SOPS**: Encrypted secrets with Mozilla SOPS
- **Vault integration**: HashiCorp Vault support

```yaml
# Argo CD with Sealed Secrets
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-with-secrets
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: k8s
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
  sources:
  - repoURL: https://github.com/myorg/secrets
    path: sealed-secrets
    targetRevision: main
```

#### Flux
- **External Secrets Operator**: Native integration
- **SOPS**: Built-in support for encrypted secrets
- **Vault integration**: HashiCorp Vault support
- **Sealed Secrets**: Bitnami Sealed Secrets support

```yaml
# Flux with External Secrets
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: flux-system
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "flux"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: my-app-secrets
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: my-app
      property: db-password
```

### When to Choose Which?

#### Choose Argo CD when:
- You need a rich UI for GitOps operations
- Your team prefers manual control over deployments
- You have complex multi-cluster requirements
- You want extensive RBAC and project management
- You need detailed application health monitoring

#### Choose Flux when:
- You prefer GitOps-native approach
- You want lower resource consumption
- You need simple, automated deployments
- You're comfortable with CLI and YAML
- You want to leverage GitOps Toolkit components

### Best Practices

#### For Both Tools:
- **Repository structure**: Organize manifests logically
- **Environment separation**: Use different branches or directories
- **Security**: Implement proper RBAC and secret management
- **Monitoring**: Set up alerts for sync failures and drift
- **Testing**: Use staging environments for validation

#### Repository Structure Example:
```
my-gitops-repo/
├── clusters/
│   ├── production/
│   │   ├── apps/
│   │   └── infrastructure/
│   └── staging/
│       ├── apps/
│       └── infrastructure/
├── apps/
│   ├── my-app/
│   │   ├── base/
│   │   └── overlays/
│   └── another-app/
└── infrastructure/
    ├── monitoring/
    └── networking/
```

### Migration Considerations

- **From manual kubectl**: Both tools provide smooth migration paths
- **From Helm**: Both support Helm charts natively
- **Between tools**: Migration requires reconfiguring sync policies and RBAC
- **State management**: Ensure proper backup of existing cluster state

[back](../)
