# Kubernetes Backend Implementation Guide

## Overview

This document provides technical implementation details for the Kubernetes backend, including code structure, interfaces, and integration points.

## Implementation Architecture

### 1. Core Components Structure

```
pkg/
├── runner/
│   ├── kubernetes_execution_environment.go
│   ├── kubernetes_step.go
│   ├── kubernetes_job_executor.go
│   └── kubernetes_config.go
├── container/
│   └── kubernetes_container.go
└── k8s/
    ├── client.go
    ├── pod_manager.go
    ├── resource_manager.go
    └── utils.go
```

### 2. Interface Implementation

#### KubernetesExecutionEnvironment

```go
package runner

import (
    "context"
    "fmt"
    "path/filepath"
    
    "github.com/nektos/act/pkg/common"
    "github.com/nektos/act/pkg/container"
    "github.com/sirupsen/logrus"
    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type KubernetesExecutionEnvironment struct {
    client       kubernetes.Interface
    config       *KubernetesConfig
    podName      string
    namespace    string
    logger       *logrus.Entry
    podSpec      *v1.Pod
    runContext   *RunContext
    
    // Internal state
    isCreated    bool
    isStarted    bool
    containerID  string
}

func NewKubernetesExecutionEnvironment(config *KubernetesConfig, runContext *RunContext) *KubernetesExecutionEnvironment {
    return &KubernetesExecutionEnvironment{
        config:     config,
        runContext: runContext,
        namespace:  config.Namespace,
        podName:    generatePodName(runContext),
        logger:     logrus.WithField("component", "k8s-execution-env"),
    }
}

// Implement ExecutionsEnvironment interface
func (k *KubernetesExecutionEnvironment) Create(capAdd []string, capDrop []string) common.Executor {
    return func(ctx context.Context) error {
        if k.isCreated {
            return nil
        }
        
        // Initialize Kubernetes client
        if err := k.initializeClient(ctx); err != nil {
            return fmt.Errorf("failed to initialize Kubernetes client: %w", err)
        }
        
        // Build pod specification
        pod, err := k.buildPodSpec(ctx, capAdd, capDrop)
        if err != nil {
            return fmt.Errorf("failed to build pod spec: %w", err)
        }
        
        // Create pod in cluster
        createdPod, err := k.client.CoreV1().Pods(k.namespace).Create(ctx, pod, metav1.CreateOptions{})
        if err != nil {
            return fmt.Errorf("failed to create pod: %w", err)
        }
        
        k.podSpec = createdPod
        k.isCreated = true
        k.logger.Infof("Created pod %s in namespace %s", k.podName, k.namespace)
        
        return nil
    }
}

func (k *KubernetesExecutionEnvironment) Start(attach bool) common.Executor {
    return func(ctx context.Context) error {
        if !k.isCreated {
            return fmt.Errorf("pod not created")
        }
        
        if k.isStarted {
            return nil
        }
        
        // Wait for pod to be ready
        if err := k.waitForPodReady(ctx); err != nil {
            return fmt.Errorf("pod failed to become ready: %w", err)
        }
        
        // Setup log streaming if requested
        if attach {
            go k.streamLogs(ctx)
        }
        
        k.isStarted = true
        k.logger.Infof("Pod %s is ready and started", k.podName)
        
        return nil
    }
}

func (k *KubernetesExecutionEnvironment) Exec(command []string, env map[string]string, user, workdir string) common.Executor {
    return func(ctx context.Context) error {
        return k.execInPod(ctx, command, env, user, workdir)
    }
}

func (k *KubernetesExecutionEnvironment) Remove() common.Executor {
    return func(ctx context.Context) error {
        if !k.isCreated {
            return nil
        }
        
        deletePolicy := metav1.DeletePropagationForeground
        err := k.client.CoreV1().Pods(k.namespace).Delete(ctx, k.podName, metav1.DeleteOptions{
            PropagationPolicy: &deletePolicy,
        })
        
        if err != nil {
            k.logger.WithError(err).Warnf("Failed to delete pod %s", k.podName)
            return err
        }
        
        k.logger.Infof("Deleted pod %s", k.podName)
        return nil
    }
}

// Additional ExecutionsEnvironment methods
func (k *KubernetesExecutionEnvironment) ToContainerPath(hostPath string) string {
    // Convert host path to container path
    return filepath.Join("/github/workspace", filepath.Base(hostPath))
}

func (k *KubernetesExecutionEnvironment) GetActPath() string {
    return "/act"
}

func (k *KubernetesExecutionEnvironment) GetPathVariableName() string {
    return "PATH"
}

func (k *KubernetesExecutionEnvironment) DefaultPathVariable() string {
    return "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
}

func (k *KubernetesExecutionEnvironment) JoinPathVariable(paths ...string) string {
    return strings.Join(paths, ":")
}

func (k *KubernetesExecutionEnvironment) GetRunnerContext(ctx context.Context) map[string]interface{} {
    return map[string]interface{}{
        "os":   "Linux",
        "arch": "X64",
        "name": "Kubernetes Runner",
        "tool_cache": "/opt/hostedtoolcache",
    }
}

func (k *KubernetesExecutionEnvironment) IsEnvironmentCaseInsensitive() bool {
    return false
}
```

### 3. Pod Management Implementation

```go
func (k *KubernetesExecutionEnvironment) buildPodSpec(ctx context.Context, capAdd []string, capDrop []string) (*v1.Pod, error) {
    // Resolve platform configuration
    runsOn := k.runContext.Run.Job().RunsOn()
    platformConfig, err := k.resolvePlatformConfig(ctx, runsOn)
    if err != nil {
        return nil, err
    }
    
    // Build pod metadata
    pod := &v1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      k.podName,
            Namespace: k.namespace,
            Labels:    k.buildPodLabels(),
            Annotations: k.buildPodAnnotations(),
        },
        Spec: v1.PodSpec{
            RestartPolicy:      v1.RestartPolicyNever,
            ServiceAccountName: k.config.ServiceAccount,
            NodeSelector:       platformConfig.NodeSelector,
            Tolerations:        platformConfig.Tolerations,
            Affinity:          platformConfig.Affinity,
            SecurityContext:   platformConfig.SecurityContext,
        },
    }
    
    // Add init containers
    pod.Spec.InitContainers = k.buildInitContainers(platformConfig)
    
    // Add main containers
    pod.Spec.Containers = k.buildContainers(platformConfig, capAdd, capDrop)
    
    // Add volumes
    pod.Spec.Volumes = k.buildVolumes()
    
    // Add image pull secrets
    if len(k.config.ImagePullSecrets) > 0 {
        for _, secret := range k.config.ImagePullSecrets {
            pod.Spec.ImagePullSecrets = append(pod.Spec.ImagePullSecrets, v1.LocalObjectReference{
                Name: secret,
            })
        }
    }
    
    return pod, nil
}

func (k *KubernetesExecutionEnvironment) buildContainers(platformConfig *KubernetesRunsOnConfig, capAdd []string, capDrop []string) []v1.Container {
    containers := []v1.Container{
        {
            Name:    "runner",
            Image:   platformConfig.Image,
            Command: []string{"/bin/bash"},
            Args:    []string{"-c", "while true; do sleep 30; done"},
            Env:     k.buildEnvironmentVariables(),
            VolumeMounts: []v1.VolumeMount{
                {
                    Name:      "workspace",
                    MountPath: "/github/workspace",
                },
                {
                    Name:      "runner-temp",
                    MountPath: "/runner/temp",
                },
                {
                    Name:      "tool-cache",
                    MountPath: "/opt/hostedtoolcache",
                },
            },
            Resources: platformConfig.Resources,
            SecurityContext: k.buildSecurityContext(capAdd, capDrop),
        },
    }
    
    // Add service containers if defined
    serviceContainers := k.buildServiceContainers()
    containers = append(containers, serviceContainers...)
    
    return containers
}

func (k *KubernetesExecutionEnvironment) buildVolumes() []v1.Volume {
    volumes := []v1.Volume{
        {
            Name: "workspace",
            VolumeSource: v1.VolumeSource{
                EmptyDir: &v1.EmptyDirVolumeSource{},
            },
        },
        {
            Name: "runner-temp",
            VolumeSource: v1.VolumeSource{
                EmptyDir: &v1.EmptyDirVolumeSource{},
            },
        },
    }
    
    // Add tool cache volume
    if k.config.Storage.ToolCache.Type == "pvc" {
        volumes = append(volumes, v1.Volume{
            Name: "tool-cache",
            VolumeSource: v1.VolumeSource{
                PersistentVolumeClaim: &v1.PersistentVolumeClaimVolumeSource{
                    ClaimName: "act-tool-cache",
                },
            },
        })
    } else {
        volumes = append(volumes, v1.Volume{
            Name: "tool-cache",
            VolumeSource: v1.VolumeSource{
                EmptyDir: &v1.EmptyDirVolumeSource{},
            },
        })
    }
    
    return volumes
}
```

### 4. Command Execution Implementation

```go
func (k *KubernetesExecutionEnvironment) execInPod(ctx context.Context, command []string, env map[string]string, user, workdir string) error {
    // Prepare exec request
    req := k.client.CoreV1().RESTClient().Post().
        Resource("pods").
        Name(k.podName).
        Namespace(k.namespace).
        SubResource("exec")
    
    // Build exec options
    execOptions := &v1.PodExecOptions{
        Container: "runner",
        Command:   k.buildExecCommand(command, env, user, workdir),
        Stdin:     false,
        Stdout:    true,
        Stderr:    true,
        TTY:       false,
    }
    
    req.VersionedParams(execOptions, scheme.ParameterCodec)
    
    // Create SPDY executor
    exec, err := remotecommand.NewSPDYExecutor(k.config.RestConfig, "POST", req.URL())
    if err != nil {
        return fmt.Errorf("failed to create executor: %w", err)
    }
    
    // Setup streams
    stdout := &bytes.Buffer{}
    stderr := &bytes.Buffer{}
    
    // Execute command
    err = exec.Stream(remotecommand.StreamOptions{
        Stdout: stdout,
        Stderr: stderr,
    })
    
    // Log output
    if stdout.Len() > 0 {
        k.logger.Info(stdout.String())
    }
    if stderr.Len() > 0 {
        k.logger.Error(stderr.String())
    }
    
    return err
}

func (k *KubernetesExecutionEnvironment) buildExecCommand(command []string, env map[string]string, user, workdir string) []string {
    var execCommand []string
    
    // Add environment variables
    if len(env) > 0 {
        execCommand = append(execCommand, "env")
        for key, value := range env {
            execCommand = append(execCommand, fmt.Sprintf("%s=%s", key, value))
        }
    }
    
    // Change working directory if specified
    if workdir != "" {
        execCommand = append(execCommand, "sh", "-c", fmt.Sprintf("cd %s && %s", workdir, strings.Join(command, " ")))
    } else {
        execCommand = append(execCommand, command...)
    }
    
    return execCommand
}
```

### 5. Step Implementation

```go
package runner

type KubernetesStep struct {
    stepModel  *model.Step
    runContext *RunContext
    k8sEnv     *KubernetesExecutionEnvironment
}

func NewKubernetesStep(stepModel *model.Step, runContext *RunContext, k8sEnv *KubernetesExecutionEnvironment) *KubernetesStep {
    return &KubernetesStep{
        stepModel:  stepModel,
        runContext: runContext,
        k8sEnv:     k8sEnv,
    }
}

func (ks *KubernetesStep) pre() common.Executor {
    return func(ctx context.Context) error {
        // Setup step environment
        return ks.setupStepEnvironment(ctx)
    }
}

func (ks *KubernetesStep) main() common.Executor {
    return func(ctx context.Context) error {
        switch {
        case ks.stepModel.Uses != "":
            return ks.executeActionStep(ctx)
        case ks.stepModel.Run != "":
            return ks.executeRunStep(ctx)
        default:
            return fmt.Errorf("step must have either 'uses' or 'run'")
        }
    }
}

func (ks *KubernetesStep) post() common.Executor {
    return func(ctx context.Context) error {
        // Cleanup step resources
        return ks.cleanupStepResources(ctx)
    }
}

func (ks *KubernetesStep) executeRunStep(ctx context.Context) error {
    script := ks.stepModel.Run
    shell := ks.getShell()
    workdir := ks.getWorkingDirectory()
    env := ks.buildStepEnvironment(ctx)
    
    command := []string{shell, "-c", script}
    
    return ks.k8sEnv.execInPod(ctx, command, env, "", workdir)
}

func (ks *KubernetesStep) executeActionStep(ctx context.Context) error {
    actionRef := ks.stepModel.Uses
    
    // Parse action reference
    action, err := ks.resolveAction(ctx, actionRef)
    if err != nil {
        return err
    }
    
    switch action.Runs.Using {
    case model.ActionRunsUsingNode12, model.ActionRunsUsingNode16, model.ActionRunsUsingNode20:
        return ks.executeJavaScriptAction(ctx, action)
    case model.ActionRunsUsingDocker:
        return ks.executeDockerAction(ctx, action)
    case model.ActionRunsUsingComposite:
        return ks.executeCompositeAction(ctx, action)
    default:
        return fmt.Errorf("unsupported action type: %s", action.Runs.Using)
    }
}
```

## Integration Points

### 1. Configuration Integration

```go
func (r *runnerImpl) configure() (Runner, error) {
    // Check if Kubernetes backend is enabled
    if r.config.Backend == "kubernetes" || (r.config.KubernetesConfig != nil && r.config.KubernetesConfig.Enabled) {
        return r.configureKubernetesBackend()
    }
    
    return r.configureDefaultBackend()
}

func (r *runnerImpl) configureKubernetesBackend() (Runner, error) {
    // Validate Kubernetes configuration
    if err := ValidateKubernetesConfig(r.config.KubernetesConfig); err != nil {
        return nil, fmt.Errorf("invalid Kubernetes configuration: %w", err)
    }
    
    // Initialize Kubernetes client
    client, err := NewKubernetesClient(r.config.KubernetesConfig)
    if err != nil {
        return nil, fmt.Errorf("failed to create Kubernetes client: %w", err)
    }
    
    r.k8sClient = client
    return r, nil
}
```

### 2. Platform Resolution

```go
func (k *KubernetesExecutionEnvironment) resolvePlatformConfig(ctx context.Context, runsOn []string) (*KubernetesRunsOnConfig, error) {
    // Try exact platform match first
    for _, platform := range runsOn {
        if config, exists := k.config.Platforms[strings.ToLower(platform)]; exists {
            return &config, nil
        }
    }
    
    // Try composite platform resolution
    return k.resolveCompositePlatform(ctx, runsOn)
}

func (k *KubernetesExecutionEnvironment) resolveCompositePlatform(ctx context.Context, runsOn []string) (*KubernetesRunsOnConfig, error) {
    config := &KubernetesRunsOnConfig{
        NodeSelector: make(map[string]string),
        Resources:    k.config.DefaultResources,
    }
    
    // Map standard GitHub runner labels to Kubernetes node selectors
    for _, label := range runsOn {
        switch strings.ToLower(label) {
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
            // Custom label - add as-is
            config.NodeSelector[label] = "true"
        }
    }
    
    // Set default image if not specified
    if config.Image == "" {
        config.Image = k.getDefaultImage(runsOn)
    }
    
    return config, nil
}
```

This implementation provides a solid foundation for the Kubernetes backend while maintaining compatibility with the existing Act architecture.
