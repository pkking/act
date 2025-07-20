# Kubernetes Backend Architecture

## Overview

The Kubernetes backend for Act provides a cloud-native execution environment for GitHub Actions workflows. It integrates seamlessly with the existing Act architecture while leveraging Kubernetes' scheduling, scaling, and resource management capabilities.

## Core Components

### 1. Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    Act CLI Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Workflow    │  │ Job         │  │ Step                │ │
│  │ Parser      │  │ Scheduler   │  │ Executor            │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────┐
│                Runner Interface Layer                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Docker      │  │ Host        │  │ Kubernetes          │ │
│  │ Backend     │  │ Backend     │  │ Backend             │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Execution Layer                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Pod         │  │ Service     │  │ Resource            │ │
│  │ Manager     │  │ Manager     │  │ Manager             │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────┐
│                Kubernetes Cluster                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Worker      │  │ Worker      │  │ Control Plane       │ │
│  │ Node 1      │  │ Node 2      │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 2. Key Interfaces

#### ExecutionsEnvironment Interface

The Kubernetes backend implements the existing `ExecutionsEnvironment` interface:

```go
type ExecutionsEnvironment interface {
    Container
    ToContainerPath(string) string
    GetActPath() string
    GetPathVariableName() string
    DefaultPathVariable() string
    JoinPathVariable(...string) string
    GetRunnerContext(ctx context.Context) map[string]interface{}
    IsEnvironmentCaseInsensitive() bool
}
```

#### KubernetesExecutionEnvironment Implementation

```go
type KubernetesExecutionEnvironment struct {
    client       kubernetes.Interface
    config       *KubernetesConfig
    podName      string
    namespace    string
    logger       *logrus.Entry
    podSpec      *v1.Pod
    runContext   *RunContext
}
```

### 3. Component Responsibilities

#### Pod Manager
- Creates and manages Kubernetes pods for job execution
- Handles pod lifecycle (creation, monitoring, cleanup)
- Manages pod specifications based on runs-on labels
- Implements resource allocation and scheduling constraints

#### Service Manager
- Manages service containers defined in workflows
- Creates Kubernetes services for inter-container communication
- Handles service discovery and networking

#### Resource Manager
- Manages persistent volumes for workspace and caches
- Handles ConfigMaps and Secrets for environment variables
- Manages resource quotas and limits

## Design Principles

### 1. Compatibility
- Full backward compatibility with existing Act features
- Seamless integration with current CLI and configuration
- Support for all existing workflow syntax

### 2. Scalability
- Horizontal scaling through Kubernetes cluster resources
- Efficient resource utilization with pod-based execution
- Support for concurrent job execution

### 3. Flexibility
- Support for multiple architectures (x86, ARM, etc.)
- Custom node selection through runs-on labels
- Configurable resource requirements

### 4. Reliability
- Fault tolerance through Kubernetes' self-healing capabilities
- Proper error handling and retry mechanisms
- Comprehensive logging and monitoring

## Integration Points

### 1. Configuration System
- Extends existing Act configuration with Kubernetes-specific options
- Maintains compatibility with `.actrc` files
- Supports environment-specific configurations

### 2. Platform Mapping
- Extends the existing platform mapping system
- Maps runs-on labels to Kubernetes node selectors
- Supports custom platform definitions

### 3. Logging and Monitoring
- Integrates with existing Act logging infrastructure
- Provides Kubernetes-native monitoring capabilities
- Supports log aggregation and streaming

## Security Considerations

### 1. RBAC Integration
- Supports Kubernetes Role-Based Access Control
- Configurable service accounts for different execution contexts
- Namespace-based isolation

### 2. Network Security
- Support for Kubernetes network policies
- Configurable pod security contexts
- Secure communication between components

### 3. Secret Management
- Integration with Kubernetes secrets
- Support for external secret management systems
- Secure handling of sensitive workflow data

## Performance Characteristics

### 1. Startup Time
- Pod creation and scheduling overhead
- Image pull optimization strategies
- Warm pool capabilities for faster startup

### 2. Resource Efficiency
- Efficient resource allocation and cleanup
- Support for resource limits and requests
- Optimization for different workload types

### 3. Scalability Limits
- Cluster resource constraints
- Kubernetes API rate limits
- Network and storage performance considerations
