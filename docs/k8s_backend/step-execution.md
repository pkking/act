# Step Execution in Kubernetes Backend

## Overview

This document describes how individual GitHub Actions steps are executed within Kubernetes pods, including container selection, environment configuration, and execution strategies.

## Step Execution Strategies

### 1. Container Selection Strategy

#### Default Container Execution
Most steps execute in the main runner container:

```go
type KubernetesStep struct {
    stepModel  *model.Step
    runContext *RunContext
    podClient  kubernetes.Interface
    podName    string
    namespace  string
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
```

#### Action Container Execution
For Docker-based actions, create dedicated containers:

```go
func (ks *KubernetesStep) executeActionStep(ctx context.Context) error {
    action := ks.stepModel.Uses
    
    if isDockerAction(action) {
        return ks.executeInActionContainer(ctx, action)
    }
    
    // For composite or JavaScript actions, execute in main container
    return ks.executeInMainContainer(ctx)
}
```

### 2. Execution Environments

#### Run Steps
Execute shell commands in the main container:

```yaml
containers:
- name: runner
  image: ubuntu:22.04
  command: ["/bin/bash"]
  args: ["-c", "while true; do sleep 30; done"]
  env:
  - name: GITHUB_WORKSPACE
    value: /github/workspace
```

#### Docker Actions
Create sidecar containers for Docker actions:

```yaml
containers:
- name: action-checkout
  image: actions/checkout:v3
  env:
  - name: INPUT_REPOSITORY
    value: "owner/repo"
  - name: INPUT_PATH
    value: "."
  volumeMounts:
  - name: workspace
    mountPath: /github/workspace
```

#### Composite Actions
Execute in main container with step isolation:

```go
func (ks *KubernetesStep) executeCompositeAction(ctx context.Context) error {
    compositeSteps := ks.getCompositeSteps()
    
    for _, step := range compositeSteps {
        if err := ks.executeStep(ctx, step); err != nil {
            if !step.ContinueOnError {
                return err
            }
        }
    }
    
    return nil
}
```

## Environment Management

### 1. Environment Variable Handling

#### Static Environment Variables
Configured via ConfigMaps:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: act-job-env-{job-id}
data:
  GITHUB_WORKSPACE: "/github/workspace"
  GITHUB_REPOSITORY: "owner/repo"
  GITHUB_SHA: "abc123"
  CI: "true"
```

#### Dynamic Environment Variables
Managed through step execution:

```go
func (ks *KubernetesStep) setEnvironmentVariable(ctx context.Context, key, value string) error {
    // Write to environment file for next steps
    envFile := "/runner/temp/_runner_file_commands/set_env"
    content := fmt.Sprintf("%s=%s\n", key, value)
    
    return ks.execInPod(ctx, []string{
        "sh", "-c", 
        fmt.Sprintf("echo '%s' >> %s", content, envFile),
    })
}
```

#### Secret Management
Sensitive data via Kubernetes Secrets:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: act-job-secrets-{job-id}
type: Opaque
data:
  GITHUB_TOKEN: <base64-encoded-token>
  NPM_TOKEN: <base64-encoded-token>
```

### 2. File System Management

#### Directory Structure
```
/github/workspace/     # Working directory (persistent)
/runner/temp/          # Temporary files (shared between steps)
/runner/tool-cache/    # Tool cache (persistent, optional)
/act/                  # Act runtime files
/home/runner/          # Runner home directory
```

#### Volume Mounts
```yaml
volumeMounts:
- name: workspace
  mountPath: /github/workspace
- name: runner-temp
  mountPath: /runner/temp
- name: tool-cache
  mountPath: /runner/tool-cache
- name: runner-home
  mountPath: /home/runner
```

#### File Sharing Between Steps
```go
func (ks *KubernetesStep) shareFile(ctx context.Context, src, dst string) error {
    // Copy file to shared temp directory
    tempPath := filepath.Join("/runner/temp", filepath.Base(dst))
    
    return ks.execInPod(ctx, []string{
        "cp", src, tempPath,
    })
}
```

## Command Execution

### 1. Shell Command Execution

#### Basic Run Step
```go
func (ks *KubernetesStep) executeRunStep(ctx context.Context) error {
    script := ks.stepModel.Run
    shell := ks.getShell()
    workdir := ks.getWorkingDirectory()
    
    command := []string{shell, "-c", script}
    env := ks.buildEnvironment(ctx)
    
    return ks.execInPod(ctx, command, env, workdir)
}
```

#### Multi-line Scripts
```go
func (ks *KubernetesStep) executeMultilineScript(ctx context.Context, script string) error {
    // Create temporary script file
    scriptPath := "/runner/temp/step_script.sh"
    
    // Write script to file
    if err := ks.writeFile(ctx, scriptPath, script, 0755); err != nil {
        return err
    }
    
    // Execute script
    return ks.execInPod(ctx, []string{"bash", scriptPath})
}
```

### 2. Pod Command Execution

#### Kubernetes Exec Implementation
```go
func (ks *KubernetesStep) execInPod(ctx context.Context, command []string, env map[string]string, workdir string) error {
    req := ks.podClient.CoreV1().RESTClient().Post().
        Resource("pods").
        Name(ks.podName).
        Namespace(ks.namespace).
        SubResource("exec")
    
    req.VersionedParams(&v1.PodExecOptions{
        Container: "runner",
        Command:   command,
        Stdin:     false,
        Stdout:    true,
        Stderr:    true,
        TTY:       false,
    }, scheme.ParameterCodec)
    
    exec, err := remotecommand.NewSPDYExecutor(ks.config, "POST", req.URL())
    if err != nil {
        return err
    }
    
    return exec.Stream(remotecommand.StreamOptions{
        Stdout: ks.stdout,
        Stderr: ks.stderr,
    })
}
```

#### Command with Environment
```go
func (ks *KubernetesStep) execWithEnv(ctx context.Context, command []string, env map[string]string) error {
    // Prepare environment variables
    var envVars []string
    for k, v := range env {
        envVars = append(envVars, fmt.Sprintf("%s=%s", k, v))
    }
    
    // Construct command with environment
    fullCommand := append([]string{"env"}, envVars...)
    fullCommand = append(fullCommand, command...)
    
    return ks.execInPod(ctx, fullCommand)
}
```

## Action Execution

### 1. JavaScript Actions

#### Node.js Runtime Setup
```go
func (ks *KubernetesStep) executeJavaScriptAction(ctx context.Context, action *model.Action) error {
    nodePath := ks.runContext.GetNodeToolFullPath(ctx)
    actionMain := filepath.Join("/act/actions", action.Runs.Main)
    
    command := []string{nodePath, actionMain}
    env := ks.buildActionEnvironment(ctx, action)
    
    return ks.execInPod(ctx, command, env)
}
```

#### Pre and Post Steps
```go
func (ks *KubernetesStep) executeActionWithHooks(ctx context.Context, action *model.Action) error {
    // Execute pre step if exists
    if action.Runs.Pre != "" {
        if err := ks.executePreStep(ctx, action); err != nil {
            return err
        }
    }
    
    // Execute main step
    if err := ks.executeMainStep(ctx, action); err != nil {
        return err
    }
    
    // Execute post step if exists
    if action.Runs.Post != "" {
        return ks.executePostStep(ctx, action)
    }
    
    return nil
}
```

### 2. Docker Actions

#### Container Creation
```go
func (ks *KubernetesStep) executeDockerAction(ctx context.Context, image string) error {
    // Create action container spec
    container := &v1.Container{
        Name:  fmt.Sprintf("action-%s", ks.stepModel.ID),
        Image: image,
        Env:   ks.buildActionEnvironment(ctx),
        VolumeMounts: []v1.VolumeMount{
            {
                Name:      "workspace",
                MountPath: "/github/workspace",
            },
        },
    }
    
    // Add container to pod
    return ks.addContainerToPod(ctx, container)
}
```

#### Action Input/Output Handling
```go
func (ks *KubernetesStep) buildActionEnvironment(ctx context.Context, action *model.Action) map[string]string {
    env := make(map[string]string)
    
    // Add action inputs as INPUT_* environment variables
    for key, value := range ks.stepModel.With {
        envKey := fmt.Sprintf("INPUT_%s", strings.ToUpper(key))
        env[envKey] = value
    }
    
    // Add GitHub context
    env["GITHUB_WORKSPACE"] = "/github/workspace"
    env["GITHUB_REPOSITORY"] = ks.runContext.Config.GitHubInstance
    
    return env
}
```

## Service Containers

### 1. Service Definition

#### Service Container Spec
```yaml
containers:
- name: redis
  image: redis:6
  ports:
  - containerPort: 6379
    name: redis
  env:
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-secret
        key: password
```

#### Service Discovery
```go
func (ks *KubernetesStep) configureServiceContainers(ctx context.Context) error {
    services := ks.runContext.Run.Job().Services
    
    for name, service := range services {
        // Create service container
        container := ks.buildServiceContainer(name, service)
        
        // Add to pod spec
        if err := ks.addServiceContainer(ctx, container); err != nil {
            return err
        }
        
        // Configure service networking
        if err := ks.configureServiceNetworking(ctx, name, service); err != nil {
            return err
        }
    }
    
    return nil
}
```

### 2. Network Configuration

#### Service Communication
```go
func (ks *KubernetesStep) configureServiceNetworking(ctx context.Context, name string, service *model.ContainerSpec) error {
    // Services are accessible via localhost in the same pod
    env := map[string]string{
        fmt.Sprintf("%s_HOST", strings.ToUpper(name)): "localhost",
    }
    
    // Add port mappings
    for _, port := range service.Ports {
        portKey := fmt.Sprintf("%s_PORT", strings.ToUpper(name))
        env[portKey] = port
    }
    
    return ks.updateEnvironment(ctx, env)
}
```

## Error Handling and Debugging

### 1. Step Failure Handling

#### Continue on Error
```go
func (ks *KubernetesStep) executeWithErrorHandling(ctx context.Context) error {
    err := ks.main()(ctx)
    
    if err != nil && ks.stepModel.ContinueOnError {
        ks.runContext.StepResults[ks.stepModel.ID] = &model.StepResult{
            Outcome:    model.StepStatusFailure,
            Conclusion: model.StepStatusSuccess, // Continue despite failure
        }
        return nil
    }
    
    return err
}
```

#### Timeout Handling
```go
func (ks *KubernetesStep) executeWithTimeout(ctx context.Context) error {
    timeout := ks.getTimeout()
    if timeout > 0 {
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, timeout)
        defer cancel()
    }
    
    return ks.main()(ctx)
}
```

### 2. Debugging Support

#### Log Collection
```go
func (ks *KubernetesStep) collectLogs(ctx context.Context) error {
    req := ks.podClient.CoreV1().Pods(ks.namespace).GetLogs(ks.podName, &v1.PodLogOptions{
        Container: "runner",
        Follow:    true,
    })
    
    stream, err := req.Stream(ctx)
    if err != nil {
        return err
    }
    defer stream.Close()
    
    // Stream logs to output
    _, err = io.Copy(ks.stdout, stream)
    return err
}
```

#### Step Introspection
```go
func (ks *KubernetesStep) debugStep(ctx context.Context) error {
    // Collect environment variables
    env, err := ks.getEnvironment(ctx)
    if err != nil {
        return err
    }
    
    // Collect file system state
    files, err := ks.listFiles(ctx, "/github/workspace")
    if err != nil {
        return err
    }
    
    // Log debug information
    ks.logger.WithFields(logrus.Fields{
        "step":        ks.stepModel.ID,
        "environment": env,
        "files":       files,
    }).Debug("Step debug information")
    
    return nil
}
```
