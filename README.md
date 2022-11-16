---
description: >-
  This page describes the LOGIQ deployment on Kubernetes cluster using HELM 3
  charts.
---

# K8S Quickstart guide

## 1 - Prerequisites

LOGIQ K8S components are made available as helm charts. Instructions below assume you are using HELM 3. Please read and agree to the [EULA](https://docs.logiq.ai/eula/eula) before proceeding.

### 1.1 Add LOGIQ helm repository

```bash
helm repo add logiq-repo https://logiqai.github.io/helm-charts
```

> The HELM repository will be named `logiq-repo`. For installing charts from this repository please make sure to use the repository name as the prefix e.g.
> 
> `helm install <deployment_name> logiq-repo/<chart_name>`
>

You can now run `helm search repo logiq-repo` to see the available helm charts

```bash
$ helm search repo logiq-repo
NAME                CHART VERSION    APP VERSION      DESCRIPTION
logiq-repo/logiq      v3.0.6           v3.0.6      LOGIQ Observability HELM chart for Kubernetes
```

### 1.2 Create namespace where LOGIQ will be deployed

> NOTE: Namespace name cannot be more than 15 characters in length

```bash
kubectl create namespace logiq
```

This will create a namespace **`logiq`** where we will deploy the LOGIQ Log Insights stack.

> If you choose a different name for the namespace, please remember to use the same namespace for the remainder of the steps

### 1.3 Prepare your Values YAML file

Sample YAML files for small, medium, large cluster configs can be downloaded at the links below

[LOGIQ Sample Values.yaml files](https://docs.logiq.ai/logiq-server/k8s-quickstart-guide#1-3-prepare-your-values-yaml-file)

These YAML files can be used for deployment with -f parameter as shown below in the description.

```bash
helm install logiq --namespace logiq \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq -f values.small.yaml
```
Please refer [Section 3.10 ](k8s-quickstart-guide.md#3-10-sizing-your-LOGIQ-cluster) for sizing your LOGIQ cluster as specified  in these yaml files.

## 2. Install LOGIQ

```bash
helm install logiq --namespace logiq \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

This will install LOGIQ and expose the LOGIQ services and UI on the ingress IP. Please refer [Section 3.4 ](k8s-quickstart-guide.md#3-4-changing-the-storage-class)for details about storage class. Service ports are described in the [Port details section](https://docs.logiq.ai/logiq-server/quickstart-guide#ports). You should now be able to go to `http://ingress-ip/`

> The default login and password to use is `flash-admin@foo.com` and `flash-password`. You can change these in the UI once logged in. Helm chart can override the default admin settings as well. See section[ 3.7](k8s-quickstart-guide.md#3-7-customize-admin-account) on customizing the admin settings

<img width="1786" alt="landing" src="https://user-images.githubusercontent.com/67860971/178478406-f183ecbf-678a-482a-8d4c-3edd45e7b405.png">


LOGIQ server provides Ingest, log tailing, data indexing, query and search capabilities.  
Besides the web based UI, LOGIQ also offers [logiqctl, LOGIQ CLI](https://docs.logiq.ai/logiq-cli) for accessing the above features.

## 3 Customizing the deployment

### 3.1 Enabling https for the UI

```bash
helm install logiq --namespace logiq \
--set global.domain=logiq.my-domain.com \
--set ingress.tlsEnabled=true \
--set kubernetes-ingress.controller.defaultTLSSecret.enabled=true \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

> You should now be able to login to LOGIQ UI at your domain using `https://logiq.my-domain.com` that you set in the ingress after you have updated your DNS server to point to the Ingress controller service IP
>
> The default login and password to use is `flash-admin@foo.com` and `flash-password`. You can change these in the UI once logged in.

> The `logiq.my-domain.com` also fronts all the LOGIQ service ports as described in the [port details section](quickstart-guide.md#ports).

| HELM Option | Description | Defaults |
| :--- | :--- | :--- |
| `global.domain` | DNS domain where the LOGIQ service will be running. This is required for HTTPS | No default |
| `ingress.tlsEnabled` | Enable the ingress controller to front HTTPS for services | false |
| `kubernetes-ingress.controller.defaultTLSSecret.enabled` | Specify if a default certificate is enabled for the ingress gateway | false |
| `kubernetes-ingress.controller.defaultTLSSecret.secret` | Specify the name of a TLS Secret for the ingress gateway. If this is not specified, a secret is automatically generated of option `kubernetes-ingress.controller.defaultTLSSecret.enabled` above is enabled. |  |

#### 3.1.1 Passing an ingress secret

If you want to pass your own ingress secret, you can do so when installing the HELM chart

```bash
helm install logiq --namespace logiq \
--set global.domain=logiq.my-domain.com \
--set ingress.tlsEnabled=true \
--set kubernetes-ingress.controller.defaultTLSSecret.enabled=true \
--set kubernetes-ingress.controller.defaultTLSSecret.secret=<secret_name> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

### 3.2 Using an AWS S3 bucket

Depending on your requirements, you may want to host your storage in your own K8S cluster or create a bucket in a cloud provider like AWS.

> Please note that cloud providers may charge data transfer costs between regions. It is important that the LOGIQ cluster be deployed in the same region where the S3 bucket is hosted

#### 3.2.1 Create an access/secret key pair for creating and managing your bucket <a id="3-1-1"></a>

Go to AWS IAM console and create an access key and secret key that can be used to create your bucket and manage access to the bucket for writing and reading your log files

#### 3.2.2 Deploy the LOGIQ helm in gateway mode

Make sure to pass your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` and give a bucket name. The S3 gateway acts as a caching gateway and helps reduce API costs.  
Create a bucket in AWS s3 with a unique bucket name in the region where you plan to host the deployment.

> You do not need to create the bucket, we will automatically provision it for you. Just provide the bucket name and access credentials in the step below.
>
> If the bucket already exists, LOGIQ will use it. Check to make sure the access and secret key work with it. Additionally, provide a valid amazon service endpoint for s3 else the config defaults to [https://s3.us-east-1.amazonaws.com](https://s3.us-east-1.amazonaws.com)

```bash
helm install logiq --namespace logiq --set global.domain=logiq.my-domain.com \
--set global.environment.s3_bucket=<bucket_name> \
--set global.environment.awsServiceEndpoint=https://s3.<region>.amazonaws.com \
--set global.environment.AWS_ACCESS_KEY_ID=<access_key> \
--set global.environment.AWS_SECRET_ACCESS_KEY=<secret_key> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

| HELM Option | Description | Defaults |
| :--- | :--- | :--- |
| `global.environment.s3_bucket` | Name of the S3 bucket in AWS | logiq |
| `global.environment.awsServiceEndpoint` | S3 Service endpoint : [https://s3.\*\*&lt;region&gt;\*\*.amazonaws.com](https://s3.**<region>**.amazonaws.com) | [https://s3.us-east-1.amazonaws.com](https://s3.us-east-1.amazonaws.com) |
| `global.environment.AWS_ACCESS_KEY_ID` | AWS Access key for accessing the bucket | No default |
| `global.environment.AWS_SECRET_ACCESS_KEY` | AWS Secret key for accessing the bucket | No default |
| `global.environment.s3_region` | AWS Region where the bucket is hosted | us-east-1 |

> S3 providers may have restrictions on bucket names for e.g. AWS S3 bucket names are globally unique.

### 3.3 Install LOGIQ server certificates and Client CA `[OPTIONAL]`

LOGIQ supports TLS for all ingest. We also enable non-TLS ports by default. It is however recommended that non-TLS ports not be used unless running in a secure VPC or cluster. The certificates can be provided to the cluster using K8S secrets. Replace the template sections below with your Base64 encoded secret files.

> If you skip this step, the LOGIQ server automatically generates a ca and a pair of client and server certificates for you to use. you can get them from the ingest server pods under the folder `/flash/certs`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: logiq-certs
type: Opaque
data:
  ca.crt: {{ .Files.Get "certs/ca.crt.b64" }}
  syslog.crt: {{ .Files.Get "certs/syslog.crt.b64" }}
  syslog.key: {{ .Files.Get "certs/syslog.key.b64" }}
```

Save the secret file e.g. `logiq-certs.yaml`. Proceed to install the secret in the same namespace where you want to deploy LOGIQ

The secret can now be passed into the LOGIQ deployment

```bash
helm install logiq --namespace logiq --set global.domain=logiq.my-domain.com \
--set logiq-flash.secrets_name=logiq-certs \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

| HELM Option | Description | Defaults |
| :--- | :--- | :--- |
| `logiq-flash.secrets_name` | TLS certificate key pair and CA cert for TLS transport | No default |

### 3.4 Changing the storage class

If you are planning on using a specific storage class for your volumes, you can customize it for the LOGIQ deployment. By default, LOGIQ uses the `standard` storage class

> It is quite possible that your environment may use a different storage class name for the provisioner. In that case please use the appropriate storage class name. E.g. if a user creates a storage class `ebs-volume` for the EBS provisioner for their cluster, you can use `ebs-volume` instead of `gp2` as suggested below

| Cloud Provider | K8S StorageClassName | Default Provisioner |
| :--- | :--- | :--- |
| AWS | gp2 | EBS |
| Azure | standard | azure-disc |
| GCP | standard | pd-standard |
| Digital Ocean | do-block-storage | Block Storage Volume |
| Oracle | oci | Block Volume |

```bash
helm upgrade --namespace logiq \
--set global.persistence.storageClass=<storage class name> \
logiq logiq-repo/logiq
```

### 3.5 Using external AWS RDS Postgres database instance

To use external AWS RDS Postgres database for your LOGIQ deployment, execute the following command.

```bash
helm install logiq --namespace logiq \
--set global.chart.postgres=false \
--set global.environment.postgres_host=<postgres-host-ip/dns> \
--set global.environment.postgres_user=<username> \
--set global.environment.postgres_password=<password> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

| HELM Option | Description | Default |
| :--- | :--- | :--- |
| `global.chart.postgres` | Deploy Postgres which is needed for LOGIQ metadata. Set this to false if an external Postgres cluster is being used | true |
| `global.environment.postgres_host` | Host IP/DNS for external Postgres | postgres |
| `global.environment.postgres_user` | Postgres admin user | postgres |
| `global.environment.postgres_password` | Postgres admin user password | postgres |
| `global.environment.postgres_port` | Host Port for external Postgres | 5432 |

> While configuring RDS, create a new parameter group that sets autoVaccum to true or the value "1", associate this parameter group to your RDS instance.
>
> Auto vacuum automates the execution of `VACUUM` and `ANALYZE` \(to gather statistics\) commands. Auto vacuum checks for bloated tables in the database and reclaims the space for reuse.

### 3.6 Upload LOGIQ professional license

The deployment described above offers 30 days trial license. Email `license@logiq.ai` to obtain a professional license. After obtaining the license, use the logiqctl tool to apply the license to the deployment. Please refer `logiqctl` details at [https://logiqctl.logiq.ai/](https://logiqctl.logiq.ai/). You will need API-token from LOGIQ UI as shown below

![Logiq Insights Login Api-token ](https://github.com/logiqai/docs/raw/master/.gitbook/assets/Screen-Shot-2020-08-09-ALERT.png)

```bash
Setup your LOGIQ Cluster endpoint
- logiqctl config set-cluster logiq.my-domain.com

Sets a logiq ui api token
- logiqctl config set-token api_token

Upload your LOGIQ deployment license
- logiqctl license set -l=license.jws

View License information
 - logiqctl license get
```

### 3.7 Customize Admin account

```bash
helm install logiq --namespace logiq \
--set global.environment.admin_name="LOGIQ Administrator" \
--set global.environment.admin_password="admin_password" \
--set global.environment.admin_email="admin@example.com" \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

| HELM Option | Description | Default |
| :--- | :--- | :--- |
| `global.environment.admin_name` | LOGIQ Administrator name | flash-admin@foo.com |
| `global.environment.admin_password` | LOGIQ Administrator password | flash-password |
| `global.environment.admin_email` | LOGIQ Administrator e-mail | flash-admin@foo.com |

### 3.8 Using external Redis instance

To use external Redis for your LOGIQ deployment, execute the following command.

> NOTE: At this time LOGIQ only supports connecting to a Redis cluster in a local VPC without authentication

```bash
helm install logiq --namespace logiq \
--set global.chart.redis=false \
--set global.environment.redis_host=<redis-host-ip/dns> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

| HELM Option | Description | Default |
| :--- | :--- | :--- |
| `global.chart.redis` | Deploy Redis which is needed for log tailing. Set this to false if an external Redis cluster is being used | true |
| `global.environment.redis_host` | Host IP/DNS of the external Redis cluster | redis-master |
| `global.environment.redis_port` | Host Port where external Redis service is exposed | 6379 |

### 3.9 Configuring cluster id

When deploying LOGIQ, configure the cluster id to monitor your own LOGIQ deployment. For details about the `cluster_id` refer to section [Managing multiple K8S clusters](agentless.md#managing-multiple-k-8-s-clusters-in-a-single-logiq-instance)

```bash
helm install logiq --namespace logiq \
--set global.environment.cluster_id=<cluster id> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

| HELM Option | Description | Default |
| :--- | :--- | :--- |
| global.environment.cluster\_id | Cluster Id being used for the K8S cluster running LOGIQ. See Section on [Managing multiple K8S](agentless.md#managing-multiple-k-8-s-clusters-in-a-single-logiq-instance) clusters for more details. | LOGIQ |

### 3.10 Sizing your LOGIQ cluster

When deploying LOGIQ, size your infrastructure to provide appropriate vcpu and memory requirements. We recommened the following minimum size for small. medium and large cluster specification from [Section 1.3 ](k8s-quickstart-guide.md#1-3-prepare-your-values-YAML-file) values yaml files.

| LOGIQ Cluster | vCPU| Memory | NodeCount |
| :--- | :--- | :--- | :--- |
| small | 12| 32 gb | 3 |
| medium  | 20| 56 gb | 5 |
| large  | 32| 88 gb | 8 |

### 3.11 NodePort/ClusterIP/LoadBalancer

The service type configurations are exposed in values.yaml as below 

```bash
flash-coffee:
  service:
    type: ClusterIP
logiq-flash:
  service:
    type: NodePort
kubernetes-ingress:
  controller:
    service:
      type: LoadBalancer

```

For e.g. if you are running on bare-metal and want an external LB to front LOGIQ, configure all services as `NodePort`

```bash
helm install logiq -n logiq -f values.yaml \
--set flash-coffee.service.type=NodePort \
--set logiq-flash.service.type=NodePort \
--set kubernetes-ingress.controller.service.type=NodePort \
logiq-repo/logiq
```

### 3.12 Using Node Selectors

The LOGIQ stack deployment can be optimized using node labels and node selectors to place various components of the stack optimally

```bash
logiq.ai/node=ingest
```

The node label `logiq.ai/node` above can be used to control the placement of ingest pods for log data into ingest optimized nodes. This allows for managing cost and instance sizing effectively.

The various nodeSelectors are defined in the globals section of the values.yaml file

```bash
globals:
  nodeSelectors:
    enabled: true
    ingest: ingest
    infra: common
    other: common
    db: common
    cache: common
    ingest_sync: common
```

In the example above, there are two node selectors in use - `ingest` and `common`. 

> Node selectors are enabled by setting `enabled` to `true` for `globals.nodeSelectors`

### 3.13 Installing Grafana

The LOGIQ stack includes Grafana as part of the deployment as an optional component. To enable Grafana in your cluster, follow the steps below

```bash
helm upgrade --install logiq --namespace logiq \
--set global.chart.grafana=true \ 
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

The Grafana instance is exposed at port 3000 on the ingress controller.

## 4 Teardown

If and when you want to decommission the installation using the following commands

```bash
helm delete logiq --namespace logiq
helm repo remove logiq-repo
kubectl delete namespace logiq
```

If you followed installation steps in section 3.1 - Using an AWS S3 bucket, you may want to delete the s3 bucket that was specified at deployment time.

