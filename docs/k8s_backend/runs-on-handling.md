# Runs-On Label Handling in Kubernetes Backend

## Overview

This document describes how the Kubernetes backend processes `runs-on` labels from GitHub Actions workflows and maps them to appropriate Kubernetes scheduling configurations.

## Current Runs-On Processing

### Existing Implementation

The current Act implementation processes `runs-on` labels through the platform mapping system:

```go
func (rc *RunContext) runsOnImage(ctx context.Context) string {
    if rc.Run.Job().RunsOn() == nil {
        common.Logger(ctx).Errorf("'runs-on' key not defined in %s", rc.String())
    }

    for _, platformName := range rc.runsOnPlatformNames(ctx) {
        image := rc.Config.Platforms[strings.ToLower(platformName)]
        if image != "" {
            return image
        }
    }

    return ""
}
```

### Runs-On Label Parsing

```go
func (j *Job) RunsOn() []string {
    switch j.RawRunsOn.Kind {
    case yaml.MappingNode:
        var val struct {
            Group  string
            Labels yaml.Node
        }
        
        if !decodeNode(j.RawRunsOn, &val) {
            return nil
        }
        
        labels := nodeAsStringSlice(val.Labels)
        
        if val.Group != "" {
            labels = append(labels, val.Group)
        }
        
        return labels
    default:
        return nodeAsStringSlice(j.RawRunsOn)
    }
}
```

## Kubernetes Extension

### 1. Enhanced Platform Configuration

#### Extended Configuration Structure

```go
type KubernetesRunsOnConfig struct {
    Image           string                      `yaml:"image"`
    NodeSelector    map[string]string           `yaml:"nodeSelector"`
    Tolerations     []v1.Toleration             `yaml:"tolerations"`
    Affinity        *v1.Affinity                `yaml:"affinity"`
    Resources       v1.ResourceRequirements     `yaml:"resources"`
    RuntimeClass    string                      `yaml:"runtimeClass"`
    SecurityContext *v1.PodSecurityContext      `yaml:"securityContext"`
    ServiceAccount  string                      `yaml:"serviceAccount"`
    Priority        int32                       `yaml:"priority"`
    Labels          map[string]string           `yaml:"labels"`
    Annotations     map[string]string           `yaml:"annotations"`
}
```

#### Configuration Examples

```yaml
kubernetes:
  platforms:
    # Standard Ubuntu runner
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64
        node-pool: standard
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
        limits:
          memory: "4Gi"
          cpu: "2000m"
    
    # Windows runner
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
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "8Gi"
          cpu: "4000m"
    
    # GPU-enabled runner
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
        requests:
          memory: "4Gi"
          cpu: "2000m"
        limits:
          memory: "16Gi"
          cpu: "8000m"
          nvidia.com/gpu: 1
      runtimeClass: "nvidia"
    
    # Self-hosted runner
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

### 2. Label Mapping Strategies

#### Simple Label Mapping

```yaml
# Workflow
runs-on: ubuntu-latest

# Maps to
nodeSelector:
  kubernetes.io/os: linux
  kubernetes.io/arch: amd64
```

#### Multi-Label Mapping

```yaml
# Workflow
runs-on: [self-hosted, linux, x64, gpu]

# Maps to
nodeSelector:
  node-type: self-hosted
  kubernetes.io/os: linux
  kubernetes.io/arch: amd64
  accelerator: gpu
```

#### Expression-Based Mapping

```yaml
# Workflow
runs-on: ${{ matrix.os }}
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]

# Dynamic mapping based on matrix value
```

### 3. Advanced Scheduling Features

#### Node Affinity Configuration

```go
func (k *KubernetesExecutionEnvironment) buildNodeAffinity(config *KubernetesRunsOnConfig) *v1.NodeAffinity {
    if config.Affinity != nil && config.Affinity.NodeAffinity != nil {
        return config.Affinity.NodeAffinity
    }
    
    // Build from nodeSelector
    if len(config.NodeSelector) > 0 {
        var terms []v1.NodeSelectorTerm
        var expressions []v1.NodeSelectorRequirement
        
        for key, value := range config.NodeSelector {
            expressions = append(expressions, v1.NodeSelectorRequirement{
                Key:      key,
                Operator: v1.NodeSelectorOpIn,
                Values:   []string{value},
            })
        }
        
        terms = append(terms, v1.NodeSelectorTerm{
            MatchExpressions: expressions,
        })
        
        return &v1.NodeAffinity{
            RequiredDuringSchedulingIgnoredDuringExecution: &v1.NodeSelector{
                NodeSelectorTerms: terms,
            },
        }
    }
    
    return nil
}
```

#### Pod Anti-Affinity for Load Distribution

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: act.nektos.com/job
            operator: In
            values: ["build"]
        topologyKey: kubernetes.io/hostname
```

#### Topology Spread Constraints

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      act.nektos.com/workflow: ci-pipeline
```

## Implementation Details

### 1. Platform Resolution

```go
func (k *KubernetesExecutionEnvironment) resolvePlatform(ctx context.Context, runsOn []string) (*KubernetesRunsOnConfig, error) {
    // Try exact match first
    for _, label := range runsOn {
        if config, exists := k.config.Platforms[strings.ToLower(label)]; exists {
            return &config, nil
        }
    }
    
    // Try composite matching
    return k.resolveComposite(ctx, runsOn)
}

func (k *KubernetesExecutionEnvironment) resolveComposite(ctx context.Context, runsOn []string) (*KubernetesRunsOnConfig, error) {
    config := &KubernetesRunsOnConfig{
        NodeSelector: make(map[string]string),
        Labels:       make(map[string]string),
        Annotations:  make(map[string]string),
    }
    
    // Map standard labels
    for _, label := range runsOn {
        switch label {
        case "self-hosted":
            config.NodeSelector["node-type"] = "self-hosted"
        case "linux":
            config.NodeSelector["kubernetes.io/os"] = "linux"
        case "windows":
            config.NodeSelector["kubernetes.io/os"] = "windows"
        case "x64":
            config.NodeSelector["kubernetes.io/arch"] = "amd64"
        case "arm64":
            config.NodeSelector["kubernetes.io/arch"] = "arm64"
        case "gpu":
            config.NodeSelector["accelerator"] = "gpu"
        default:
            // Custom label
            config.NodeSelector[label] = "true"
        }
    }
    
    return config, nil
}
```

### 2. Pod Specification Building

```go
func (k *KubernetesExecutionEnvironment) buildPodSpec(config *KubernetesRunsOnConfig) *v1.Pod {
    pod := &v1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:        k.podName,
            Namespace:   k.namespace,
            Labels:      k.buildPodLabels(config),
            Annotations: k.buildPodAnnotations(config),
        },
        Spec: v1.PodSpec{
            RestartPolicy:    v1.RestartPolicyNever,
            NodeSelector:     config.NodeSelector,
            Tolerations:      config.Tolerations,
            Affinity:         config.Affinity,
            RuntimeClassName: &config.RuntimeClass,
            SecurityContext:  config.SecurityContext,
            ServiceAccountName: config.ServiceAccount,
            Priority:         &config.Priority,
        },
    }
    
    // Add containers
    pod.Spec.Containers = k.buildContainers(config)
    pod.Spec.InitContainers = k.buildInitContainers(config)
    pod.Spec.Volumes = k.buildVolumes(config)
    
    return pod
}
```

### 3. Dynamic Configuration

#### Runtime Label Resolution

```go
func (k *KubernetesExecutionEnvironment) resolveRuntimeLabels(ctx context.Context, runsOn []string) ([]string, error) {
    var resolved []string
    
    for _, label := range runsOn {
        // Check if label contains expressions
        if strings.Contains(label, "${{") {
            evaluatedLabel, err := k.runContext.ExprEval.Interpolate(ctx, label)
            if err != nil {
                return nil, fmt.Errorf("failed to evaluate runs-on expression %s: %w", label, err)
            }
            resolved = append(resolved, evaluatedLabel)
        } else {
            resolved = append(resolved, label)
        }
    }
    
    return resolved, nil
}
```

#### Matrix Strategy Integration

```go
func (k *KubernetesExecutionEnvironment) resolveMatrixPlatform(ctx context.Context, matrix map[string]interface{}) (*KubernetesRunsOnConfig, error) {
    runsOn := k.runContext.Run.Job().RunsOn()
    
    // Resolve matrix variables in runs-on
    for i, label := range runsOn {
        if strings.HasPrefix(label, "${{ matrix.") {
            matrixKey := strings.TrimSuffix(strings.TrimPrefix(label, "${{ matrix."), " }}")
            if value, exists := matrix[matrixKey]; exists {
                runsOn[i] = fmt.Sprintf("%v", value)
            }
        }
    }
    
    return k.resolvePlatform(ctx, runsOn)
}
```

## Configuration Examples

### 1. Basic Configuration

```yaml
# .actrc
kubernetes:
  enabled: true
  namespace: "act-runners"
  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux
```

### 2. Multi-Environment Configuration

```yaml
kubernetes:
  platforms:
    # Development environment
    ubuntu-dev:
      image: "ubuntu:22.04"
      nodeSelector:
        environment: development
        node-pool: burstable
      resources:
        requests:
          memory: "512Mi"
          cpu: "250m"
    
    # Production environment
    ubuntu-prod:
      image: "ubuntu:22.04"
      nodeSelector:
        environment: production
        node-pool: guaranteed
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "4Gi"
          cpu: "2000m"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-reliability
                operator: In
                values: ["high"]
```

### 3. Specialized Workload Configuration

```yaml
kubernetes:
  platforms:
    # Machine learning workload
    ml-training:
      image: "tensorflow/tensorflow:latest-gpu"
      nodeSelector:
        workload-type: ml
        accelerator: nvidia-v100
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
      resources:
        limits:
          nvidia.com/gpu: 2
          memory: "32Gi"
          cpu: "16000m"
      runtimeClass: "nvidia"
    
    # Database testing
    db-test:
      image: "ubuntu:22.04"
      nodeSelector:
        workload-type: database
        storage-type: ssd
      tolerations:
      - key: "database-workload"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      resources:
        requests:
          memory: "4Gi"
          cpu: "2000m"
        limits:
          memory: "8Gi"
          cpu: "4000m"
```

## Command Line Integration

### 1. Platform Override

```bash
# Override platform mapping
act -P ubuntu-latest=k8s:ubuntu:22.04

# Multiple platform overrides
act -P ubuntu-latest=k8s:ubuntu:22.04 -P windows-latest=k8s:windows:ltsc2022

# Custom node selector
act -P ubuntu-latest=k8s:ubuntu:22.04:node-pool=compute
```

### 2. Runtime Configuration

```bash
# Specify namespace
act --backend kubernetes --namespace ci-runners

# Custom kubeconfig
act --backend kubernetes --kubeconfig ./cluster-config.yaml

# Debug mode with verbose scheduling info
act --backend kubernetes --verbose --debug-scheduling
```
