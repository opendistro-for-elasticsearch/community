# Opendistro Elasticsearch
This chart installs [Opendistro Kibana](https://opendistro.github.io/for-elasticsearch-docs/docs/kibana/) + [Opendistro Elasticsearch](https://opendistro.github.io/for-elasticsearch-docs/docs/elasticsearch/) with configurable TLS, RBAC, and more.
Due to the uniqueness of different users environments, this chart aims to cater to a number of different use cases and setups.

## TL;DR
```
❯ helm package .
❯ helm install opendistro-es-0.0.1.tgz --name opendistro-es
```

## Installing the Chart
To install the chart with the release name `my-release`:

`❯ helm install --name my-release opendistro-es-0.0.1.tgz`

The command deploys OpenDistro Kibana and Elasticsearch with its associated components (data statefulsets, masters, clients) on the Kubernetes cluster in the default configuration.

## Uninstalling the Chart
To delete/uninstall the chart with the release name `my-release`:
```
❯ helm delete --name opendistro-es
```

### Notes About Default Installation
By default, on startup, opendistro will update the [default](https://github.com/opendistro-for-elasticsearch/opendistro-build/blob/master/elasticsearch/docker/build/elasticsearch/elasticsearch.yml) `elasticsearch.yml` via the [install_demo_configuration.sh](https://github.com/opendistro-for-elasticsearch/security/blob/dfc41db0d0123cd0965d40ee47d61266e560f7e6/tools/install_demo_configuration.sh) to mirror the below and generate some [default certs](https://github.com/opendistro-for-elasticsearch/security/blob/dfc41db0d0123cd0965d40ee47d61266e560f7e6/tools/install_demo_configuration.sh#L201):
```
cluster.name: "docker-cluster"
network.host: 0.0.0.0

# minimum_master_nodes need to be explicitly set when bound on a public IP
# set to 1 to allow single node clusters
# Details: https://github.com/elastic/elasticsearch/pull/17288
discovery.zen.minimum_master_nodes: 1

######## Start OpenDistro for Elasticsearch Security Demo Configuration ########
# WARNING: revise all the lines below before you go into production
opendistro_security.ssl.transport.pemcert_filepath: esnode.pem
opendistro_security.ssl.transport.pemkey_filepath: esnode-key.pem
opendistro_security.ssl.transport.pemtrustedcas_filepath: root-ca.pem
opendistro_security.ssl.transport.enforce_hostname_verification: false
opendistro_security.ssl.http.enabled: true
opendistro_security.ssl.http.pemcert_filepath: esnode.pem
opendistro_security.ssl.http.pemkey_filepath: esnode-key.pem
opendistro_security.ssl.http.pemtrustedcas_filepath: root-ca.pem
opendistro_security.allow_unsafe_democertificates: true
opendistro_security.allow_default_init_securityindex: true
opendistro_security.authcz.admin_dn:
  - CN=kirk,OU=client,O=client,L=test, C=de

opendistro_security.audit.type: internal_elasticsearch
opendistro_security.enable_snapshot_restore_privilege: true
opendistro_security.check_snapshot_restore_write_privileges: true
opendistro_security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
cluster.routing.allocation.disk.threshold_enabled: false
node.max_local_storage_nodes: 3
######## End OpenDistro for Elasticsearch Security Demo Configuration ########
```

This will only be done if `opendistro_security` is [not found](https://github.com/opendistro-for-elasticsearch/security/blob/dfc41db0d0123cd0965d40ee47d61266e560f7e6/tools/install_demo_configuration.sh#L194) in the `elasticsearch.yml` file. With this in mind, it's important to note that as a user, providing a complete configuration to both `kibana.config` and `elasticsearch.config` is required when working with custom certs, secrets, etc.

## Certs, Secrets, and Configuration
Prior to installation there are a number of secrets that can be defined that will get mounted into the
different elasticsearch/kibana components.

### Kibana

#### Kibana.yml Config
All values defined under `kibana.config` will be converted to yaml and mounted into the config directory.

#### Example Secure Kibana Config With Custom Certs
The below config requires the following secrets to be defined:

* `elasticsearch-account` - Elasticsearch account with a username, password, and cookie secret field to be expanded into`${ELASTICSEARCH_USERNAME}`, `${ELASTICSEARCH_PASSWORD}`, and `${COOKIE_PASS}` in `kibana.yml`
* `kibana-certs` - Kibana certs matching the form under [ssl](#ssl)
* `elasticsearch-rest-certs` - Elasticsearch rest certs matching the form under [ssl](#ssl)

With the above secrets, and the below config, deployment with custom signed certs is possible:
```
kibana:

  elasticsearchAccount:
    secret: elasticsearch-account

  ssl:
    kibana:
      enabled: true
      existingCertSecret: kibana-certs
    elasticsearch:
      enabled: true
      existingCertSecret: elasticsearch-rest-certs

  config:
    # Default Kibana configuration from kibana-docker.
    server.name: kibana
    server.host: "0"

    elasticsearch.hosts: https://elasticsearch.example.com:443
    elasticsearch.requestTimeout: 360000

    logging.verbose: true

    # Kibana TLS Config
    server.ssl.enabled: true
    server.ssl.key: /usr/share/kibana/certs/kibana-key.pem
    server.ssl.certificate: /usr/share/kibana/certs/kibana-crt.pem

    opendistro_security.cookie.secure: true
    opendistro_security.cookie.password: ${COOKIE_PASS}

    elasticsearch.username: ${ELASTICSEARCH_USERNAME}
    elasticsearch.password: ${ELASTICSEARCH_PASSWORD}

    opendistro_security.allow_client_certificates: true
    elasticsearch.ssl.certificate: /usr/share/kibana/certs/elk-rest-crt.pem
    elasticsearch.ssl.key: /usr/share/kibana/certs/elk-rest-key.pem
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/certs/elk-rest-root-ca.pem"]

    # Multitenancy with global/private tenants disabled,
    # set to both to true if you want them to be available.
    opendistro_security.multitenancy.enabled: true
    opendistro_security.multitenancy.tenants.enable_private: false
    opendistro_security.multitenancy.tenants.enable_global: false
    opendistro_security.readonly_mode.roles: ["kibana_read_only"]
    elasticsearch.requestHeadersWhitelist: ["securitytenant","Authorization"]
    opendistro_security.allow_client_certificates: true
```

#### Elasticsearch Specific Secrets
Elasticsearch specific values passed in through the environment including `ELASTICSEARCH_USERNAME`, `ELASTICSEARCH_PASSWORD`,
`COOKIE_PASS`, and optionally a keypass phrase under `KEY_PASSPHRASE`.
```
elasticsearchAccount:
  secret: ""
  keyPassphrase:
    enabled: false
```

#### SSL
Optionally you can define ssl secrets for kibana as well as secrets for interactions between kibana and elasticsearch's rest clients:
```
ssl:
  kibana:
    enabled: true
    existingCertSecret: kibana-certs
  elasticsearch:
    enabled: true
    existingCertSecret: elasticsearch-rest-certs
```
The chart expects the `kibana.existingCertSecret` to have the following values:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: kibana-certs
  namespace: desired_namespace
  labels:
    app: elasticsearch
data:
  kibana-crt.pem: base64value
  kibana-key.pem: base64value
  kibana-root-ca.pem: base64value
```
Similarly for the elasticsearch rest certs:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-rest-certs
  namespace: desired_namespace
  labels:
    app: elasticsearch
data:
  elk-rest-crt.pem: base64value
  elk-rest-key.pem: base64value
  elk-rest-root-ca.pem: base64value

```

### Elasticsearch

#### SSL
For Elasticsearch you can optionally define ssl secrets for elasticsearch transport, rest, and admin certs:

***NOTE:*** The manifests require the keys for the certs (e.g. `elk-transport-crt.pem`) to match up in order
to properly mount them to their corresponding `subPath`.
```
ssl:
  transport:
    enabled: true
    existingCertSecret: elasticsearch-transport-certs
  rest:
    enabled: true
    existingCertSecret: elasticsearch-rest-certs
  admin:
    enabled: true
    existingCertSecret: elasticsearch-admin-certs

```
The transport certs are expected to be formatted in the following way:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-transport-certs
  namespace: desired_namespace
  labels:
    app: elasticsearch
data:
  elk-transport-crt.pem:
  elk-transport-key.pem:
  elk-transport-root-ca.pem:
```
The admin certs are expected to be formatted in the following way:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: elasticsearch-admin-certs
  namespace: desired_namespace
  labels:
    app: elasticsearch
data:
  admin-crt.pem:
  admin-key.pem:
  admin-root-ca.pem:
```

#### Security Configuration
In an effort to avoid having to `exec` into the pod to configure the various security options see
[here](https://github.com/opendistro-for-elasticsearch/security/tree/master/securityconfig),
you can define secrets that map to each config given that each secret contains the matching file name:
```
securityConfig:
  enabled: true
  path: "/usr/share/elasticsearch/plugins/opendistro_security/securityconfig"
  actionGroupsSecret:
  configSecret:
  internalUsersSecret:
  rolesSecret:
  rolesMappingSecret:
```
Example:
```
❯ cat config.yml
opendistro_security:
  dynamic:
    kibana:
      multitenancy_enabled: true
      server_username: atgdev_opendistro-es-svc
      index: '.kibana'
      do_not_fail_on_forbidden: false
    authc:
      ldap_service_accounts:
        enabled: true
        order: 1
        http_authenticator:
.......
❯ kubectl create secret generic -n logging security-config --from-file=config.yml
```
By coupling the above secrets with `opendistro_security.allow_default_init_securityindex: true` in your
`elasticsearch.config:` at startup all of the secrets will be mounted in and read.

Alternatively you can set `securityConfig.enabled` to `false` and `exec` into the container and make changes as you see fit using the instructions
[here](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/elasticsearch.yml.example)

#### elasticsearch.yml Config
All values defined under `elasticsearch.config` will be converted to yaml and mounted into the config directory.
See example [here](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/elasticsearch.yml.example)

#### Example Secure Elasticsearch Config With Custom Certs
The below config enables TLS with custom certs, sets up the admin certs, and configures
the [securityconfigs](https://github.com/opendistro-for-elasticsearch/security/tree/master/securityconfig) with custom entries.

The following secrets are required:
* `elasticsearch-rest-certs` - Elasticsearch rest certs matching the form under [ssl](#ssl)
* `elasticsearch-transport-certs` - Elasticsearch transport certs matching the form under [ssl](#ssl)
* `elasticsearch-admin-certs` - Elasticsearch admin certs matching the form under [ssl](#ssl)
* `configSecret` - [config.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/config.yml)
* `rolesSecret` - [roles.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/roles.yml)
* `rolesMappingSecret` - [roles_mapping.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/roles_mapping.yml)
* `internalUsersSecret` - [internal_users.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/internal_users.yml)
```
elasticsearch:

  ssl:
    transport:
      enabled: true
      existingCertSecret: elasticsearch-transport-certs
    rest:
      enabled: true
      existingCertSecret: elasticsearch-rest-certs
    admin:
      enabled: true
      existingCertSecret: elasticsearch-admin-certs

  securityConfig:
    enabled: true
    path: "/usr/share/elasticsearch/plugins/opendistro_security/securityconfig"
    configSecret: "security-config"
    rolesSecret: "roles-config"
    rolesMappingSecret: "roles-mapping-config"
    internalUsersSecret: "internal-users-config"


  config:
    # Majority of options described here: https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/elasticsearch.yml.example
    opendistro_security.audit.ignore_users: ["kibanaserver"]
    opendistro_security.allow_unsafe_democertificates: false
    # Set to false if running securityadmin.sh manually following deployment
    opendistro_security.allow_default_init_securityindex: true
    # See: https://opendistro.github.io/for-elasticsearch-docs/docs/security-audit-logs/
    opendistro_security.audit.type: internal_elasticsearch
    # See: https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/elasticsearch.yml.example#L27
    opendistro_security.roles_mapping_resolution: BOTH
    opendistro_security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
    cluster:
      name: ${CLUSTER_NAME}
    node:
      master: ${NODE_MASTER}
      data: ${NODE_DATA}
      name: ${NODE_NAME}
      ingest: ${NODE_INGEST}
      max_local_storage_nodes: 1
      attr.box_type: hot

    # See: https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/elasticsearch.yml.example#L17
    opendistro_security.nodes_dn:
      - 'CN=nodes.example.com'

    processors: ${PROCESSORS:1}

    network.host: ${NETWORK_HOST}

    thread_pool.bulk.queue_size: 800

    http:
      enabled: ${HTTP_ENABLE}
      compression: true

    discovery:
      zen:
        minimum_master_nodes: ${NUMBER_OF_MASTERS}

    # TLS Configuration Transport Layer
    opendistro_security.ssl.transport.pemcert_filepath: elk-transport-crt.pem
    opendistro_security.ssl.transport.pemkey_filepath: elk-transport-key.pem
    opendistro_security.ssl.transport.pemtrustedcas_filepath: elk-transport-root-ca.pem

    # TLS Configuration REST Layer
    opendistro_security.ssl.http.enabled: true
    opendistro_security.ssl.http.pemcert_filepath: elk-rest-crt.pem
    opendistro_security.ssl.http.pemkey_filepath: elk-rest-key.pem
    opendistro_security.ssl.http.pemtrustedcas_filepath: elk-rest-root-ca.pem
    opendistro_security.ssl.transport.truststore_filepath: opendistro-es.truststore

    # See: https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/elasticsearch.yml.example#L23
    opendistro_security.authcz.admin_dn:
      - CN=admin-prod
```

#### logging.yml Config
All values defined under `elasticsearch.loggingConfig` will be converted to yaml and mounted into the config directory.

#### log4j2.properties Config
All values defined under `elasticsearch.log4jConfig` will be mounted into the config directory.

### Configuration
The following table lists the configurable parameters of the opendistro elasticsearch chart and their default values.

| Parameter                                                 | Description                                                                                                                                              | Default                                                                 |
|-----------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| `global.clusterName`                                      | Name of elasticsearch cluster                                                                                                                            | `"elasticsearch"`                                                       |
| `global.psp.create`                                       | Create and use `podsecuritypolicy`  resources                                                                                                            | `"true"`                                                                |
| `global.rbac.enabled`                                     | Create and use `rbac` resources                                                                                                                          | `"true"`                                                                |
| `global.imagePullSecrets`                                 | Global Docker registry secret names as an array                                                                                                          | `[]` (does not add image pull secrets to deployed pods)                 |
| `kibana.enabled`                                          | Enable the installation of kibana                                                                                                                        | `true`                                                                  |
| `kibana.image`                                            | Kibana container image                                                                                                                                   | `amazon/opendistro-for-elasticsearch-kibana`                            |
| `kibana.imageTag`                                         | Kibana container image  tag                                                                                                                              | `0.9.0`                                                                 |
| `kibana.replicas`                                         | Number of Kibana instances to deploy                                                                                                                     | `1`                                                                     |
| `kibana.port`                                             | Internal Port for service                                                                                                                                | `5601`                                                                  |
| `kibana.externalPort`                                     | External Port for service                                                                                                                                | `443`                                                                   |
| `kibana.resources`                                        | Kibana pod resource requests & limits                                                                                                                    | `{}`                                                                    |
| `kibana.elasticsearchAccount.secret`                      | The name of the secret with the Kibana server user as configured in your kibana.yml                                                                      | `""`                                                                    |
| `kibana.elasticsearchAccount.keyPassphrase.enabled`       | Enable mounting in keypassphrase for the `elasticsearchAccount`                                                                                          | `false`                                                                 |
| `kibana.ssl.kibana.enabled`                               | Enabled SSL for kibana                                                                                                                                   | `false`                                                                 |
| `kibana.ssl.kibana.existingCertSecret`                    | Name of secret that contains the Kibana certs                                                                                                            | `""`                                                                    |
| `kibana.ssl.elasticsearch.enabled`                        | Enable SSL for interactions between Kibana and Elasticsearch REST clients                                                                                | `false`                                                                 |
| `kibana.ssl.elasticsearch.existingCertSecret`             | Name of secret that contains the Elasticsearch REST certs                                                                                                | `""`                                                                    |
| `kibana.configDirectory`                                  | Location of where to mount in kibana specific configuration                                                                                              | `"/usr/share/kibana/config"`                                            |
| `kibana.certsDirectory`                                   | Location of where to mount in kibana certs configuration                                                                                                 | `"/usr/share/kibana/certs"`                                             |
| `kibana.ingress.enabled`                                  | Enable Kibana Ingress                                                                                                                                    | `false`                                                                 |
| `kibana.ingress.annotations`                              | Kibana Ingress annotations                                                                                                                               | `{}`                                                                    |
| `kibana.ingress.hosts`                                    | Kibana Ingress Hostnames                                                                                                                                 | `[]`                                                                    |
| `kibana.ingress.tls`                                      | Kibana Ingress TLS configuration                                                                                                                         | `[]`                                                                    |
| `kibana.ingress.labels`                                   | Kibana Ingress labels                                                                                                                                    | `{}`                                                                    |
| `kibana.ingress.path`                                     | Kibana Ingress paths                                                                                                                                     | `[]`                                                                    |
| `kibana.config`                                           | Kibana Configuration (`kibana.yml`)                                                                                                                      | `{}`                                                                    |
| `kibana.nodeSelector`                                     | Define which Nodes the Pods are scheduled on.                                                                                                            | `{}`                                                                    |
| `kibana.tolerations`                                      | If specified, the pod's tolerations.                                                                                                                     | `[]`                                                                    |
| `kibana.serviceAccount.create`                            | Create a default serviceaccount for Kibana to use                                                                                                        | `true`                                                                  |
| `kibana.serviceAccount.name`                              | Name for Kibana serviceaccount                                                                                                                           | `""`                                                                    |
| `kibana.extraEnvs`                                        | Extra environments variables to be passed to kibana                                                                                                      | `[]`                                                                    |
| `kibana.readinessProbe`                                   | Configuration for the [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                    | `[]`                                                                    |
| `kibana.livenessProbe`                                    | Configuration for the [livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                     | `[]`                                                                    |
| `elasticsearch.securityConfig.enabled`                    | Use custom [security configs](https://github.com/opendistro-for-elasticsearch/security/tree/master/securityconfig)                                       | `"true"`                                                                |
| `elasticsearch.securityConfig.path`                       | Path to security config files                                                                                                                            | `"/usr/share/elasticsearch/plugins/opendistro_security/securityconfig"` |
| `elasticsearch.securityConfig.actionGroupsSecret`         | Name of secret with [action_groups.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/action_groups.yml) defined   | `""`                                                                    |
| `elasticsearch.securityConfig.configSecret`               | Name of secret with [config.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/config.yml) defined                 | `""`                                                                    |
| `elasticsearch.securityConfig.internalUsersSecret`        | Name of secret with [internal_users.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/internal_users.yml) defined | `""`                                                                    |
| `elasticsearch.securityConfig.rolesSecret`                | Name of secret with [roles.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/roles.yml) defined                   | `""`                                                                    |
| `elasticsearch.securityConfig.rolesMappingSecret`         | Name of secret with [roles_mapping.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/roles_mapping.yml) defined   | `""`                                                                    |
| `elasticsearch.securityConfig.rolesMappingSecret`         | Name of secret with [roles_mapping.yml](https://github.com/opendistro-for-elasticsearch/security/blob/master/securityconfig/roles_mapping.yml) defined   | `""`                                                                    |
| `elasticsearch.ssl.transport.enabled`                     | Enable Transport SSL for Elasticsearch                                                                                                                   | `false`                                                                 |
| `elasticsearch.ssl.transport.existingCertSecret`          | Name of secret that contains the transport certs                                                                                                         | `""`                                                                    |
| `elasticsearch.ssl.rest.enabled`                          | Enable REST SSL for Elasticsearch                                                                                                                        | `false`                                                                 |
| `elasticsearch.ssl.rest.existingCertSecret`               | Name of secret that contains the Elasticsearch REST certs                                                                                                | `""`                                                                    |
| `elasticsearch.ssl.admin.enabled`                         | Enable Admin SSL cert usage for Elasticsearch                                                                                                            | `false`                                                                 |
| `elasticsearch.ssl.admin.existingCertSecret`              | Name of secret that contains the admin users Elasticsearch certs                                                                                         | `""`                                                                    |
| `elasticsearch.master.replicas`                           | Number of Elasticsearch masters to spin up                                                                                                               | `3`                                                                     |
| `elasticsearch.master.nodeAffinity`                       | Elasticsearch masters nodeAffinity                                                                                                                       | `{}`                                                                    |
| `elasticsearch.master.resources`                          | Elasticsearch masters resource requests & limits                                                                                                         | `{}`                                                                    |
| `elasticsearch.master.javaOpts`                           | Elasticsearch masters configurable java options to pass to startup script                                                                                | `"-Xms512m -Xmx512m"`                                                   |
| `elasticsearch.master.podDisruptionBudget.enabled`        | If true, create a disruption budget for elasticsearch master                                                                                             | `false`                                                                 |
| `elasticsearch.master.podDisruptionBudget.minAvailable`   | Minimum number / percentage of pods that should remain scheduled                                                                                         | `1`                                                                     |
| `elasticsearch.master.podDisruptionBudget.maxUnavailable` | Maximum number / percentage of pods that should remain scheduled                                                                                         | `""`                                                                    |
| `elasticsearch.master.tolerations`                        | If specified, the elasticsearch client pod's tolerations.                                                                                                | `[]`                                                                    |
| `elasticsearch.master.nodeSelector`                       | Define which Nodes the master pods are scheduled on.                                                                                                     | `{}`                                                                    |
| `elasticsearch.master.livenessProbe`                      | Configuration for the [livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                     | `[]`                                                                    |
| `elasticsearch.master.readinessProbe`                     | Configuration for the [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                    | `[]`                                                                    |
| `elasticsearch.client.replicas`                           | Number of Elasticsearch clients to spin up                                                                                                               | `3`                                                                     |
| `elasticsearch.client.nodeAffinity`                       | Elasticsearch clients nodeAffinity                                                                                                                       | `{}`                                                                    |
| `elasticsearch.client.resources`                          | Elasticsearch clients resource requests & limits                                                                                                         | `{}`                                                                    |
| `elasticsearch.client.javaOpts`                           | Elasticsearch clients configurable java options to pass to startup script                                                                                | `"-Xms512m -Xmx512m"`                                                   |
| `elasticsearch.client.service.type`                       | Elasticsearch clients service type                                                                                                                       | `ClusterIP`                                                             |
| `elasticsearch.client.service.annotations`                | Elasticsearch clients service annotations                                                                                                                | `{}`                                                                    |
| `elasticsearch.client.ingress.enabled`                    | Enable Elasticsearch clients Ingress                                                                                                                     | `false`                                                                 |
| `elasticsearch.client.ingress.annotations`                | Elasticsearch clients Ingress annotations                                                                                                                | `{}`                                                                    |
| `elasticsearch.client.ingress.hosts`                      | Elasticsearch clients Ingress Hostnames                                                                                                                  | `[]`                                                                    |
| `elasticsearch.client.ingress.tls`                        | Elasticsearch clients Ingress TLS configuration                                                                                                          | `[]`                                                                    |
| `elasticsearch.client.ingress.labels`                     | Elasticsearch clients Ingress labels                                                                                                                     | `{}`                                                                    |
| `elasticsearch.client.livenessProbe`                      | Configuration for the [livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                     | `[]`                                                                    |
| `elasticsearch.client.readinessProbe`                     | Configuration for the [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                    | `[]`                                                                    |
| `elasticsearch.client.podDisruptionBudget.enabled`        | If true, create a disruption budget for elasticsearch client                                                                                             | `false`                                                                 |
| `elasticsearch.client.podDisruptionBudget.minAvailable`   | Minimum number / percentage of pods that should remain scheduled                                                                                         | `1`                                                                     |
| `elasticsearch.client.podDisruptionBudget.maxUnavailable` | Maximum number / percentage of pods that should remain scheduled                                                                                         | `""`                                                                    |
| `elasticsearch.client.tolerations`                        | If specified, the elasticsearch client pod's tolerations.                                                                                                | `[]`                                                                    |
| `elasticsearch.client.nodeSelector`                       | Define which Nodes the client pods are scheduled on.                                                                                                     | `{}`                                                                    |
| `elasticsearch.data.replicas`                             | Number of Elasticsearch data nodes to spin up                                                                                                            | `3`                                                                     |
| `elasticsearch.data.nodeAffinity`                         | Elasticsearch data nodeAffinity                                                                                                                          | `{}`                                                                    |
| `elasticsearch.data.resources`                            | Elasticsearch data resource requests & limits                                                                                                            | `{}`                                                                    |
| `elasticsearch.data.javaOpts`                             | Elasticsearch data configurable java options to pass to startup script                                                                                   | `"-Xms512m -Xmx512m"`                                                   |
| `elasticsearch.data.storageClassName`                     | Elasticsearch data storageClassName                                                                                                                      | `"gp2-encrypted"`                                                       |
| `elasticsearch.data.storage`                              | Elasticsearch data storage                                                                                                                               | `"100Gi"`                                                               |
| `elasticsearch.data.podDisruptionBudget.enabled`          | If true, create a disruption budget for elasticsearch data node                                                                                          | `false`                                                                 |
| `elasticsearch.data.podDisruptionBudget.minAvailable`     | Minimum number / percentage of pods that should remain scheduled                                                                                         | `1`                                                                     |
| `elasticsearch.data.podDisruptionBudget.maxUnavailable`   | Maximum number / percentage of pods that should remain scheduled                                                                                         | `""`                                                                    |
| `elasticsearch.data.tolerations`                          | If specified, the elasticsearch client pod's tolerations.                                                                                                | `[]`                                                                    |
| `elasticsearch.data.nodeSelector`                         | Define which Nodes the data pods are scheduled on.                                                                                                       | `{}`                                                                    |
| `elasticsearch.data.livenessProbe`                        | Configuration for the [livenessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                     | `[]`                                                                    |
| `elasticsearch.data.readinessProbe`                       | Configuration for the [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)                    | `[]`                                                                    |
| `elasticsearch.config`                                    | Elasticsearch Configuration (`elasticsearch.yml`)                                                                                                        | `{}`                                                                    |
| `elasticsearch.loggingConfig`                             | Elasticsearch Logging Configuration (`logging.yml`)                                                                                                      | see `values.yaml` for defaults                                          |
| `elasticsearch.log4jConfig`                               | Elasticsearch log4j Configuration                                                                                                                        | `""`                                                                    |
| `elasticsearch.transportKeyPassphrase.enabled`            | Elasticsearch transport key passphrase required                                                                                                          | `false`                                                                 |
| `elasticsearch.transportKeyPassphrase.passPhrase`         | Elasticsearch transport key passphrase                                                                                                                   | `""`                                                                    |
| `elasticsearch.sslKeyPassphrase.enabled`                  | Elasticsearch ssl key passphrase required                                                                                                                | `false`                                                                 |
| `elasticsearch.sslKeyPassphrase.passPhrase`               | Elasticsearch ssl key passphrase                                                                                                                         | `""`                                                                    |
| `elasticsearch.image`                                     | Elasticsearch container image                                                                                                                            | `amazon/opendistro-for-elasticsearch`                                   |
| `elasticsearch.imageTag`                                  | Elasticsearch container image  tag                                                                                                                       | `0.9.0`                                                                 |
| `elasticsearch.serviceAccount.create`                     | Create a default serviceaccount for elasticsearch to use                                                                                                 | `true`                                                                  |
| `elasticsearch.initContainer.image`                       | Init container image                                                                                                                                     | `busybox`                                                               |
| `elasticsearch.initContainer.imageTag`                    | Init container image Tag                                                                                                                                 | `busybox`                                                               |
| `elasticsearch.serviceAccount.name`                       | Name for elasticsearch serviceaccount                                                                                                                    | `""`                                                                    |
| `elasticsearch.configDirectory`                           | Location of elasticsearch configuration                                                                                                                  | `"/usr/share/elasticsearch/config"`                                     |
| `elasticsearch.maxMapCount`                               | elasticsearch max_map_count                                                                                                                              | `262144`                                                                |
| `elasticsearch.extraEnvs`                                 | Extra environments variables to be passed to elasticsearch services                                                                                      | `[]`                                                                    |

## Acknowledgements
* [Kalvin Chau](https://github.com/kalvinnchau) (Software Engineer - Viasat) for all his help with the Kubernetes internals, certs, and debugging
