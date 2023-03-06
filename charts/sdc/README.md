![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.16.0](https://img.shields.io/badge/AppVersion-1.16.0-informational?style=flat-square)

## AGENTS-INLETS

The following HELM charts deploy AGENTS-API server and INLETS-UPLINK.

INLETS_UPLINK is deployed as dependency chart.

Additionally CERT-MANAGER, POSTGRES and PGADMIN can also be deployed as dependencies as required.

## PRE-REQUISITES

- Kubernetes Cluster

- Helm 3


## Dependency Charts

| Repository | Name | Version |
|------------|------|---------|
| https://charts.bitnami.com/bitnami | postgresql | 12.1.3 |
| https://charts.jetstack.io | cert-manager | 1.11.0 |
| https://helm.runix.net | pgadmin4 | 1.13.8 |
| oci://ghcr.io/openfaasltd | inlets(inlets-uplink-provider) | 0.2.2 |

1. INLETS

- Inlets creates encrypted Tunnels for data/traffic transfer between environments
- Tunnels are secured by default so anything that you transfer over them is encrypted. And they'll work through the toughest network conditions such as NAT, firewalls and corporate proxies

2. Cert-Manager

- cert-manager adds certificates and certificate issuers as resource types in Kubernetes clusters, and simplifies the process of obtaining, renewing and using those certificates.
- It will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry.

3. Posgresql

- PostgreSQL, also known as Postgres, is a free and open-source relational database management system emphasizing extensibility and SQL compliance

4. PGAdmin

- PGAdmin is a web-based GUI tool used to interact with the Postgres database sessions, both locally and remote servers as well. You can use PGAdmin to perform any sort of database administration required for a Postgres database.


### R1-AGENT Server

The R1-AGENT Server is a web server that has multiple functions:
- It provides a REST API that can be used by Cloud Manager or external API clients to register, configure, and monitor the On-Prem Agents
- It contains the business logic that orchestrates the entire R1-AGENT system (client and server) to make sure everything is in the desired state, and
that the desired configuration is applied.
- It hosts the server part of the SignalR system (described in detail later in this document) that we use to send bidirectional messages
between the R1-AGENT Client and R1-AGENT Server. Note: this system is not used for tunneling traffic and is separate from the WebSocket Tunnel
provided by Inlets.

### R1-AGENT Client

- The R1-AGENT Client is mainly a wrapper around the Inlets Clients and HAProxy processes (described in detail later in this document).
- The R1-AGENT client will orchestrate these processes and re-configure them based on the commands that it receives from the R1-AGENT Server.
- The R1-AGENT client process itself will be deployed as a containerized application that will be launched in the on-prem environment.
(TBD, Design Decision) The R1-AGENT Client may run in a Kubernetes cluster for high availability and failover

### DEPLOYING R1-AGENT (without any dependencies)

```
helm -n <namespace> install <release_name> <chart> --set inlets.enabled=false --set cert-manager.enabled=false --set postgresql.enabled=false --set pgadmin4.enabled=false --debug
```

#### Accessing R1-AGENT server

```
kubectl port-forward svc/<release_name> -n <namespace> 80

```

The R1-AGENT swagger page can be reached at http://localhost:80/swagger


### DEPLOYING R1-AGENT (with dependencies)

```
helm -n <namespace> install <release_name> <chart> --set inlets.enabled=true --set cert-manager.enabled=true --set postgresql.enabled=true --set pgadmin4.enabled=true --debug
```

### Sample Values file

```

image:
  repository: registry.gitlab.com/radiant-logic-engineering/radiant-one-opa-server/sdc-server
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "latest"

imagePullSecrets:
  - name: regcred
nameOverride: ""
fullnameOverride: ""

prometheus:
  enabled: false

service:
  type: NodePort
  port: 80

nodeSelector: {}
  # tenantname:

agents:
  database:
    ConnectionStrings__AgentsDatabase: "Host=postgresql; Database=agentsdb; Username=agentsadmin; Password=xxxx; SearchPath=agents"
    DefaultApiClient__ClientId: "xxxxxx"
    DefaultApiClient__ClientSecret: "xxxxxx"
    PortForward__Range: "5001-5009"
  jwtIssuer: "OPAServer"
  dockerconfigjson: "xxxxxx"

tunnel:
  tunnelname: acmeco
  nodeSelector: {}
  token: "xxxxxx"
  pregentokenname: "xxxx"
  tcpPorts:
    start: 5001
    end: 5010

inlets:
  token: "xxxxxx"
  license: "xxxxxxx"
  enabled: true
  # A router to work on a wildcard domain and to direct
  # traffic according to request domain
  clientRouter:
    # Customer tunnels will connect with a URI of:
    # wss://uplink.example.com/namespace/tunnel
    domain: example.com

    service:
      type: NodePort

    tls:
      issuerName: "letsencrypt-prod"

      issuer:
        enabled: false
        # Email address used for ACME registration
        email: "user@example.com"

  # A separate REST API for automation of inlets tunnels
  clientApi:
    enabled: false
    image: "ghcr.io/openfaasltd/uplink-api:0.1.7"

  prometheus:
    image: prom/prometheus:v2.40.1
    create: false
  nameOverride: ""
  fullnameOverride: ""
  nodeSelector: {}

cert-manager:
  enabled: false
  fullnameOverride: cert-manager
  installCRDs: true
  clusterResourceNamespace:
  namespace:
  nodeSelector: {}
  webhook:
    nodeSelector: {}
  cainjector:
    nodeSelector: {}
  startupapicheck:
    nodeSelector: {}

postgresql:
  enabled: false
  fullnameOverride: postgresql
  primary:
    nodeSelector: {}
      # tenantname: duploservices-ensemble-svc
    initdb:
      scripts:
        01_agents_init_script.sql: |
          CREATE DATABASE agentsdb;
          CREATE USER agentsadmin WITH ENCRYPTED PASSWORD 'xxxxxx';
          GRANT ALL PRIVILEGES ON DATABASE agentsdb TO agentsadmin;
        02_eoc_init_script.sql: |
          CREATE DATABASE eocdb;
          CREATE USER eocadmin WITH ENCRYPTED PASSWORD 'xxxxxx';
          GRANT ALL PRIVILEGES ON DATABASE eocdb TO eocadmin;
        XX_create_schema_init_script.sh: |
          #!/bin/bash
          PGPASSWORD=$POSTGRES_PASSWORD psql -v ON_ERROR_STOP=1 <<-EOSQL
            \connect eocdb;
            CREATE SCHEMA IF NOT EXISTS eoc;
            GRANT ALL ON SCHEMA eoc TO eocadmin;
            GRANT ALL ON SCHEMA public TO eocadmin;
            \connect agentsdb;
            CREATE SCHEMA IF NOT EXISTS agents;
            GRANT ALL ON SCHEMA agents TO agentsadmin;
          EOSQL

pgadmin4:
  enabled: false
  fullnameOverride: pgadmin4
  nodeSelector: {}
  persistentVolume:
    enabled: false
  env:
    contextPath: "/pgadmin4"

```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` |  |
| agents.database.ConnectionStrings__AgentsDatabase | string | `"Host=postgresql; Database=agentsdb; Username=agentsadmin; Password=9iKnkK4Xzi; SearchPath=agents"` |  |
| agents.database.DefaultApiClient__ClientId | string | `"xxxxxx"` |  |
| agents.database.DefaultApiClient__ClientSecret | string | `"xxxxxx"` |  |
| agents.database.PortForward__Range | string | `"xxxxxx"` | Ports that have been selcted to be opened - start-end |
| agents.dockerconfigjson | string | `"xxxxxxx"` |  |
| agents.imagePullSecretName | string | `"gitlab-pull-secret"` |  |
| agents.jwtIssuer | string | `"OPAServer"` |  |
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
| image.pullPolicy | string | `"Always"` |  |
| image.repository | string | `"registry.gitlab.com/radiant-logic-engineering/radiant-one-opa-server/sdc-server"` |  |
| image.tag | string | `"latest"` |  |
| imagePullSecrets | list | `regcred` |  |
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
| inlets.clientRouter.service.type | string | `NodePort` |  |
| inlets.enabled | bool | `true` |  |
| inlets.fullnameOverride | string | `""` |  |
| inlets.inletsVersion | string | `"0.9.12"` |  |
| inlets.license | string | `"xxxxxxx"` |  |
| inlets.nameOverride | string | `""` |  |
| inlets.nodeSelector | object | `{}` |  |
| inlets.prometheus.annotations | object | `{}` |  |
| inlets.prometheus.create | bool | `false` |  |
| inlets.prometheus.image | string | `"prom/prometheus:v2.40.1"` |  |
| inlets.prometheus.resources.requests.memory | string | `"512Mi"` |  |
| inlets.resources.requests.cpu | string | `"100m"` |  |
| inlets.resources.requests.memory | string | `"128Mi"` |  |
| inlets.tolerations | list | `[]` |  |
| nameOverride | string | `""` |  |
| nodeSelector | object | `{}` |  |
| prometheus.enabled | bool | `false` |  |
| pgadmin4.enabled | bool | `false` |  |
| pgadmin4.env.contextPath | string | `"/pgadmin4"` |  |
| pgadmin4.fullnameOverride | string | `"pgadmin4"` |  |
| pgadmin4.nodeSelector | object | `{}` |  |
| pgadmin4.persistentVolume.enabled | bool | `false` |  |
| podAnnotations | object | `{}` |  |
| podSecurityContext | object | `{}` |  |
| postgresql.enabled | bool | `true` |  |
| postgresql.fullnameOverride | string | `"postgresql"` |  |
| postgresql.primary.initdb.scripts."01_agents_init_script.sql" | string | `"CREATE DATABASE agentsdb;\nCREATE USER agentsadmin WITH ENCRYPTED PASSWORD '9iKnkK4Xzi';\nGRANT ALL PRIVILEGES ON DATABASE agentsdb TO agentsadmin;\n"` |  |
| postgresql.primary.initdb.scripts."02_eoc_init_script.sql" | string | `"CREATE DATABASE eocdb;\nCREATE USER eocadmin WITH ENCRYPTED PASSWORD '9iKnkK4Xzi';\nGRANT ALL PRIVILEGES ON DATABASE eocdb TO eocadmin;\n"` |  |
| postgresql.primary.initdb.scripts."XX_create_schema_init_script.sh" | string | `"#!/bin/bash\nPGPASSWORD=$POSTGRES_PASSWORD psql -v ON_ERROR_STOP=1 <<-EOSQL\n  \\connect eocdb;\n  CREATE SCHEMA IF NOT EXISTS eoc;\n  GRANT ALL ON SCHEMA eoc TO eocadmin;\n  GRANT ALL ON SCHEMA public TO eocadmin;\n  \\connect agentsdb;\n  CREATE SCHEMA IF NOT EXISTS agents;\n  GRANT ALL ON SCHEMA agents TO agentsadmin;\nEOSQL\n"` |  |
| postgresql.primary.nodeSelector | object | `{}` |  |
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
| tunnel.tcpPorts.start | int | `5001` |  |
| tunnel.tcpPorts.end | int | `5010` |  |
| tunnel.pregentokenname | string | `"pregentoken"` |  |
| tunnel.token | string | `"xxxxxxx"` |  |
| tunnel.tunnelname | string | `"acmeco"` |  |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.11.0](https://github.com/norwoodj/helm-docs/releases/v1.11.0)
