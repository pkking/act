# Job Scheduling in Kubernetes Backend

## Overview

This document describes how GitHub Actions jobs are mapped to Kubernetes resources and scheduled for execution in the Kubernetes backend.

## Job-to-Pod Mapping Strategy

### Recommended Approach: One Job = One Pod

Each GitHub Actions job is mapped to a single Kubernetes Pod with the following characteristics:

- **Main Container**: Executes the job steps
- **Init Containers**: Handle environment setup and preparation
- **Sidecar Containers**: Provide logging, monitoring, and auxiliary services
- **Service Containers**: Support workflow-defined services

### Pod Structure

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: act-job-{job-id}
  labels:
    act.nektos.com/workflow: {workflow-name}
    act.nektos.com/job: {job-name}
    act.nektos.com/run-id: {run-id}
    act.nektos.com/backend: kubernetes
spec:
  initContainers:
  - name: setup
    image: act-runner:latest
    command: ["/setup.sh"]
    volumeMounts:
    - name: workspace
      mountPath: /github/workspace
    - name: runner-temp
      mountPath: /runner/temp
    - name: tool-cache
      mountPath: /opt/hostedtoolcache
  
  containers:
  - name: runner
    image: {runs-on-image}
    command: ["/act-runner"]
    env:
    - name: GITHUB_WORKSPACE
      value: /github/workspace
    - name: RUNNER_TEMP
      value: /runner/temp
    - name: RUNNER_TOOL_CACHE
      value: /opt/hostedtoolcache
    volumeMounts:
    - name: workspace
      mountPath: /github/workspace
    - name: runner-temp
      mountPath: /runner/temp
    - name: tool-cache
      mountPath: /opt/hostedtoolcache
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
  
  - name: log-collector
    image: act-log-collector:latest
    volumeMounts:
    - name: runner-temp
      mountPath: /runner/temp
  
  volumes:
  - name: workspace
    emptyDir: {}
  - name: runner-temp
    emptyDir: {}
  - name: tool-cache
    persistentVolumeClaim:
      claimName: act-tool-cache
```

## Resource Management

### 1. Resource Allocation

#### Default Resource Configuration
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "1000m"
```

#### Platform-Specific Resources
```yaml
kubernetes:
  platforms:
    ubuntu-latest:
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
        limits:
          memory: "4Gi"
          cpu: "2000m"
    
    ubuntu-gpu:
      resources:
        requests:
          memory: "4Gi"
          cpu: "2000m"
        limits:
          memory: "16Gi"
          cpu: "8000m"
          nvidia.com/gpu: 1
```

### 2. Storage Management

#### Workspace Storage
- **EmptyDir**: For temporary workspace (default)
- **PVC**: For persistent workspace across job restarts
- **HostPath**: For direct node storage access (advanced use cases)

#### Tool Cache Storage
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: act-tool-cache
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd
```

## Lifecycle Management

### 1. Pod Creation Process

```go
func (k *KubernetesExecutionEnvironment) Create(capAdd []string, capDrop []string) common.Executor {
    return func(ctx context.Context) error {
        // 1. Build pod specification
        pod := k.buildPodSpec(capAdd, capDrop)
        
        // 2. Apply security context
        k.applySecurityContext(pod)
        
        // 3. Configure volumes and mounts
        k.configureVolumes(pod)
        
        // 4. Set resource requirements
        k.setResourceRequirements(pod)
        
        // 5. Create pod in cluster
        _, err := k.client.CoreV1().Pods(k.namespace).Create(ctx, pod, metav1.CreateOptions{})
        if err != nil {
            return fmt.Errorf("failed to create pod: %w", err)
        }
        
        // 6. Wait for pod to be scheduled
        return k.waitForPodScheduled(ctx)
    }
}
```

### 2. Pod Startup Sequence

1. **Pod Scheduling**: Kubernetes scheduler assigns pod to appropriate node
2. **Image Pulling**: Container images are pulled to the node
3. **Init Container Execution**: Setup containers prepare the environment
4. **Main Container Startup**: Runner container starts and waits for commands
5. **Readiness Check**: Pod reports ready status

### 3. Job Execution Flow

```go
func (k *KubernetesExecutionEnvironment) Start(attach bool) common.Executor {
    return func(ctx context.Context) error {
        // 1. Wait for pod ready
        if err := k.waitForPodReady(ctx); err != nil {
            return err
        }
        
        // 2. Setup log streaming
        if attach {
            go k.streamLogs(ctx)
        }
        
        // 3. Initialize runner environment
        return k.initializeRunner(ctx)
    }
}
```

### 4. Cleanup Process

#### Immediate Cleanup
- Remove temporary files and directories
- Stop running processes
- Flush logs and outputs

#### Deferred Cleanup
```yaml
cleanupPolicy:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  ttlSecondsAfterFinished: 3600  # 1 hour
```

## Scheduling Strategies

### 1. Node Selection

#### Basic Node Selection
```yaml
nodeSelector:
  kubernetes.io/os: linux
  kubernetes.io/arch: amd64
```

#### Advanced Affinity Rules
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-type
          operator: In
          values: ["compute-optimized"]
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

### 2. Tolerations and Taints

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "ci-runners"
  effect: "NoSchedule"
- key: "nvidia.com/gpu"
  operator: "Exists"
  effect: "NoSchedule"
```

### 3. Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: act-high-priority
value: 1000
globalDefault: false
description: "High priority for critical CI jobs"
```

## Concurrent Execution

### 1. Matrix Strategy Support

For matrix builds, each combination creates a separate pod:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [14, 16, 18]
```

Results in 6 pods:
- `act-job-{id}-ubuntu-latest-node14`
- `act-job-{id}-ubuntu-latest-node16`
- `act-job-{id}-ubuntu-latest-node18`
- `act-job-{id}-windows-latest-node14`
- `act-job-{id}-windows-latest-node16`
- `act-job-{id}-windows-latest-node18`

### 2. Parallel Job Execution

```go
func (r *runnerImpl) NewPlanExecutor(plan *model.Plan) common.Executor {
    return func(ctx context.Context) error {
        maxParallel := r.config.GetConcurrentJobs()
        
        var stagePipeline []common.Executor
        for _, stage := range plan.Stages {
            var stageExecutor []common.Executor
            
            for _, run := range stage.Runs {
                // Create Kubernetes execution environment for each job
                k8sEnv := NewKubernetesExecutionEnvironment(r.config, run)
                executor := newJobExecutor(run, k8sEnv)
                stageExecutor = append(stageExecutor, executor)
            }
            
            // Execute jobs in parallel within stage
            stagePipeline = append(stagePipeline, 
                common.NewParallelExecutor(maxParallel, stageExecutor...))
        }
        
        // Execute stages sequentially
        return common.NewPipelineExecutor(stagePipeline...)(ctx)
    }
}
```

## Error Handling and Recovery

### 1. Pod Failure Scenarios

- **Image Pull Failures**: Retry with exponential backoff
- **Resource Constraints**: Queue and retry when resources available
- **Node Failures**: Automatic rescheduling to healthy nodes
- **Network Issues**: Retry with circuit breaker pattern

### 2. Graceful Degradation

```go
func (k *KubernetesExecutionEnvironment) handlePodFailure(ctx context.Context, err error) error {
    switch {
    case isImagePullError(err):
        return k.retryWithBackoff(ctx, k.pullImage)
    case isResourceConstraintError(err):
        return k.waitForResources(ctx)
    case isNodeFailureError(err):
        return k.rescheduleToHealthyNode(ctx)
    default:
        return fmt.Errorf("unrecoverable pod failure: %w", err)
    }
}
```
