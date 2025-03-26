# SDC-Client Helm Chart

This Helm chart deploys the SDC-Client, which creates secure tunnels between systems running SDC servers and this client. The chart enables deploying multiple independent SDC-Client instances to connect to different servers.

## Background

The SDC-Client connects to SDC-Server instances. The architecture works as follows:

- **SDC-Server**: Runs on any system (Kubernetes cluster, standalone server, etc.) and generates access tokens
- **SDC-Client**: Runs on any system (Kubernetes cluster, standalone server, etc.) and connects back to servers using tokens
- Each client creates a secure tunnel, enabling communication between the systems

This chart supports deploying multiple SDC-Client instances, each connecting to a different server with its own token.

## Prerequisites

Before you begin, ensure you have:

- Kubernetes 1.16+ cluster
- Helm 3.0+ installed
- Access tokens generated from your SDC-Server instances
- Network connectivity between your client cluster and server systems

## Installing the Chart

To install the chart with the release name `sdc-client`:

```bash
# Install the chart
helm install sdc-client ./sdc-client \
  --values my-values.yaml
```

## Uninstalling the Chart

To uninstall/delete the deployment:

```bash
helm uninstall sdc-client
```

## Architecture

The chart creates separate deployments for each agent defined in your values file. Each deployment:

1. Runs its own instance of the SDC-Client container
2. Uses a dedicated ConfigMap containing the unique token for its server connection
3. Can have custom placement rules through nodeSelector, affinity, and tolerations

This approach allows you to establish multiple secure tunnels from a single Kubernetes cluster to different target servers.

## Configuration

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | SDC-Client image repository | `radiantone/sdc-client` |
| `image.tag` | Image tag (version) | Chart appVersion |
| `image.pullPolicy` | Image pull policy | `Always` |
| `replicaCount` | Number of replicas per agent | `1` |
| `agents` | List of agent configurations (see below) | `[]` |

### Agent Configuration

The `agents` section allows you to define multiple client instances, each connecting to a different server:

```yaml
agents:
  - name: "agent1"                     # Unique name for this agent
    token: "your-server-token-here"    # Token from the SDC-Server
    nodeSelector: {}                   # Optional node selection rules
    affinity: {}                       # Optional affinity rules
    tolerations: []                    # Optional tolerations
```

## Detailed Configuration

### Global Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nameOverride` | Override the name of the chart | `""` |
| `fullnameOverride` | Override the full name of the chart | `""` |
| `serviceAccount.create` | Create a service account | `false` |
| `serviceAccount.name` | Name of the service account | `""` |
| `podAnnotations` | Annotations for pods | `{}` |
| `podSecurityContext` | Security context for pods | `{}` |
| `securityContext` | Security context for containers | `{}` |
| `nodeSelector` | Default node selector for all agents | `{}` |
| `tolerations` | Default tolerations for all agents | `[]` |
| `affinity` | Default affinity rules for all agents | `{}` |
| `prometheus.enabled` | Enable Prometheus metrics | `false` |

### Agent-Specific Configuration

Each entry in the `agents` list can have the following parameters:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `name` | Unique name for this agent (required) | - |
| `token` | Token for connecting to the SDC-Server (required) | - |
| `nodeSelector` | Node selector for this specific agent | Global `nodeSelector` |
| `affinity` | Affinity rules for this specific agent | Global `affinity` |
| `tolerations` | Tolerations for this specific agent | Global `tolerations` |

## Example Configuration

```yaml
agents:
  # Production server connection
  - name: "production"
    token: "prod-token-123"

  # Staging server connection
  - name: "staging"
    token: "staging-token-456"

  # Development server connection with node selection
  - name: "development"
    token: "dev-token-789"
    nodeSelector:
      environment: development
```

## How It Works

When you deploy this chart:

1. For each entry in the `agents` list, the chart creates:
   - A dedicated Deployment with the specified name
   - A ConfigMap containing the token for that agent

2. Each SDC-Client instance:
   - Connects to its designated server using the provided token
   - Establishes a secure tunnel between the client cluster and server system
   - Operates independently from other agents

3. The overall architecture enables:
   - Multiple tunnels from a single Kubernetes cluster
   - Independent configuration for each tunnel
   - Isolated failure domains (if one tunnel fails, others continue to operate)

## Minimal values.yaml file

```yaml
# Minimal values.yaml for SDC-Client Helm chart

image:
  repository: radiantone/sdc-client
  pullPolicy: Always
  # tag: "1.0.0"  # Uncomment to override the default tag (Chart.appVersion)

# Define your SDC-Client agents here
agents:
  - name: "agent1"
    token: "your-sdc-server-token-here"

# Basic configurations - all are optional and can be removed if defaults are acceptable
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: false
  name: ""

# Pod configurations - all are optional
podAnnotations: {}
podSecurityContext: {}
securityContext: {}
nodeSelector: {}
tolerations: []
affinity: {}
```

## Maintainers

This chart is maintained by:
- pgodey (pgodey@radiantlogic.com)
