# LOGIQ Observability - K8S Quickstart guide

## 1 - Introduction

LOGIQ is a monitoring and log analytics solution for infrastructure and application observability

1. [LOGIQ Home](https://logiq.ai)
2. [LOGIQ Documentation](https://docs.logiq.ai)

## 2 - Prerequisites

LOGIQ K8S components are made available as helm charts. Instructions below assume you are using HELM 3. Please read and agree to the [EULA](https://docs.logiq.ai/eula/eula) before proceeding.

### 2.1 Add LOGIQ helm repository


```bash
$ helm repo add logiq-repo https://logiqai.github.io/helm-charts
```

The HELM repository will be named `logiq-repo`. For installing charts from this repository please make sure to use the repository name as the prefix e.g.

`helm install <deployment_name> logiq-repo/<chart_name>`

You can now run `helm search repo logiq-repo` to see the available helm charts

```bash
$ helm search repo logiq-repo
NAME                CHART VERSION    APP VERSION    DESCRIPTION
logiq-repo/logiq    2.0.0            2.0.0          A Helm chart for Kubernetes
```

### 2.2 Create namespace where LOGIQ will be deployed

```bash
$ kubectl create namespace logiq
```

This will create a namespace **`logiq`** where we will deploy the LOGIQ Log Insights stack.

If you choose a different name for the namespace, please remember to use the same namespace for the remainder of the steps

## 3. Install LOGIQ

```bash
$ helm install logiq --namespace logiq \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

This will install LOGIQ and expose the LOGIQ services and UI on the ingress IP. Please refer Section 3.5 for details about storage class. Service ports are described in the [Port details section](https://docs.logiq.ai/logiq-server/quickstart-guide#ports). You should now be able to go to `http://ingress-ip/`

The default login and password to use is `flash-admin@foo.com` and `flash-password`. You can change these in the UI once logged in. Helm chart can override the default admin settings as well. See section[ 3.7](k8s-quickstart-guide.md#3-7-customize-admin-account) on customizing the admin settings

LOGIQ server provides Ingest, log tailing, data indexing, query and search capabilities.  
Besides the web based UI, LOGIQ also offers [logiqctl, LOGIQ CLI](https://docs.logiq.ai/logiq-cli) for accessing the above features.

## 4 Customizing the deployment

### 4.1 Enabling https for the UI

```bash
$ helm install logiq --namespace logiq \
--set global.domain=logiq.my-domain.com \
--set ingress.tlsEnabled=true \
--set kubernetes-ingress.controller.defaultTLSSecret.enabled=true \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

You should now be able to login to LOGIQ UI at your domain using `https://logiq.my-domain.com` that you set in the ingress after you have updated your DNS server to point to the Ingress controller service IP

The default login and password to use is `flash-admin@foo.com` and `flash-password`. You can change these in the UI once logged in.

The `logiq.my-domain.com` also fronts all the LOGIQ service ports as described in the [port details section](quickstart-guide.md#ports).

#### 4.1.1 Passing an ingress secret

If you want to pass your own ingress secret, you can do so when installing the HELM chart

```bash
$ helm install logiq --namespace logiq \
--set global.domain=logiq.my-domain.com \
--set ingress.tlsEnabled=true \
--set kubernetes-ingress.controller.defaultTLSSecret.enabled=true \
--set kubernetes-ingress.controller.defaultTLSSecret.secret=<secret_name> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

### 4.2 Using an AWS S3 bucket

Depending on your requirements, you may want to host your storage in your own K8S cluster or create a bucket in a cloud provider like AWS.

Please note that cloud providers may charge data transfer costs between regions. It is important that the LOGIQ cluster be deployed in the same region where the S3 bucket is hosted

#### 4.2.1 Create an access/secret key pair for creating and managing your bucket <a id="3-1-1"></a>

Go to AWS IAM console and create an access key and secret key that can be used to create your bucket and manage access to the bucket for writing and reading your log files

#### 4.2.2 Deploy the LOGIQ helm in gateway mode

Make sure to pass your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from [step 3.1.1](k8s-quickstart-guide.md#3-1-1) above and give a bucket name. The S3 gateway acts as a caching gateway and helps reduce API cost.

You do not need to create the bucket, we will automatically provision it for you. Just provide the bucket name and access credentials in the the step below.

If the bucket already exists, LOGIQ will use it. Check to make sure the access and secret key work with it. Additionally provide a valid amazon service endpoint for s3 else the config defaults to [https://s3.us-east-1.amazonaws.com](https://s3.us-east-1.amazonaws.com)

```bash
$ helm install logiq --namespace logiq --set global.domain=logiq.my-domain.com \
--set global.environment.s3_bucket=<bucket_name>
--set global.environment.awsServiceEndpoint=https://s3.<region>.amazonaws.com \
--set global.environment.AWS_ACCESS_KEY_ID=<access_key> \
--set global.environment.AWS_SECRET_ACCESS_KEY=<secret_key> \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

S3 providers may have restrictions on bucket name for e.g. AWS S3 bucket names are globally unique.

### 4.3 Install LOGIQ server certificates and Client CA `[OPTIONAL]`

LOGIQ supports TLS for all ingest. We also enable non-TLS ports by default. It is however recommended that non-TLS ports not be used unless running in a secure VPC or cluster. The certificates can be provided to the cluster using K8S secrets. Replace the template sections below with your Base64 encoded secret files.

If you skip this step, LOGIQ server automatically generates a ca and a pair of client and server certificates for you to use. you can get them from the ingest server pods under the folder `/flash/certs`

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
$ helm install logiq --namespace logiq --set global.domain=logiq.my-domain.com \
--set logiq-flash.secrets_name=logiq-certs \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

### 4.4 Changing the storage class

If you are planning on using a specific storage class for your volumes, you can customize it for the LOGIQ deployment. By default, LOGIQ uses the `standard` storage class

It is quite possible that your environment may use a different storage class name for the provisioner. In that case please use the appropriate storage class name. E.g. if a user creates a storage class `ebs-volume` for the EBS provisioner for their cluster, you can use `ebs-volume` instead of `gp2` as suggested below

| Cloud Provider | K8S StorageClassName | Default Provisioner |
| :--- | :--- | :--- |
| AWS | gp2 | EBS |
| Azure | standard | azure-disc |
| GCP | standard | pd-standard |
| Digital Ocean | do-block-storage | Block Storage Volume |
| Oracle | oci | Block Volume |

```bash
$ helm upgrade --namespace logiq \
--set global.persistence.storageClass=<storage class name> \
logiq logiq-repo/logiq
```

### 4.5 Using external AWS RDS Postgres database instance

To use external AWS RDS Postgres database for your LOGIQ deployment, execute the following command.

```bash
$ helm install --namespace logiq \
--set postgres.enabled=false \
--set global.environment.postgres_host=<postgres-host-ip/dns> \
--set global.environment.postgres_user=<username> \
--set global.environment.postgres_password=<password> \
logiq logiq-repo/logiq
```

### 4.6 Upload LOGIQ professional license

The deployment described above offers 30 days trial license. Email to license@logiq.ai to obtain the professional license. After obtaining the license, use the logiqctl tool to apply the license to the deployment. Please refer logiqctl details at [https://logiqctl.logiq.ai/](https://logiqctl.logiq.ai/). You will need api-token from LOGIQ ui once logged in.

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

### 4.7 Customize Admin account

```bash
$ helm install logiq --namespace logiq \
--set global.environment.admin_name="LOGIQ Administrator" \
--set global.environment.admin_password="admin_password" \
--set global.environment.admin_org="example_com_org" \
--set global.environment.admin_email="admin@example.com" \
--set global.persistence.storageClass=<storage class name> logiq-repo/logiq
```

## 5 Teardown

If and when you want to decommission the installation using the following commands

```bash
$ helm delete logiq --namespace logiq
$ helm repo remove logiq-repo
$ kubectl delete namespace logiq
```

If you followed installation steps in section 4.2 - Using an AWS S3 bucket, you may want to delete the s3 bucket that was specified at deployment time.

