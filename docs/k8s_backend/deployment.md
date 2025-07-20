# Kubernetes Backend Deployment Guide

## Overview

This document provides guidance on deploying and configuring the Kubernetes backend for Act in various environments.

## Prerequisites

### 1. Kubernetes Cluster Requirements

#### Minimum Requirements
- Kubernetes version 1.20+
- RBAC enabled
- Container runtime (Docker, containerd, or CRI-O)
- Persistent storage support (for tool cache and artifacts)

#### Recommended Requirements
- Kubernetes version 1.24+
- Multiple node pools for different workload types
- Network policies support
- Monitoring stack (Prometheus, Grafana)
- Log aggregation (ELK, Fluentd)

### 2. Client Requirements

#### Act CLI
- Act version with Kubernetes backend support
- kubectl configured with cluster access
- Appropriate RBAC permissions

#### Dependencies
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Act (with Kubernetes support)
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | bash
```

## Cluster Setup

### 1. Namespace Configuration

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: act-runners
  labels:
    name: act-runners
    purpose: github-actions-execution
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: act-runners-quota
  namespace: act-runners
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    persistentvolumeclaims: "10"
```

### 2. RBAC Configuration

```yaml
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: act-runner
  namespace: act-runners
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: act-runners
  name: act-runner-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "create", "delete", "watch"]
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "create"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "create", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: act-runner-binding
  namespace: act-runners
subjects:
- kind: ServiceAccount
  name: act-runner
  namespace: act-runners
roleRef:
  kind: Role
  name: act-runner-role
  apiGroup: rbac.authorization.k8s.io
```

### 3. Storage Configuration

```yaml
# storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: act-fast-ssd
provisioner: kubernetes.io/gce-pd  # Adjust for your cloud provider
parameters:
  type: pd-ssd
  replication-type: none
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: act-tool-cache
  namespace: act-runners
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: act-fast-ssd
```

### 4. Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: act-runner-policy
  namespace: act-runners
spec:
  podSelector:
    matchLabels:
      app: act-runner
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: act-system
  egress:
  - {} # Allow all egress for now
  # Restrict as needed:
  # - to:
  #   - namespaceSelector:
  #       matchLabels:
  #         name: kube-system
  #   ports:
  #   - protocol: TCP
  #     port: 53
  #   - protocol: UDP
  #     port: 53
```

## Node Configuration

### 1. Node Labels and Taints

```bash
# Label nodes for different workload types
kubectl label nodes node-1 node-pool=standard
kubectl label nodes node-2 node-pool=compute-optimized
kubectl label nodes node-3 node-pool=gpu accelerator=nvidia-tesla-v100

# Taint GPU nodes
kubectl taint nodes node-3 nvidia.com/gpu=true:NoSchedule

# Taint Windows nodes (if applicable)
kubectl taint nodes windows-node-1 os=windows:NoSchedule
```

### 2. Runtime Classes

```yaml
# runtime-classes.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

## Security Configuration

### 1. Pod Security Standards

```yaml
# pod-security-policy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: act-runners
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 2. Security Context Constraints (OpenShift)

```yaml
# security-context-constraint.yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: act-runner-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegedContainer: false
allowedCapabilities: []
defaultAddCapabilities: []
fsGroup:
  type: MustRunAs
  ranges:
  - min: 1000
  - max: 65535
runAsUser:
  type: MustRunAsNonRoot
seLinuxContext:
  type: MustRunAs
users:
- system:serviceaccount:act-runners:act-runner
```

## Monitoring and Observability

### 1. Prometheus Monitoring

```yaml
# monitoring.yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: act-runner-metrics
  namespace: act-runners
spec:
  selector:
    matchLabels:
      app: act-runner
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: act-runner-alerts
  namespace: act-runners
spec:
  groups:
  - name: act-runner
    rules:
    - alert: ActRunnerPodFailed
      expr: kube_pod_status_phase{namespace="act-runners", phase="Failed"} > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Act runner pod failed"
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has failed"
    
    - alert: ActRunnerHighResourceUsage
      expr: |
        (
          sum(rate(container_cpu_usage_seconds_total{namespace="act-runners"}[5m])) by (pod) /
          sum(kube_pod_container_resource_limits{namespace="act-runners", resource="cpu"}) by (pod)
        ) > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage in Act runner pod"
        description: "Pod {{ $labels.pod }} is using more than 90% of its CPU limit"
```

### 2. Logging Configuration

```yaml
# logging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: act-runners
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*act-runner*.log
      pos_file /var/log/fluentd-act-runner.log.pos
      tag kubernetes.act-runner
      format json
    </source>
    
    <match kubernetes.act-runner>
      @type elasticsearch
      host elasticsearch.logging.svc.cluster.local
      port 9200
      index_name act-runner-logs
    </match>
```

## Configuration Examples

### 1. Development Environment

```yaml
# dev-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: act-config-dev
  namespace: act-runners
data:
  config.yaml: |
    backend: kubernetes
    kubernetes:
      enabled: true
      namespace: act-runners
      platforms:
        ubuntu-latest:
          image: ubuntu:22.04
          nodeSelector:
            node-pool: standard
          resources:
            requests:
              memory: 512Mi
              cpu: 250m
            limits:
              memory: 2Gi
              cpu: 1000m
      cleanupPolicy:
        autoCleanup: true
        ttlSecondsAfterFinished: 300
```

### 2. Production Environment

```yaml
# prod-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: act-config-prod
  namespace: act-runners
data:
  config.yaml: |
    backend: kubernetes
    kubernetes:
      enabled: true
      namespace: act-runners
      serviceAccount: act-runner
      storageClass: act-fast-ssd
      platforms:
        ubuntu-latest:
          image: ubuntu:22.04
          nodeSelector:
            node-pool: guaranteed
          resources:
            requests:
              memory: 2Gi
              cpu: 1000m
            limits:
              memory: 8Gi
              cpu: 4000m
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
      monitoring:
        enabled: true
        prometheusMetrics: true
      cleanupPolicy:
        successfulJobsHistoryLimit: 10
        failedJobsHistoryLimit: 5
        ttlSecondsAfterFinished: 7200
```

## Deployment Scripts

### 1. Quick Setup Script

```bash
#!/bin/bash
# setup-act-k8s.sh

set -e

NAMESPACE=${NAMESPACE:-act-runners}
KUBECONFIG=${KUBECONFIG:-~/.kube/config}

echo "Setting up Act Kubernetes backend..."

# Create namespace
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# Apply RBAC
kubectl apply -f rbac.yaml

# Apply storage configuration
kubectl apply -f storage.yaml

# Apply network policies
kubectl apply -f network-policy.yaml

# Apply monitoring configuration
kubectl apply -f monitoring.yaml

echo "Setup complete! You can now use Act with Kubernetes backend:"
echo "act --backend kubernetes --namespace $NAMESPACE"
```

### 2. Cleanup Script

```bash
#!/bin/bash
# cleanup-act-k8s.sh

set -e

NAMESPACE=${NAMESPACE:-act-runners}

echo "Cleaning up Act Kubernetes resources..."

# Delete all pods in namespace
kubectl delete pods --all -n $NAMESPACE

# Delete PVCs (optional - comment out to preserve data)
# kubectl delete pvc --all -n $NAMESPACE

# Delete namespace (optional - comment out to preserve namespace)
# kubectl delete namespace $NAMESPACE

echo "Cleanup complete!"
```

## Troubleshooting

### 1. Common Issues

#### Pod Creation Failures
```bash
# Check pod events
kubectl describe pod -n act-runners <pod-name>

# Check node resources
kubectl top nodes

# Check resource quotas
kubectl describe resourcequota -n act-runners
```

#### RBAC Issues
```bash
# Test permissions
kubectl auth can-i create pods --namespace act-runners --as=system:serviceaccount:act-runners:act-runner

# Check role bindings
kubectl describe rolebinding -n act-runners
```

#### Storage Issues
```bash
# Check PVC status
kubectl get pvc -n act-runners

# Check storage class
kubectl describe storageclass act-fast-ssd
```

### 2. Debug Commands

```bash
# Enable debug logging
act --backend kubernetes --verbose --debug

# Check cluster connectivity
kubectl cluster-info

# Monitor pod creation
kubectl get pods -n act-runners -w

# View pod logs
kubectl logs -n act-runners -l app=act-runner --follow
```

## Best Practices

### 1. Resource Management
- Set appropriate resource requests and limits
- Use node affinity for workload placement
- Implement resource quotas per namespace
- Monitor resource usage regularly

### 2. Security
- Use least-privilege RBAC policies
- Enable Pod Security Standards
- Implement network policies
- Regularly update container images

### 3. Monitoring
- Set up comprehensive monitoring and alerting
- Implement log aggregation
- Monitor resource usage and costs
- Track job success/failure rates

### 4. Maintenance
- Regularly clean up completed pods
- Update Kubernetes cluster and Act CLI
- Review and update security policies
- Backup persistent data
