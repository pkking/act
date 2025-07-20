# Kubernetes Backend Usage Examples

## Overview

This document provides practical examples of using the Act Kubernetes backend for various scenarios and use cases.

## Basic Usage Examples

### 1. Simple Workflow Execution

#### Workflow File (.github/workflows/ci.yml)
```yaml
name: CI Pipeline
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
    - name: Install dependencies
      run: npm install
    - name: Run tests
      run: npm test
```

#### Command Execution
```bash
# Basic execution with Kubernetes backend
act --backend kubernetes

# With custom namespace
act --backend kubernetes --namespace ci-runners

# With specific kubeconfig
act --backend kubernetes --kubeconfig ~/.kube/staging-config
```

### 2. Multi-Platform Testing

#### Workflow File
```yaml
name: Multi-Platform Tests
on: [push]

jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    runs-on: ${{ matrix.os }}
    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js ${{ matrix.node }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node }}
    - name: Run tests
      run: npm test
```

#### Configuration (.actrc)
```yaml
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "multi-platform-tests"
  
  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        kubernetes.io/os: linux
    
    windows-latest:
      image: "mcr.microsoft.com/windows/servercore:ltsc2022"
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
      - key: "os"
        operator: "Equal"
        value: "windows"
        effect: "NoSchedule"
    
    macos-latest:
      image: "macos:monterey"
      nodeSelector:
        kubernetes.io/os: darwin
      tolerations:
      - key: "os"
        operator: "Equal"
        value: "darwin"
        effect: "NoSchedule"
```

## Advanced Use Cases

### 1. GPU-Accelerated Machine Learning

#### Workflow File
```yaml
name: ML Training Pipeline
on: [push]

jobs:
  train:
    runs-on: [self-hosted, gpu, cuda]
    steps:
    - uses: actions/checkout@v3
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    - name: Install dependencies
      run: |
        pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
        pip install -r requirements.txt
    - name: Check GPU availability
      run: |
        nvidia-smi
        python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}')"
    - name: Train model
      run: python train.py --epochs 100 --batch-size 32
    - name: Upload model artifacts
      uses: actions/upload-artifact@v3
      with:
        name: trained-model
        path: models/
```

#### Configuration
```yaml
kubernetes:
  platforms:
    gpu-runner:
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
          memory: "8Gi"
          cpu: "4000m"
        limits:
          memory: "32Gi"
          cpu: "16000m"
          nvidia.com/gpu: 2
      runtimeClass: "nvidia"
```

### 2. Database Integration Testing

#### Workflow File
```yaml
name: Database Tests
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    - name: Install dependencies
      run: |
        pip install psycopg2-binary redis pytest
        pip install -r requirements.txt
    - name: Run database tests
      env:
        DATABASE_URL: postgresql://postgres:postgres@postgres:5432/testdb
        REDIS_URL: redis://redis:6379
      run: pytest tests/
```

### 3. Self-Hosted Runner Configuration

#### Workflow File
```yaml
name: Self-Hosted Build
on: [push]

jobs:
  build:
    runs-on: [self-hosted, linux, x64, build-server]
    steps:
    - uses: actions/checkout@v3
    - name: Build application
      run: |
        make clean
        make build
        make package
    - name: Run security scan
      run: |
        ./security-scanner --target ./dist/
    - name: Deploy to staging
      if: github.ref == 'refs/heads/main'
      run: |
        ./deploy.sh staging
```

#### Configuration
```yaml
kubernetes:
  platforms:
    self-hosted:
      image: "build-server:latest"
      nodeSelector:
        node-type: self-hosted
        environment: production
        build-tools: available
      tolerations:
      - key: "self-hosted"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      resources:
        requests:
          memory: "4Gi"
          cpu: "2000m"
        limits:
          memory: "16Gi"
          cpu: "8000m"
      securityContext:
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
```

## Environment-Specific Examples

### 1. Development Environment

#### Configuration
```yaml
# .actrc.dev
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "act-dev"
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
        environment: development
  
  cleanupPolicy:
    autoCleanup: true
    ttlSecondsAfterFinished: 300
```

#### Usage
```bash
# Use development configuration
act --config .actrc.dev --backend kubernetes

# Quick test with minimal resources
act --backend kubernetes --k8s-cpu-limit 200m --k8s-memory-limit 512Mi
```

### 2. Staging Environment

#### Configuration
```yaml
# .actrc.staging
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "act-staging"
  context: "staging-cluster"
  serviceAccount: "act-runner-staging"
  
  imagePullSecrets:
  - "staging-registry-secret"
  
  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        environment: staging
        node-pool: standard
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
        limits:
          memory: "4Gi"
          cpu: "2000m"
  
  monitoring:
    enabled: true
    prometheusMetrics: true
  
  cleanupPolicy:
    successfulJobsHistoryLimit: 5
    failedJobsHistoryLimit: 2
    ttlSecondsAfterFinished: 1800
```

### 3. Production Environment

#### Configuration
```yaml
# .actrc.prod
backend: kubernetes

kubernetes:
  enabled: true
  namespace: "act-prod"
  context: "production-cluster"
  serviceAccount: "act-runner-prod"
  storageClass: "fast-ssd"
  
  imagePullSecrets:
  - "prod-registry-secret"
  
  platforms:
    ubuntu-latest:
      image: "ubuntu:22.04"
      nodeSelector:
        environment: production
        node-pool: guaranteed
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "8Gi"
          cpu: "4000m"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-reliability
                operator: In
                values: ["high"]
  
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

## Specialized Workflows

### 1. Container Image Building

#### Workflow File
```yaml
name: Container Build
on: [push]

jobs:
  build:
    runs-on: [self-hosted, docker]
    steps:
    - uses: actions/checkout@v3
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    - name: Login to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

#### Configuration
```yaml
kubernetes:
  platforms:
    docker-builder:
      image: "docker:dind"
      nodeSelector:
        docker-enabled: "true"
      securityContext:
        privileged: true
      resources:
        requests:
          memory: "2Gi"
          cpu: "1000m"
        limits:
          memory: "8Gi"
          cpu: "4000m"
```

### 2. Performance Testing

#### Workflow File
```yaml
name: Performance Tests
on: [push]

jobs:
  performance:
    runs-on: [self-hosted, performance]
    steps:
    - uses: actions/checkout@v3
    - name: Setup performance testing tools
      run: |
        apt-get update
        apt-get install -y apache2-utils wrk
    - name: Start application
      run: |
        docker-compose up -d
        sleep 30
    - name: Run load tests
      run: |
        ab -n 10000 -c 100 http://localhost:8080/
        wrk -t12 -c400 -d30s http://localhost:8080/
    - name: Collect metrics
      run: |
        docker stats --no-stream > performance-metrics.txt
    - name: Upload results
      uses: actions/upload-artifact@v3
      with:
        name: performance-results
        path: performance-metrics.txt
```

## Troubleshooting Examples

### 1. Debug Mode Execution

```bash
# Enable debug logging
act --backend kubernetes --verbose --debug

# Check pod status
kubectl get pods -n act-runners -l act.nektos.com/workflow=ci

# View pod logs
kubectl logs -n act-runners -l act.nektos.com/job=test --follow

# Describe pod for troubleshooting
kubectl describe pod -n act-runners <pod-name>
```

### 2. Resource Monitoring

```bash
# Monitor resource usage
kubectl top pods -n act-runners

# Check node resources
kubectl top nodes

# View events
kubectl get events -n act-runners --sort-by='.lastTimestamp'
```

### 3. Configuration Validation

```bash
# Validate configuration
act --backend kubernetes --dry-run

# Test cluster connectivity
kubectl cluster-info

# Verify RBAC permissions
kubectl auth can-i create pods --namespace act-runners
```
