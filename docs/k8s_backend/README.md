# Act Kubernetes Backend

This directory contains the design documentation and implementation guide for the Kubernetes execution backend for the Act project.

## Overview

The Kubernetes backend enables Act to execute GitHub Actions workflows on Kubernetes clusters, providing scalable, cloud-native execution capabilities while maintaining full compatibility with existing Act features.

## Documentation Structure

- [Architecture Design](./architecture.md) - Overall system architecture and component design
- [Job Scheduling](./job-scheduling.md) - How GitHub Actions jobs are mapped to Kubernetes resources
- [Step Execution](./step-execution.md) - Step execution mechanisms within Kubernetes pods
- [Runs-On Handling](./runs-on-handling.md) - How runs-on labels are processed and mapped to Kubernetes scheduling
- [Configuration](./configuration.md) - Configuration options and integration with existing Act config
- [Examples](./examples.md) - Usage examples and common scenarios
- [Implementation Guide](./implementation.md) - Technical implementation details

## Quick Start

To use the Kubernetes backend:

```bash
# Enable Kubernetes backend
act --backend kubernetes

# With custom kubeconfig and namespace
act --backend kubernetes --kubeconfig ~/.kube/config --namespace act-runners
```

## Key Features

- **Seamless Integration**: Full compatibility with existing Act architecture
- **Flexible Scheduling**: Precise pod scheduling control via runs-on labels
- **Resource Management**: Support for resource limits and node affinity
- **Multi-Architecture**: Linux, Windows, ARM support
- **GPU Support**: Machine learning and compute-intensive workloads
- **Service Containers**: Complete service dependency support
- **Persistent Storage**: Tool cache and workspace persistence
- **Enterprise Features**: RBAC, network policies, monitoring integration

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Act CLI                                  │
├─────────────────────────────────────────────────────────────┤
│                 Runner Interface                            │
├─────────────────────────────────────────────────────────────┤
│  Docker Backend  │  Host Backend  │  Kubernetes Backend     │
└─────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────┐
│                Kubernetes Cluster                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │ Job Pod     │  │ Job Pod     │  │ Shared Services     │ │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────────────┐ │ │
│  │ │ Step 1  │ │  │ │ Step 1  │ │  │ │ Registry Cache  │ │ │
│  │ │ Step 2  │ │  │ │ Step 2  │ │  │ │ Artifact Store  │ │ │
│  │ │ Step 3  │ │  │ │ Step 3  │ │  │ │ Log Aggregator  │ │ │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────────────┘ │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Contributing

Please refer to the [Implementation Guide](./implementation.md) for technical details on contributing to the Kubernetes backend implementation.
