# JFrog Xray HA on Kubernetes Helm Chart

## Openshift
The Xray chart has been made a subchart of this chart.

Note due to this change we now reference values through the subchart name as shown below:

original:
``` 
xray.jfrogUrl
```

now:
``` 
xray.xray.jfrogUrl
```

This is due to helm referencing the value through the subchart named xray now.

## Security Context Constraints

To deploy this helm chart you will need to be a cluster admin w/ access to the anyuid scc.

````bash
oc adm policy add-scc-to-user anyuid -z my_service_account -n my_namespace
````

# Master and Join Key

The master and join key used to deploy Artifactory must be supplied to Xray at the time of installation.

## Deploying the Helm Chart

1. Deploy a Postgresql to use an external database. You can find additional information on how to configure your Postgresql database [here](https://www.jfrog.com/confluence/display/JFROG/Configuring+the+Database).
2. Run `helm dep build` to pull the subchart referenced by the `requirements.yaml`
3. Update POSTGRES_HOST, MASTER_KEY, JOIN_KEY variables below and install `openshift-xray` with the example commands:

````bash
POSTGRES_HOST=postgres-postgresql
MASTER_KEY=my_artifactory_master_key
JOIN_KEY=my_artifactory_join_key
helm upgrade --install openshift-xray . \
               --set xray.database.url=postgres://$POSTGRES_HOST:5432/xraydb?sslmode=disable \
               --set xray.database.user=artifactory \
               --set xray.database.password=password \
               --set xray.xray.jfrogUrl=http://openshift-artifactory-ha-nginx" \
               --set xray.xray.joinKey=$JOIN_KEY \
               --set xray.xray.masterKey=$MASTER_KEY
````


## Prerequisites Details

* Kubernetes 1.12+

## Chart Details

This chart will do the following:

* Optionally deploy PostgreSQL
* Deploy RabbitMQ (optionally as an HA cluster)
* Deploy JFrog Xray micro-services

## Requirements

- A running Kubernetes cluster
  - Dynamic storage provisioning enabled
  - Default StorageClass set to allow services using the default StorageClass for persistent storage
- A running Artifactory
- [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) installed and setup to use the cluster
- [Helm](https://helm.sh/) v2 or v3 installed


## Install JFrog Xray

### Add JFrog Helm repository

Before installing JFrog helm charts, you need to add the [JFrog helm repository](https://charts.jfrog.io/) to your helm client

```bash
helm repo add jfrog https://charts.jfrog.io
```

### Install Chart

#### Artifactory Connection Details

In order to connect Xray to your Artifactory installation, you have to use a Join Key, hence it is *MANDATORY* to provide a Join Key and Jfrog Url to your Xray installation. Here's how you do that:

Retrieve the connection details of your Artifactory installation, from the UI - https://www.jfrog.com/confluence/display/JFROG/General+Security+Settings#GeneralSecuritySettings-ViewingtheJoinKey. 

#### Initiate Installation

Provide join key and jfrog url as a parameter to the Xray chart installation:

```bash
helm upgrade --install --set xray.joinKey=<YOUR_PREVIOUSLY_RETIREVED_JOIN_KEY> \
             --set xray.jfrogUrl=<YOUR_PREVIOUSLY_RETIREVED_BASE_URL>  --namespace xray jfrog/xray
```

Alternatively, you can create a secret containing the join key manually and pass it to the template at install/upgrade time.
```bash

# Create a secret containing the key. The key in the secret must be named join-key
kubectl create secret generic my-secret --from-literal=join-key=<YOUR_PREVIOUSLY_RETIREVED_JOIN_KEY>

# Pass the created secret to helm
helm upgrade --install --set xray.joinKeySecretName=my-secret --namespace xray jfrog/xray
```
**NOTE:** In either case, make sure to pass the same join key on all future calls to `helm install` and `helm upgrade`! This means always passing `--set xray.joinKey=<YOUR_PREVIOUSLY_RETIREVED_JOIN_KEY>`. In the second, this means always passing `--set xray.joinKeySecretName=my-secret` and ensuring the contents of the secret remain unchanged.


### System Configuration

Xray uses a common system configuration file - `system.yaml`. See [official documentation](https://www.jfrog.com/confluence/display/JFROG/System+YAML+Configuration+File) on its usage.

## Status

See the status of your deployed **helm** releases

```bash
helm status xray
```

## Upgrade
To upgrade an existing Xray, you still use **helm**

```bash
# Update existing deployed version to 2.1.2
helm upgrade --set common.xrayVersion=2.1.2 jfrog/xray
```

If Xray was installed without providing a value to postgresql.postgresqlPassword (a password was autogenerated), follow these instructions:
1. Get the current password by running:

```bash
POSTGRES_PASSWORD=$(kubectl get secret -n <namespace> <myrelease>-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)
```

2. Upgrade the release by passing the previously auto-generated secret:

```bash
helm upgrade <myrelease> jfrog/xray --set postgresql.postgresqlPassword=${POSTGRES_PASSWORD}
```

If Xray was installed without providing a value to rabbitmq.rabbitmqPassword/rabbitmq-ha.rabbitmqPassword (a password was autogenerated), follow these instructions:
1. Get the current password by running:

```bash
RABBITMQ_PASSWORD=$(kubectl get secret -n <namespace> <myrelease>-rabbitmq -o jsonpath="{.data.rabbitmq-password}" | base64 --decode)
```

2. Upgrade the release by passing the previously auto-generated secret:

```bash
helm upgrade <myrelease> jfrog/xray --set rabbitmq.rabbitmqPassword=${RABBITMQ_PASSWORD}/rabbitmq-ha.rabbitmqPassword=${RABBITMQ_PASSWORD}
```

If Xray was installed with all of the default values (e.g. with no user-provided values for rabbit/postgres), follow these steps:
1. Retrieve all current passwords (rabbitmq/postgresql) as explained in the above section.
2. Upgrade the release by passing the previously auto-generated secrets:

```bash
helm upgrade --install xray --namespace xray jfrog/xray --set rabbitmq-ha.rabbitmqPassword=<rabbit-password> --set postgresql.postgresqlPassword=<postgresql-password>
```

## Remove

Removing a **helm** release is done with

```bash
# Remove the Xray services and data tools

#On helm v2:
helm delete --purge xray

#On helm v3:
helm delete xray --namespace xray

# Remove the data disks
kubectl delete pvc -l release=xray
```

### Deploying Xray for small/medium/large instllations
In the chart directory, we have added three values files, one for each installation type - small/medium/large. These values files are recommendations for setting resources requests and limits for your installation. The values are derived from the following [documentation](https://www.jfrog.com/confluence/display/EP/Installing+on+Kubernetes#InstallingonKubernetes-Systemrequirements). You can find them in the corresponding chart directory -  values-small.yaml, values-medium.yaml and values-large.yaml

### Create a unique Master Key

JFrog Xray requires a unique master key to be used by all micro-services in the same cluster. By default the chart has one set in values.yaml (`xray.masterKey`).

**This key is for demo purpose and should not be used in a production environment!**

You should generate a unique one and pass it to the template at install/upgrade time.

```bash
# Create a key
export MASTER_KEY=$(openssl rand -hex 32)
echo ${MASTER_KEY}

# Pass the created master key to helm
helm upgrade --install --set xray.masterKey=${MASTER_KEY} --namespace xray jfrog/xray

```

Alternatively, you can create a secret containing the master key manually and pass it to the template at install/upgrade time.
```bash
# Create a key
export MASTER_KEY=$(openssl rand -hex 32)
echo ${MASTER_KEY}

# Create a secret containing the key. The key in the secret must be named master-key
kubectl create secret generic my-secret --from-literal=master-key=${MASTER_KEY}

# Pass the created secret to helm
helm upgrade --install xray --set xray.masterKeySecretName=my-secret --namespace xray jfrog/xray
```
**NOTE:** In either case, make sure to pass the same master key on all future calls to `helm install` and `helm upgrade`! In the first case, this means always passing `--set xray.masterKey=${MASTER_KEY}`. In the second, this means always passing `--set xray.masterKeySecretName=my-secret` and ensuring the contents of the secret remain unchanged.


## Special deployments
This is a list of special use cases for non-standard deployments

### High Availability

For **high availability** of Xray, set the replica count to be equal or higher than **2**. Recommended is **3**.
> It is highly recommended to also set **RabbitMQ** to run as an HA cluster.

```bash
# Start Xray with 3 replicas per service and 3 replicas for RabbitMQ
helm upgarde --install xray --namespace xray --set server.replicaCount=3 jfrog/xray
```

### External Databases
There is an option to use external PostgreSQL database for your Xray.

#### PostgreSQL

##### PostgreSQL without TLS

To use an external **PostgreSQL**, you need to disable the use of the bundled **PostgreSQL** and set a custom **PostgreSQL** connection URL.

For this, pass the parameters: `postgresql.enabled=false` and `database.url=${XRAY_POSTGRESQL_CONN_URL}`.

**IMPORTANT:** Make sure the DB is already created before deploying Xray services

```bash
# Passing a custom PostgreSQL to Xray

# Example
export POSTGRESQL_HOST=custom-postgresql-host
export POSTGRESQL_PORT=5432
export POSTGRESQL_USER=xray
export POSTGRESQL_PASSWORD=password2_X
export POSTGRESQL_DATABASE=xraydb

export XRAY_POSTGRESQL_CONN_URL="postgres://${POSTGRESQL_HOST}:${POSTGRESQL_PORT}/${POSTGRESQL_DATABASE}?sslmode=disable"
helm upgrade --install xray --namespace xray \
    --set postgresql.enabled=false \
    --set database.url="${XRAY_POSTGRESQL_CONN_URL}" \
    --set database.user="${POSTGRESQL_USER}" \
    --set database.password="${POSTGRESQL_PASSWORD}" \
    jfrog/xray
```

##### PostgreSQL with TLS
If external **PostgreSQL** is set with TLS, you need to disable the use of the bundled **PostgreSQL**, set a custom **PostgreSQL** connection URL and provide a secret with **PostgreSQL** TLS certificates.

Create the Kubernetes secret (assuming the local files are `client-cert.pem	client-key.pem server-ca.pem`)

```bash
kubectl create secret generic postgres-tls --from-file=client-key.pem --from-file=client-cert.pem --from-file=server-ca.pem

```

**IMPORTANT:** `PostgreSQL` connection URL needs to have listed TLS files with the path `/var/opt/jfrog/xray/data/tls/` 
and `sslmode==verify-ca` otherwise Xray will fail to connect to Postgres.

```bash
# Passing a custom PostgreSQL with TLS to Xray

# Example
export POSTGRESQL_HOST=custom-postgresql-host
export POSTGRESQL_PORT=5432
export POSTGRESQL_USER=xray
export POSTGRESQL_PASSWORD=password2_X
export POSTGRESQL_DATABASE=xraydb
export POSTGRESQL_SERVER_CA=server-ca.pem
export POSTGRESQL_CLIENT_CERT=client-key.pem
export POSTGRESQL_CLIENT_KEY=client-cert.pem
export POSTGRESQL_TLS_SECRET=postgres-tls

export XRAY_POSTGRESQL_CONN_URL="postgres://${POSTGRESQL_HOST}:${POSTGRESQL_PORT}/${POSTGRESQL_DATABASE}?sslrootcert=/var/opt/jfrog/xray/data/tls/${POSTGRESQL_SERVER_CA}&sslkey=/var/opt/jfrog/xray/data/tls/${POSTGRESQL_CLIENT_KEY}&sslcert=/var/opt/jfrog/xray/data/tls/${POSTGRESQL_CLIENT_CERT}&sslmode=verify-ca"
helm upgrade --install xray --namespace xray \
    --set postgresql.enabled=false \
    --set database.url="${XRAY_POSTGRESQL_CONN_URL}" \
    --set database.user="${POSTGRESQL_USER}" \
    --set database.password="${POSTGRESQL_PASSWORD}" \
    jfrog/xray
```

### Custom init containers

There are cases where a special, unsupported init processes is needed like checking something on the file system or testing something before spinning up the main container.

For this, there is a section for writing custom init containers before and after the predefined init containers in the [values.yaml](values.yaml) . By default it's commented out

```yaml
common:
  ## Add custom init containers executed before predefined init containers
  customInitContainersBegin: |
    ## Init containers template goes here ##

    ## Add custom init containers executed after predefined init containers
  customInitContainers: |
    ## Init containers template goes here ##
```

## Configuration

The following table lists the configurable parameters of the xray chart and their default values.

|         Parameter            |                    Description                   |           Default                  |
|------------------------------|--------------------------------------------------|------------------------------------|
| `imagePullSecrets`           | Docker registry pull secret                      |                                    |
| `imagePullPolicy`            | Container pull policy                            | `IfNotPresent`                     |
| `initContainerImage`         | Init container image                             | `alpine:3.6`                       |
| `xray.jfrogUrl`              | Main Artifactory URL, without the `/artifactory` prefix .Mandatory  |                                    |
| `xray.persistence.mountPath` | Xray persistence mount path                      | `/var/opt/jfrog/xray`              |
| `xray.masterKey`             | Xray Master Key (Can be generated with `openssl rand -hex 32`) | `` |
| `xray.masterKeySecretName`   | Xray Master Key secret name                      |                                                                    |
| `xray.joinKey`               | Xray Join Key to connect to Artifactory . Mandatory | `` |
| `xray.joinKeySecretName`     | Xray Join Key secret name |                                                  |
| `xray.systemYaml`            | Xray system configuration (`system.yaml`) as described here - https://www.jfrog.com/confluence/display/JFROG/Xray+System+YAML |       |
| `xray.autoscaling.enabled`   | Enable Xray Pods autoscaling using `HorizontalPodAutoscaler` | `false`                |
| `xray.autoscaling.minReplicas`   | Minimum number of Xray replicas | `1`                |
| `xray.autoscaling.maxReplicas`   | Maximum number of Xray replicas | `1`                |
| `xray.autoscaling.targetCPUUtilizationPercentage`   | CPU usage percentage that will trigger autoscaling | `50`                |
| `xray.autoscaling.targetMemoryUtilizationPercentage`   | Memory usage percentage that will trigger autoscaling | `75`                |
| `serviceAccount.create`   | Specifies whether a ServiceAccount should be created| `true`                             |
| `serviceAccount.name`     | The name of the ServiceAccount to create            | Generated using the fullname template |
| `rbac.create`             | Specifies whether RBAC resources should be created  | `true`                             |
| `rbac.role.rules`         | Rules to create                                     | `[]`                               |
| `postgresql.enabled`              | Use enclosed PostgreSQL as database         | `true`                             |
| `postgresql.image.registry`              | PostgreSQL Docker image registry         | `docker.bintray.io`                             |
| `postgresql.image.repository`              | PostgreSQL Docker image repository         | `bitnami/postgresql`                             |
| `postgresql.image.tag`              | PostgreSQL Docker image tag         | `9.6.15-debian-9-r91`                             |
| `postgresql.postgresqlUsername`         | PostgreSQL database user                    | `xray`                             |
| `postgresql.postgresqlPassword`     | PostgreSQL database password                | ` `                                |
| `postgresql.postgresqlDatabase`     | PostgreSQL database name                    | `xraydb`                           |
| `postgresql.postgresqlExtendedConf.listenAddresses`  | PostgreSQL listen address | `"'*'"`                           |
| `postgresql.postgresqlExtendedConf.maxConnections`  | PostgreSQL max_connections parameter | `500`                           |
| `postgresql.service.port`         | PostgreSQL database port                    | `5432`                             |
| `postgresql.persistence.enabled`  | PostgreSQL use persistent storage           | `true`                             |
| `postgresql.persistence.size`     | PostgreSQL persistent storage size          | `50Gi`                             |
| `postgresql.persistence.existingClaim`  | PostgreSQL name of existing Persistent Volume Claim to use          | ` `                  |
| `postgresql.resources.requests.memory`    | PostgreSQL initial memory request   |                                    |
| `postgresql.resources.requests.cpu`       | PostgreSQL initial cpu request      |                                    |
| `postgresql.resources.limits.memory`      | PostgreSQL memory limit             |                                    |
| `postgresql.resources.limits.cpu`         | PostgreSQL cpu limit                |                                    |
| `postgresql.nodeSelector`                 | PostgreSQL node selector            | `{}`                               |
| `postgresql.affinity`                     | PostgreSQL node affinity            | `{}`                               |
| `postgresql.tolerations`                  | PostgreSQL node tolerations         | `[]`                               |
| `database.url`                            | External database connection URL                   |                                         |
| `database.user`                           | External database username                         |                                         |
| `database.password`                       | External database password                         |                                         |
| `database.secrets.user.name`              | External database username `Secret` name           |                                         |
| `database.secrets.user.key`               | External database username `Secret` key            |                                         |
| `database.secrets.password.name`          | External database password `Secret` name           |                                         |
| `database.secrets.password.key`           | External database password `Secret` key            |                                         |
| `database.secrets.url.name`               | External database url `Secret` name                |                                         |
| `database.secrets.url.key`                | External database url `Secret` key                 |                                         |
| `rabbitmq.enabled`                             | RabbitMQ enabled uses rabbitmq               | `false`              |
| `rabbitmq.replicas`                            | RabbitMQ replica count               | `1`              |
| `rabbitmq.rbacEnabled`                         | If true, create & use RBAC resources         | `true`               |
| `rabbitmq.rabbitmq.username`                    | RabbitMQ application username                | `guest`               |
| `rabbitmq.rabbitmq.password`                    | RabbitMQ application password                |                |
| `rabbitmq.rabbitmq.existingPasswordSecret`      | RabbitMQ existingPasswordSecret               |                |
| `rabbitmq.rabbitmq.erlangCookie`                | RabbitMQ Erlang cookie                       | `XRAYRABBITMQCLUSTER`|
| `rabbitmq.service.nodePort`                    | RabbitMQ node port                           | `5672`               |
| `rabbitmq.persistence.enabled`            | If `true`, persistent volume claims are created | `true`            |
| `rabbitmq.persistence.accessMode`            | RabbitMQ persistent volume claims access mode | `ReadWriteOnce`            |
| `rabbitmq.persistence.size`               | RabbitMQ Persistent volume size              | `20Gi`               |
| `rabbitmq-ha.enabled`                          | RabbitMQ enabled uses rabbitmq-ha            | `true`               |
| `rabbitmq-ha.replicaCount`                     | RabbitMQ Number of replica                   | `1`                  |
| `rabbitmq-ha.rabbitmqUsername`                 | RabbitMQ application username                | `guest`              |
| `rabbitmq-ha.rabbitmqPassword`                 | RabbitMQ application password                | ` `                  |
| `rabbitmq-ha.existingSecret`                   | RabbitMQ existingSecret                      | ` `                  |
| `rabbitmq-ha.rabbitmqErlangCookie`             | RabbitMQ Erlang cookie                       | `XRAYRABBITMQCLUSTER`|
| `rabbitmq-ha.rabbitmqMemoryHighWatermark`      | RabbitMQ Memory high watermark               | `500MB`              |
| `rabbitmq-ha.persistentVolume.enabled`         | If `true`, persistent volume claims are created | `true`            |
| `rabbitmq-ha.persistentVolume.size`            | RabbitMQ Persistent volume size              | `20Gi`               |
| `rabbitmq-ha.rbac.create`                      | If true, create & use RBAC resources         | `true`               |
| `rabbitmq-ha.nodeSelector`                     | RabbitMQ node selector                       | `{}`                 |
| `rabbitmq-ha.tolerations`                      | RabbitMQ node tolerations                    | `[]`                 |
| `common.xrayVersion`                           | Xray image tag                               | `.Chart.AppVersion`  |
| `common.preStartCommand`                       | Xray Custom command to run before startup. Runs BEFORE any microservice-specific preStartCommand |     |
| `common.xrayUserId`                            | Xray User Id                                 | `1035`               |
| `common.xrayGroupId`                           | Xray Group Id                                | `1035`               |
| `common.persistence.enabled`                   | Xray common persistence volume enabled       | `false`              |
| `common.persistence.existingClaim`             | Provide an existing PersistentVolumeClaim    | `nil`                |
| `common.persistence.storageClass`              | Storage class of backing PVC                 | `nil (uses default storage class annotation)`      |
| `common.persistence.accessMode`                | Xray common persistence volume access mode   | `ReadWriteOnce`      |
| `common.persistence.size`                      | Xray common persistence volume size          | `50Gi`               |
| `xray.systemYaml`                              | Xray system configuration (`system.yaml`)    | `see values.yaml`    |
| `common.customInitContainersBegin`             | Custom init containers to run before existing init containers                       | ` `                  |
| `common.customInitContainers`                  | Custom init containers to run after existing init containers                       | ` `                  |
| `common.xrayConfig`                            | Additional xray yaml configuration to be written to xray_config.yaml file | See [values.yaml](stable/xray/values.yaml) |
| `database.url`                         | Xray external PostgreSQL URL                 | ` `                  |
| `global.postgresqlTlsSecret`                   | Xray external PostgreSQL TLS files secret    | ` `                  |
| `analysis.name`                                | Xray Analysis name                           | `xray-analysis`      |
| `analysis.image`                               | Xray Analysis container image                | `docker.bintray.io/jfrog/xray-analysis` |
| `analysis.updateStrategy`                      | Xray Analysis update strategy                | `RollingUpdate`      |
| `analysis.podManagementPolicy`                 | Xray Analysis pod management policy          | `Parallel`           |
| `analysis.internalPort`                        | Xray Analysis internal port                  | `7000`               |
| `analysis.externalPort`                        | Xray Analysis external port                  | `7000`               |
| `analysis.livenessProbe`                       | Xray Analysis livenessProbe                  | See `values.yaml`    |
| `analysis.readinessProbe`                      | Xray Analysis readinessProbe                 | See `values.yaml`    |
| `analysis.persistence.size`                    | Xray Analysis storage size limit             | `10Gi`               |
| `analysis.resources`                           | Xray Analysis resources                      | `{}`                 |
| `analysis.preStartCommand`                     | Xray Analysis Custom command to run before startup. Runs AFTER the `common.preStartCommand` |     |
| `analysis.nodeSelector`                        | Xray Analysis node selector                  | `{}`                 |
| `analysis.affinity`                            | Xray Analysis node affinity                  | `{}`                 |
| `analysis.tolerations`                         | Xray Analysis node tolerations               | `[]`                 |
| `analysis.annotations`                          | Xray Analysis annotations                     | `{}`                               |
| `indexer.name`                                 | Xray Indexer name                            | `xray-indexer`       |
| `indexer.image`                                | Xray Indexer container image                 | `docker.bintray.io/jfrog/xray-indexer`  |
| `indexer.annotations`                          | Xray Indexer annotations                     | `{}`                               |
| `indexer.updateStrategy`                       | Xray Indexer update strategy                 | `RollingUpdate`      |
| `indexer.podManagementPolicy`                  | Xray Indexer pod management policy           | `Parallel`           |
| `indexer.internalPort`                         | Xray Indexer internal port                   | `7002`               |
| `indexer.externalPort`                         | Xray Indexer external port                   | `7002`               |
| `indexer.livenessProbe`                        | Xray Indexer livenessProbe                   | See `values.yaml`    |
| `indexer.readinessProbe`                       | Xray Indexer readinessProbe                  | See `values.yaml`    |
| `indexer.customVolumes`                         | Custom volumes                               |                                                  |
| `indexer.customVolumeMounts`                    | Custom Server volumeMounts                   |                                                  |
| `indexer.persistence.existingClaim`            | Provide an existing PersistentVolumeClaim    | `nil`                              |
| `indexer.persistence.storageClass`             | Storage class of backing PVC                 | `nil (uses default storage class annotation)`      |
| `indexer.persistence.enabled`                  | Xray Indexer persistence volume enabled      | `false`                             |
| `indexer.persistence.accessMode`               | Xray Indexer persistence volume access mode  | `ReadWriteOnce`                    |
| `indexer.persistence.size`                     | Xray Indexer persistence volume size         | `50Gi`                             |
| `indexer.resources`                            | Xray Indexer resources                       | `{}`                 |
| `indexer.preStartCommand`                      | Xray Indexer Custom command to run before startup. Runs AFTER the `common.preStartCommand` |     |
| `indexer.nodeSelector`                         | Xray Indexer node selector                   | `{}`                 |
| `indexer.affinity`                             | Xray Indexer node affinity                   | `{}`                 |
| `indexer.tolerations`                          | Xray Indexer node tolerations                | `[]`                 |
| `persist.name`                                 | Xray Persist name                            | `xray-persist`       |
| `persist.image`                                | Xray Persist container image                 | `docker.bintray.io/jfrog/xray-persist`  |
| `persist.annotations`                          | Xray Persist annotations                     | `{}`                               |
| `persist.updateStrategy`                       | Xray Persist update strategy                 | `RollingUpdate`      |
| `persist.podManagementPolicy`                  | Xray Persist pod management policy           | `Parallel`           |
| `persist.internalPort`                         | Xray Persist internal port                   | `7003`               |
| `persist.externalPort`                         | Xray Persist external port                   | `7003`               |
| `persist.livenessProbe`                        | Xray Persist livenessProbe                   | See `values.yaml`    |
| `persist.readinessProbe`                       | Xray Persist readinessProbe                  | See `values.yaml`    |
| `persist.persistence.size`                     | Xray Persist storage size limit              | `10Gi`               |
| `persist.preStartCommand`                      | Xray Persist Custom command to run before startup. Runs AFTER the `common.preStartCommand` |     |
| `persist.resources`                            | Xray Persist resources                       | `{}`                 |
| `persist.nodeSelector`                         | Xray Persist node selector                   | `{}`                 |
| `persist.affinity`                             | Xray Persist node affinity                   | `{}`                 |
| `persist.tolerations`                          | Xray Persist node tolerations                | `[]`                 |
| `server.name`                                  | Xray server name                             | `xray-server`        |
| `server.image`                                 | Xray server container image                  | `docker.bintray.io/jfrog/xray-server`   |
| `server.annotations`                           | Xray server annotations                      | `{}`                               |
| `server.customVolumes`                         | Custom volumes                               |                                                  |
| `server.customVolumeMounts`                    | Custom Server volumeMounts                   |                                                  |
| `server.replicaCount`                          | Xray services replica count                  | `1`                  |
| `server.updateStrategy`                        | Xray server update strategy                  | `RollingUpdate`      |
| `server.podManagementPolicy`                   | Xray server pod management policy            | `Parallel`           |
| `server.internalPort`                          | Xray server internal port                    | `8000`               |
| `server.externalPort`                          | Xray server external port                    | `80`                 |
| `server.service.name`                          | Xray server service name                     | `xray`               |
| `server.service.type`                          | Xray server service type                     | `ClusterIP`       |
| `server.service.annotations`                   | Xray server service annotations              | `{}`                 |
| `server.livenessProbe`                         | Xray server livenessProbe                    | See `values.yaml`    |
| `server.readinessProbe`                        | Xray server readinessProbe                   | See `values.yaml`    |
| `server.preStartCommand`                       | Xray server Custom command to run before startup. Runs AFTER the `common.preStartCommand` |     |
| `server.resources`                             | Xray server resources                        | `{}`                 |
| `server.nodeSelector`                          | Xray server node selector                    | `{}`                 |
| `server.affinity`                              | Xray server node affinity                    | `{}`                 |
| `server.tolerations`                           | Xray server node tolerations                 | `[]`                 |
| `router.name`                                  | Router name                                  | `router`             |
| `router.image.repository`                      | Container image                              | `docker.bintray.io/jfrog/router`     |
| `router.image.version`                         | Container image tag                          | `.Chart.AppVersion`                |
| `router.image.pullPolicy`                      | Container pull policy                        | `IfNotPresent`                     |
| `router.internalPort`                          | Router internal port                         | `8082`                     |
| `router.externalPort`                          | Router external port                         | `8082`                     |
| `router.resources.requests.memory`             | Router initial memory request   |                                    |
| `router.resources.requests.cpu`                | Router initial cpu request      |                                    |
| `router.resources.limits.memory`               | Router memory limit             |                                    |
| `router.resources.limits.cpu`                  | Router cpu limit                |                                    |
| `router.livenessProbe.enabled`                 | Enable Router livenessProbe                         | `true`    |
| `router.livenessProbe.config`                  | Router livenessProbe configuration           | See `values.yaml`    |
| `router.readinessProbe.enabled`                | Enable Router readinessProbe                         | `true`    |
| `router.readinessProbe.config`                 | Router readinessProbe configuration           | See `values.yaml`    |
| `router.persistence.accessMode`                | Router persistence access mode           | `ReadWriteOnce`    |
| `router.persistence.mountPath`                 | Router persistence mount path           | `/var/opt/jfrog/router`    |
| `router.persistence.size`                      | Router persistence size           | `5Gi`    |
| `router.readinessProbe.config`                 | Router readinessProbe configuration           | See `values.yaml`    |
| `router.readinessProbe.config`                 | Router readinessProbe configuration           | See `values.yaml`    |
| `router.nodeSelector`                          | Router node selector                  | `{}`                 |
| `router.affinity`                              | Router node affinity                  | `{}`                 |
| `router.tolerations`                           | Router node tolerations               | `[]`                 |
| `filebeat.enabled`           | Enable a filebeat container to send your logs to a log management solution like ELK            | `false`   |
| `filebeat.name`           | filebeat container name            | `xray-filebeat`   |
| `filebeat.image.repository`           | filebeat Docker image repository            | `docker.elastic.co/beats/filebeat`   |
| `filebeat.image.version`           | filebeat Docker image version            | `7.5.1`   |
| `filebeat.logstashUrl`           | The URL to the central Logstash service, if you have one            | `logstash:5044`   |
| `filebeat.livenessProbe.exec.command`           | liveness probe exec command            | see [values.yaml](stable/xray/values.yaml)   |
| `filebeat.livenessProbe.failureThreshold`     | Minimum consecutive failures for the probe to be considered failed after having succeeded.   | 10 |
| `filebeat.livenessProbe.initialDelaySeconds`  | Delay before liveness probe is initiated  | 180                       |
| `filebeat.livenessProbe.periodSeconds`        | How often to perform the probe            | 10                        |
| `filebeat.readinessProbe.exec.command`           | readiness probe exec command            | see [values.yaml](stable/xray/values.yaml)   |
| `filebeat.readinessProbe.failureThreshold`     | Minimum consecutive failures for the probe to be considered failed after having succeeded.   | 10 |
| `filebeat.readinessProbe.initialDelaySeconds`  | Delay before readiness probe is initiated  | 180                       |
| `filebeat.readinessProbe.periodSeconds`        | How often to perform the probe            | 10                        |
| `filebeat.resources.requests.memory` | Filebeat initial memory request                  |                          |
| `filebeat.resources.requests.cpu`    | Filebeat initial cpu request     |                                          |
| `filebeat.resources.limits.memory`   | Filebeat memory limit            |                                          |
| `filebeat.resources.limits.cpu`      | Filebeat cpu limit               |                                          |
| `filebeat.filebeatYml`      | Filebeat yaml configuration file                | see [values.yaml](stable/xray/values.yaml)                                         |


Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

### Custom volumes

If you need to use a custom volume in a custom init or sidecar container, you can use this option.

For this, there is a section for defining custom volumes in the [values.yaml](values.yaml). By default it's commented out

```yaml
server:
  ## Add custom volumes
  customVolumes: |
    ## Custom volume comes here ##
```

## Useful links
- https://www.jfrog.com/confluence/display/XRAY/Xray+High+Availability
- https://www.jfrog.com/confluence/display/EP/Getting+Started
- https://www.jfrog.com/confluence/