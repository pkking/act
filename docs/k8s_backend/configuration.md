# Kubernetes Backend Configuration

## Overview

This document describes the configuration options for the Kubernetes backend, including integration with existing Act configuration systems and new Kubernetes-specific settings.

## Configuration Structure

### 1. Extended Config Structure

```go
type Config struct {
    // ... existing Act configuration fields

    // Kubernetes-specific configuration
    KubernetesConfig *KubernetesConfig `yaml:"kubernetes"`
    Backend          string            `yaml:"backend"`
}

type KubernetesConfig struct {
    Enabled           bool                              `yaml:"enabled"`
    Kubeconfig        string                           `yaml:"kubeconfig"`
    Context           string                           `yaml:"context"`
    Namespace         string                           `yaml:"namespace"`
    Platforms         map[string]KubernetesRunsOnConfig `yaml:"platforms"`
    DefaultResources  v1.ResourceRequirements          `yaml:"defaultResources"`
    PodTemplate       *v1.PodTemplateSpec              `yaml:"podTemplate"`
    StorageClass      string                           `yaml:"storageClass"`
    ImagePullSecrets  []string                         `yaml:"imagePullSecrets"`
    ServiceAccount    string                           `yaml:"serviceAccount"`
    CleanupPolicy     CleanupPolicy                    `yaml:"cleanupPolicy"`
    Networking        NetworkingConfig                 `yaml:"networking"`
    Security          SecurityConfig                   `yaml:"security"`
    Monitoring        MonitoringConfig                 `yaml:"monitoring"`
}

type CleanupPolicy struct {
    SuccessfulJobsHistoryLimit int    `yaml:"successfulJobsHistoryLimit"`
    FailedJobsHistoryLimit     int    `yaml:"failedJobsHistoryLimit"`
    TTLSecondsAfterFinished    *int32 `yaml:"ttlSecondsAfterFinished"`
    AutoCleanup                bool   `yaml:"autoCleanup"`
}

type NetworkingConfig struct {
    DNSPolicy          v1.DNSPolicy            `yaml:"dnsPolicy"`
    DNSConfig          *v1.PodDNSConfig        `yaml:"dnsConfig"`
    HostNetwork        bool                    `yaml:"hostNetwork"`
    NetworkPolicy      string                  `yaml:"networkPolicy"`
    ServiceMesh        ServiceMeshConfig       `yaml:"serviceMesh"`
}

type SecurityConfig struct {
    PodSecurityContext    *v1.PodSecurityContext    `yaml:"podSecurityContext"`
    SecurityContext       *v1.SecurityContext       `yaml:"securityContext"`
    PodSecurityStandard   string                    `yaml:"podSecurityStandard"`
    NetworkPolicies       []string                  `yaml:"networkPolicies"`
    RBAC                  RBACConfig                `yaml:"rbac"`
}

type MonitoringConfig struct {
    Enabled           bool              `yaml:"enabled"`
    PrometheusMetrics bool              `yaml:"prometheusMetrics"`
    LogLevel          string            `yaml:"logLevel"`
    Tracing           TracingConfig     `yaml:"tracing"`
}
```

## Configuration Methods

### 1. Configuration File (.actrc)

#### Basic Configuration

```yaml
# Basic Kubernetes backend configuration
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "act-runners"
  kubeconfig: "~/.kube/config"

  # Default resource limits
  defaultResources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "2Gi"
      cpu: "1000m"

  # Platform mappings
  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64

    windows-latest:
      image: "mcr.microsoft.com/windows/servercore:ltsc2022"
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"
```

#### Advanced Configuration

```yaml
kubernetes:
  enabled: true
  namespace: "act-runners"
  context: "production-cluster"
  serviceAccount: "act-runner"
  storageClass: "fast-ssd"

  # Image pull configuration
  imagePullSecrets:
    - "docker-registry-secret"
    - "private-registry-secret"

  # Cleanup policy
  cleanupPolicy:
    successfulJobsHistoryLimit: 5
    failedJobsHistoryLimit: 2
    ttlSecondsAfterFinished: 3600
    autoCleanup: true

  # Security configuration
  security:
    podSecurityContext:
      runAsNonRoot: true
      runAsUser: 1001
      fsGroup: 1001
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    podSecurityStandard: "restricted"
    rbac:
      create: true
      rules:
        - apiGroups: [""]
          resources: ["pods", "pods/log"]
          verbs: ["get", "list", "create", "delete"]

  # Networking configuration
  networking:
    dnsPolicy: "ClusterFirst"
    networkPolicy: "act-network-policy"
    serviceMesh:
      enabled: true
      type: "istio"

  # Monitoring configuration
  monitoring:
    enabled: true
    prometheusMetrics: true
    logLevel: "info"
    tracing:
      enabled: true
      jaegerEndpoint: "http://jaeger-collector:14268/api/traces"
```

### 2. Command Line Arguments

#### Basic Usage

```bash
# Enable Kubernetes backend
act --backend kubernetes

# Specify kubeconfig
act --backend kubernetes --kubeconfig ~/.kube/config

# Specify namespace
act --backend kubernetes --namespace act-runners

# Specify context
act --backend kubernetes --context production-cluster
```

#### Advanced Usage

```bash
# Multiple configuration options
act --backend kubernetes \
    --kubeconfig ./cluster-config.yaml \
    --namespace ci-runners \
    --context staging \
    --k8s-service-account act-runner \
    --k8s-storage-class fast-ssd

# Platform overrides
act --backend kubernetes \
    -P ubuntu-latest=k8s:ubuntu:22.04 \
    -P windows-latest=k8s:windows:ltsc2022

# Resource overrides
act --backend kubernetes \
    --k8s-cpu-request 500m \
    --k8s-memory-request 1Gi \
    --k8s-cpu-limit 2000m \
    --k8s-memory-limit 4Gi
```

### 3. Environment Variables

```bash
# Backend selection
export ACT_BACKEND=kubernetes

# Kubernetes configuration
export ACT_K8S_KUBECONFIG=~/.kube/config
export ACT_K8S_NAMESPACE=act-runners
export ACT_K8S_CONTEXT=production
export ACT_K8S_SERVICE_ACCOUNT=act-runner

# Resource configuration
export ACT_K8S_CPU_REQUEST=250m
export ACT_K8S_MEMORY_REQUEST=512Mi
export ACT_K8S_CPU_LIMIT=1000m
export ACT_K8S_MEMORY_LIMIT=2Gi

# Security configuration
export ACT_K8S_RUN_AS_USER=1001
export ACT_K8S_FS_GROUP=1001
export ACT_K8S_SECURITY_CONTEXT_READONLY=true
```

## Platform Configuration

### 1. Standard Platforms

```yaml
kubernetes:
  platforms:
    # Ubuntu variants
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64

    ubuntu-20.04:
      image: "ubuntu:20.04"
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64

    ubuntu-18.04:
      image: "ubuntu:18.04"
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64

    # Windows variants
    windows-latest:
      image: "mcr.microsoft.com/windows/servercore:ltsc2022"
      nodeSelector:
        kubernetes.io/os: windows
        kubernetes.io/arch: amd64
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"

    windows-2019:
      image: "mcr.microsoft.com/windows/servercore:ltsc2019"
      nodeSelector:
        kubernetes.io/os: windows
        kubernetes.io/arch: amd64
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "windows"
          effect: "NoSchedule"

    # macOS (if supported)
    macos-latest:
      image: "macos:monterey"
      nodeSelector:
        kubernetes.io/os: darwin
        kubernetes.io/arch: amd64
      tolerations:
        - key: "os"
          operator: "Equal"
          value: "darwin"
          effect: "NoSchedule"
```

### 2. Specialized Platforms

```yaml
kubernetes:
  platforms:
    # GPU-enabled platforms
    ubuntu-gpu:
      image: "nvidia/cuda:11.8-runtime-ubuntu22.04"
      nodeSelector:
        accelerator: nvidia-tesla-v100
        node-pool: gpu
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      resources:
        limits:
          nvidia.com/gpu: 1
          memory: "16Gi"
          cpu: "8000m"
      runtimeClass: "nvidia"

    # High-memory platforms
    ubuntu-highmem:
      image: "ubuntu:22.04"
      nodeSelector:
        node-type: memory-optimized
      resources:
        requests:
          memory: "16Gi"
          cpu: "4000m"
        limits:
          memory: "64Gi"
          cpu: "16000m"

    # ARM platforms
    ubuntu-arm64:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/arch: arm64
      tolerations:
        - key: "arch"
          operator: "Equal"
          value: "arm64"
          effect: "NoSchedule"

    # Self-hosted runners
    self-hosted:
      image: "act-runner:latest"
      nodeSelector:
        node-type: self-hosted
        environment: production
      tolerations:
        - key: "self-hosted"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: "node-reliability"
                    operator: In
                    values: ["high"]
```

## Storage Configuration

### 1. Workspace Storage

```yaml
kubernetes:
  storage:
    workspace:
      type: "emptyDir" # or "pvc", "hostPath"
      size: "10Gi"
      storageClass: "fast-ssd"
      accessModes:
        - "ReadWriteOnce"

    toolCache:
      type: "pvc"
      size: "50Gi"
      storageClass: "standard"
      accessModes:
        - "ReadWriteMany"
      reclaimPolicy: "Retain"

    artifacts:
      type: "pvc"
      size: "100Gi"
      storageClass: "standard"
      accessModes:
        - "ReadWriteMany"
```

### 2. Volume Templates

```yaml
kubernetes:
  volumeTemplates:
    workspace:
      spec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: "10Gi"
        storageClassName: "fast-ssd"

    cache:
      spec:
        accessModes:
          - "ReadWriteMany"
        resources:
          requests:
            storage: "50Gi"
        storageClassName: "standard"
```

## Security Configuration

### 1. Pod Security Context

```yaml
kubernetes:
  security:
    podSecurityContext:
      runAsNonRoot: true
      runAsUser: 1001
      runAsGroup: 1001
      fsGroup: 1001
      seccompProfile:
        type: "RuntimeDefault"

    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1001
      capabilities:
        drop:
          - "ALL"
        add:
          - "NET_BIND_SERVICE"
```

### 2. RBAC Configuration

```yaml
kubernetes:
  security:
    rbac:
      create: true
      serviceAccount: "act-runner"
      rules:
        - apiGroups: [""]
          resources: ["pods", "pods/log", "pods/exec"]
          verbs: ["get", "list", "create", "delete", "watch"]
        - apiGroups: [""]
          resources: ["configmaps", "secrets"]
          verbs: ["get", "list", "create", "update", "delete"]
        - apiGroups: ["batch"]
          resources: ["jobs"]
          verbs: ["get", "list", "create", "delete"]
```

### 3. Network Policies

```yaml
kubernetes:
  security:
    networkPolicies:
      - name: "act-runner-policy"
        spec:
          podSelector:
            matchLabels:
              app: "act-runner"
          policyTypes:
            - "Ingress"
            - "Egress"
          ingress:
            - from:
                - namespaceSelector:
                    matchLabels:
                      name: "act-system"
          egress:
            - to: []
              ports:
                - protocol: "TCP"
                  port: 443
                - protocol: "TCP"
                  port: 80
```

## Monitoring and Observability

### 1. Metrics Configuration

```yaml
kubernetes:
  monitoring:
    enabled: true
    prometheusMetrics: true
    metricsPort: 8080
    metricsPath: "/metrics"

    # Custom metrics
    customMetrics:
      - name: "act_job_duration_seconds"
        type: "histogram"
        help: "Duration of Act job execution"
      - name: "act_step_duration_seconds"
        type: "histogram"
        help: "Duration of Act step execution"
      - name: "act_jobs_total"
        type: "counter"
        help: "Total number of Act jobs executed"
```

### 2. Logging Configuration

```yaml
kubernetes:
  monitoring:
    logging:
      level: "info"
      format: "json"
      output: "stdout"

      # Log aggregation
      aggregation:
        enabled: true
        endpoint: "http://fluentd:24224"
        format: "fluentd"

      # Log retention
      retention:
        days: 30
        maxSize: "1Gi"
```

### 3. Tracing Configuration

```yaml
kubernetes:
  monitoring:
    tracing:
      enabled: true
      sampler: "probabilistic"
      samplingRate: 0.1
      jaeger:
        endpoint: "http://jaeger-collector:14268/api/traces"
        serviceName: "act-kubernetes-backend"
```

## Configuration Validation

### 1. Schema Validation

```go
func ValidateKubernetesConfig(config *KubernetesConfig) error {
    if config.Namespace == "" {
        return fmt.Errorf("kubernetes namespace is required")
    }

    if len(config.Platforms) == 0 {
        return fmt.Errorf("at least one platform must be configured")
    }

    for name, platform := range config.Platforms {
        if platform.Image == "" {
            return fmt.Errorf("platform %s must specify an image", name)
        }
    }

    return nil
}
```

### 2. Runtime Validation

```go
func (k *KubernetesExecutionEnvironment) validateClusterAccess(ctx context.Context) error {
    // Test cluster connectivity
    _, err := k.client.Discovery().ServerVersion()
    if err != nil {
        return fmt.Errorf("cannot connect to Kubernetes cluster: %w", err)
    }

    // Test namespace access
    _, err = k.client.CoreV1().Namespaces().Get(ctx, k.namespace, metav1.GetOptions{})
    if err != nil {
        return fmt.Errorf("cannot access namespace %s: %w", k.namespace, err)
    }

    // Test RBAC permissions
    return k.validateRBACPermissions(ctx)
}
```

## Configuration Examples

### 1. Development Environment

```yaml
# .actrc for development
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "act-dev"
  kubeconfig: "~/.kube/config"
  context: "minikube"

  defaultResources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "1Gi"
      cpu: "500m"

  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux

  cleanupPolicy:
    autoCleanup: true
    ttlSecondsAfterFinished: 300
```

### 2. Production Environment

```yaml
# .actrc for production
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "act-prod"
  context: "production-cluster"
  serviceAccount: "act-runner-prod"
  storageClass: "fast-ssd"

  imagePullSecrets:
    - "docker-registry-secret"

  defaultResources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"

  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux
        node-pool: guaranteed
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "8Gi"
          cpu: "4000m"

  security:
    podSecurityContext:
      runAsNonRoot: true
      runAsUser: 1001
      fsGroup: 1001
    rbac:
      create: true

  monitoring:
    enabled: true
    prometheusMetrics: true
    tracing:
      enabled: true
      jaegerEndpoint: "http://jaeger-collector:14268/api/traces"

  cleanupPolicy:
    successfulJobsHistoryLimit: 10
    failedJobsHistoryLimit: 5
    ttlSecondsAfterFinished: 7200
```
