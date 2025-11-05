# sdc

![Version: 0.0.1-alpha.2](https://img.shields.io/badge/Version-0.0.1--alpha.2-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 0.2.0](https://img.shields.io/badge/AppVersion-0.2.0-informational?style=flat-square)

A Helm chart for Kubernetes

## AGENTS-INLETS

The following HELM charts deploy SDC server and INLETS-UPLINK.

INLETS_UPLINK is deployed as dependency chart.

Additionally CERT-MANAGER, POSTGRES and PGADMIN can also be deployed as dependencies as required.


1. INLETS

- Inlets creates encrypted Tunnels for data/traffic transfer between environments
- Tunnels are secured by default so anything that you transfer over them is encrypted. And they'll work through the toughest network conditions such as NAT, firewalls and corporate proxies

2. Cert-Manager

- cert-manager adds certificates and certificate issuers as resource types in Kubernetes clusters, and simplifies the process of obtaining, renewing and using those certificates.
- It will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry.

3. PostgreSQL

- PostgreSQL, also known as Postgres, is a free and open-source relational database management system emphasizing extensibility and SQL compliance
- **Chart Version:** 16.7.27 (Bitnami PostgreSQL Helm chart)
- **Image:** radiantone/postgresql:16.3.0
- **PostgreSQL Version:** 16.3.0

4. PGAdmin

- PGAdmin is a web-based GUI tool used to interact with the Postgres database sessions, both locally and remote servers as well. You can use PGAdmin to perform any sort of database administration required for a Postgres database.


### SDC Server

The SDC Server is a web server that has multiple functions:
- It provides a REST API that can be used by Cloud Manager or external API clients to register, configure, and monitor the On-Prem Agents
- It contains the business logic that orchestrates the entire SDC system (client and server) to make sure everything is in the desired state, and
that the desired configuration is applied.
- It hosts the server part of the SignalR system (described in detail later in this document) that we use to send bidirectional messages
between the SDC Client and SDC Server. Note: this system is not used for tunneling traffic and is separate from the WebSocket Tunnel
provided by Inlets.

### SDC Client

- The SDC Client is mainly a wrapper around the Inlets Clients and HAProxy processes (described in detail later in this document).
- The SDC client will orchestrate these processes and re-configure them based on the commands that it receives from the SDC Server.
- The SDC client process itself will be deployed as a containerized application that will be launched in the on-prem environment.
(TBD, Design Decision) The SDC Client may run in a Kubernetes cluster for high availability and failover

### DEPLOYING SDC (without any dependencies)

```
helm -n <namespace> install <release_name> <chart> --set inlets.enabled=false --set cert-manager.enabled=false --set postgresql.enabled=false --set pgadmin4.enabled=false --debug
```

#### Accessing SDC server

```
kubectl port-forward svc/<release_name> -n <namespace> 80

```

The SDC swagger page can be reached at http://localhost:80/swagger


### DEPLOYING SDC (with dependencies)

```
helm -n <namespace> install <release_name> <chart> --set inlets.enabled=true --set cert-manager.enabled=true --set postgresql.enabled=true --set pgadmin4.enabled=true --debug
```

### Sample Values file

```

nodeSelector: {}
  # tenantname: xxxxx

agents:
  database:
    host: postgresql
    name: agentsdb
    schema: agents
    auth:
      password: xxxxx
      username: agentsadmin
  clientId: "xxxxx"
  clientSecret: "xxxxx"
  # PortForward__Range should match the start and end ports provided under tunnel
  # PortForward__Range can be a single port if only one port is opened
  portForward__range: "5001-5010, 8080, 8081"

inlets:
  nodeSelector: {}
    # tenantname: xxxxxxx
  license: "xxxxx"
  # A router to work on a wildcard domain and to direct
  # traffic according to request domain
  clientRouter:
    # Customer tunnels will connect with a URI of:
    # wss://uplink.example.com/namespace/tunnel
    domain: uplink.example.com/namespace/tunnel

tunnel:
  tunnelname: r1tunnel
  nodeSelector: {}
    # tenantname: xxxxxxx
  # Ports opened for tunnels
  # To open single ports of choice provide values under "ports:"
  # To open range of ports provide values under "portRange" like "5001-5010"
  # Provide higher + 1 for the higher limit (port) intended (for portRange)
  portRange:
    - "5001-5010"

# Postgresql should be deployed prior to deployment or should be enabled from below
postgresql:
  primary:
    nodeSelector: {}
  enabled: false


hooks:
  post_upgrade:
    enabled: true
  hooks_sa:
    enabled: true

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| pgodey | <pgodey@radiantlogic.com> | <https://www.radiantlogic.com> |

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| https://charts.bitnami.com/bitnami | postgresql | 16.7.27 |
| https://charts.jetstack.io | cert-manager | 1.11.0 |
| https://helm.runix.net | pgadmin4 | 1.13.8 |
| oci://ghcr.io/openfaasltd | inlets(inlets-uplink-provider) | 0.2.9 |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` |  |
| agents.clientId | string | `"xxxxx"` |  |
| agents.clientSecret | string | `"xxxxx"` |  |
| agents.database.auth.password | string | `"xxxxx"` |  |
| agents.database.auth.username | string | `"agentsadmin"` |  |
| agents.database.host | string | `"postgresql"` |  |
| agents.database.name | string | `"agentsdb"` |  |
| agents.database.schema | string | `"agents"` |  |
| agents.portForward__range | string | `"5001-5009, 8080, 8081"` |  |
| autoscaling.enabled | bool | `false` |  |
| autoscaling.maxReplicas | int | `100` |  |
| autoscaling.minReplicas | int | `1` |  |
| autoscaling.targetCPUUtilizationPercentage | int | `80` |  |
| cert-manager.cainjector.nodeSelector | object | `{}` |  |
| cert-manager.clusterResourceNamespace | string | `nil` |  |
| cert-manager.enabled | bool | `false` |  |
| cert-manager.fullnameOverride | string | `"cert-manager"` |  |
| cert-manager.installCRDs | bool | `true` |  |
| cert-manager.namespace | string | `nil` |  |
| cert-manager.nodeSelector | object | `{}` |  |
| cert-manager.startupapicheck.nodeSelector | object | `{}` |  |
| cert-manager.webhook.nodeSelector | object | `{}` |  |
| fullnameOverride | string | `""` |  |
| hooks.hooks_sa.enabled | bool | `true` |  |
| hooks.post_upgrade.enabled | bool | `true` |  |
| image.pullPolicy | string | `"Always"` |  |
| image.repository | string | `"radiantone/sdc-server"` |  |
| imageCredentials.email | string | `"someone@host.com"` |  |
| imageCredentials.password | string | `"something"` |  |
| imageCredentials.registry | string | `"https://index.docker.io/v1/"` |  |
| imageCredentials.username | string | `"someone"` |  |
| imagePullSecrets[0].name | string | `"regcred"` |  |
| ingress.annotations | object | `{}` |  |
| ingress.className | string | `""` |  |
| ingress.enabled | bool | `false` |  |
| ingress.hosts[0].host | string | `"chart-example.local"` |  |
| ingress.hosts[0].paths[0].path | string | `"/"` |  |
| ingress.hosts[0].paths[0].pathType | string | `"ImplementationSpecific"` |  |
| ingress.tls | list | `[]` |  |
| inlets.affinity | object | `{}` |  |
| inlets.clientApi.enabled | bool | `false` |  |
| inlets.clientApi.image | string | `"ghcr.io/openfaasltd/uplink-api:0.1.7"` |  |
| inlets.clientRouter.domain | string | `"uplink.example.com"` |  |
| inlets.clientRouter.service.type | string | `"NodePort"` |  |
| inlets.clientRouter.tls.ingress.annotations."nginx.ingress.kubernetes.io/keepalive-timeout" | string | `"350"` |  |
| inlets.clientRouter.tls.ingress.annotations."nginx.ingress.kubernetes.io/limit-connections" | string | `"300"` |  |
| inlets.clientRouter.tls.ingress.annotations."nginx.ingress.kubernetes.io/limit-rpm" | string | `"1000"` |  |
| inlets.clientRouter.tls.ingress.annotations."nginx.ingress.kubernetes.io/proxy-buffer-size" | string | `"128k"` |  |
| inlets.clientRouter.tls.ingress.annotations."nginx.ingress.kubernetes.io/proxy-read-timeout" | string | `"3600"` |  |
| inlets.clientRouter.tls.ingress.annotations."nginx.ingress.kubernetes.io/proxy-send-timeout" | string | `"3600"` |  |
| inlets.clientRouter.tls.ingress.class | string | `"nginx"` |  |
| inlets.clientRouter.tls.ingress.enabled | bool | `false` |  |
| inlets.clientRouter.tls.issuer.email | string | `"user@example.com"` |  |
| inlets.clientRouter.tls.issuer.enabled | bool | `false` |  |
| inlets.clientRouter.tls.issuerName | string | `"letsencrypt-prod"` |  |
| inlets.clientRouter.tls.istio.enabled | bool | `false` |  |
| inlets.enabled | bool | `true` |  |
| inlets.fullnameOverride | string | `""` |  |
| inlets.inletsVersion | string | `"0.9.18"` |  |
| inlets.license | string | `"xxxxxx"` |  |
| inlets.nameOverride | string | `""` |  |
| inlets.nodeSelector | object | `{}` |  |
| inlets.prometheus.annotations | object | `{}` |  |
| inlets.prometheus.create | bool | `false` |  |
| inlets.prometheus.image | string | `"prom/prometheus:v2.40.1"` |  |
| inlets.prometheus.resources.requests.memory | string | `"512Mi"` |  |
| inlets.prometheus.service.type | string | `"NodePort"` |  |
| inlets.resources.requests.cpu | string | `"100m"` |  |
| inlets.resources.requests.memory | string | `"128Mi"` |  |
| inlets.tolerations | list | `[]` |  |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| pgadmin4.enabled | bool | `false` |  |
| pgadmin4.env.contextPath | string | `"/pgadmin4"` |  |
| pgadmin4.fullnameOverride | string | `"pgadmin4"` |  |
| pgadmin4.nodeSelector | object | `{}` |  |
| pgadmin4.persistentVolume.enabled | bool | `false` |  |
| podAnnotations | object | `{}` |  |
| podSecurityContext | object | `{}` |  |
| postgresql.enabled | bool | `false` |  |
| postgresql.fullnameOverride | string | `"postgresql"` |  |
| postgresql.primary.initdb.scripts."01_agents_init_script.sql" | string | `"CREATE DATABASE agentsdb;\nCREATE USER agentsadmin WITH ENCRYPTED PASSWORD 'xxxxx';\nGRANT ALL PRIVILEGES ON DATABASE agentsdb TO agentsadmin;\n"` |  |
| postgresql.primary.initdb.scripts."XX_create_schema_init_script.sh" | string | `"#!/bin/bash\nPGPASSWORD=$POSTGRES_PASSWORD psql -v ON_ERROR_STOP=1 <<-EOSQL\n  \\connect agentsdb;\n  CREATE SCHEMA IF NOT EXISTS agents;\n  GRANT ALL ON SCHEMA agents TO agentsadmin;\nEOSQL\n"` |  |
| postgresql.primary.nodeSelector | object | `{}` |  |
| prometheus.enabled | bool | `false` |  |
| replicaCount | int | `1` |  |
| resources | object | `{}` |  |
| securityContext | object | `{}` |  |
| service.port | int | `80` |  |
| service.type | string | `"NodePort"` |  |
| serviceAccount.annotations | object | `{}` |  |
| serviceAccount.create | bool | `false` |  |
| serviceAccount.name | string | `""` |  |
| tolerations | list | `[]` |  |
| tunnel.nodeSelector | object | `{}` |  |
| tunnel.portRange[0] | string | `"5001-5010"` |  |
| tunnel.ports[0] | int | `8080` |  |
| tunnel.ports[1] | int | `8081` |  |
| tunnel.pregentokenname | string | `""` |  |
| tunnel.token | string | `""` |  |
| tunnel.tunnelname | string | `"xxxxx"` |  |


