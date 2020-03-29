---
description: This page describes the K8S deployment for the LOGIQ using HELM 3 charts
---

# K8S Quickstart guide

## 1 - Prerequisites

LOGIQ K8S components are made available as helm charts. Instructions below assume you are using HELM 3.

### 1.1 Add LOGIQ helm repository

```bash
$ helm repo add logiq-repo https://logiqai.github.io/helm-charts
```

{% hint style="info" %}
The HELM repository will be named `logiq-repo`. For installing charts from this repository please make sure to use the repository name as the prefix e.g. 

`helm install <deployment_name> logiq-repo/<chart_name>`
{% endhint %}

You can now run `helm search repo logiq-repo` to see the available helm charts

```bash
$ helm search repo logiq-repo
NAME                      	CHART VERSION	APP VERSION	DESCRIPTION
logiq-repo/flash-brew-helm	1.0.0        	1.2.0      	LOGIQ Middleware Helm chart
logiq-repo/flash-coffee   	1.0.0        	1.2.0      	LOGIQ UI Helm chart
logiq-repo/flash-discovery	1.0.0        	1.0.0      	LOGIQ discovery server
logiq-repo/logiq-flash    	1.0.0        	1.2.0      	LOGIQ ingest server
```

### 1.2 Add additional helm repositories

```bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com 
$ helm repo add bitnami https://charts.bitnami.com/bitnami 
```

### 1.3 Run update to fetch latest charts

```bash
$ helm repo update
```

### 1.4 Create namespace where LOGIQ will be deployed

```bash
$ kubectl create namespace logiq
```

This will create a namespace _**logiq**_ where we will deploy the LOGIQ Log Insights stack. 

{% hint style="info" %}
If you choose a different name for the namespace, please remember to use the same namespace for the remainder of the steps
{% endhint %}

### 1.5 Get storage class for your K8S cluster

Get the storage class for your cluster. The command examples below from [step 2 onwards](k8s-quickstart-guide.md#2-install-dependencies) section use the _**standard**_ storage class

{% hint style="info" %}
If your cloud has a different storage class for persistent volumes, replace _**standard**_ below with an appropriate storage class name when invoking helm install for the charts

```bash
persistence.storageClass=standard
```
{% endhint %}

## 2 - Install dependencies

We are next going to install dependencies in the namespace _**logiq**_ created above. Please use an appropriate storage class from your K8S cluster and change the parameters.

```bash
$ helm install redis --namespace logiq --set usePassword=false \
--set cluster.slaveCount=2 bitnami/redis

$ helm install postgres --namespace logiq --set fullnameOverride=postgres \
--set postgresqlPassword=postgres,persistence.storageClass=standard \
--set resources.requests.memory=2Gi --set resources.requests.cpu=1000m \
bitnami/postgresql 
```

## 3 - Configuring your S3 bucket

Depending on your requirements, you may want to host your storage in your own K8S cluster or create a bucket in a cloud provider like AWS.

### 3.1 Use a bucket in AWS S3

#### 3.1.1 Create an access/secret key pair for creating and managing your bucket <a id="3-1-1"></a>

Go to AWS IAM console and create an access key and secret key that can be used to create your bucket and manage access to the bucket for writing and reading your log files

#### 3.1.2 Deploy the S3 gateway to create and access your bucket

Make sure to pass your AWS\_ACCESS\_KEY\_ID and AWS\_SECRET\_ACCESS\_KEY from [step 3.1.1](k8s-quickstart-guide.md#3-1-1) above and give a bucket name. The S3 gateway acts as a caching gateway and helps reduce API cost.

{% hint style="info" %}
You do not need to create the bucket, we will automatically provision it for you. Just provide the bucket name and access credentials in the the step below. 

If the bucket already exists, LOGIQ will use it. Check to make sure the access and secret key work with it.
{% endhint %}

```bash
$ helm install s3-gateway --namespace logiq --set fullnameOverride=s3-gateway \
--set s3gateway.enabled=true \
--set environment.AWS_ACCESS_KEY_ID=<access_key> \
--set environment.AWS_SECRET_ACCESS_KEY=<secret_key> \
--set defaultBucket.enabled=true,defaultBucket.name=<bucket_name> \
--set accessKey=logiq_access,secretKey=logiq_secret \
--set persistence.enabled=true \
--set persistence.storageClass=standard,persistence.size=2Gi stable/minio
```

{% hint style="info" %}
S3 providers may have restrictions on bucket name for e.g. AWS S3 bucket names are globally unique. 
{% endhint %}

### 3.2 Run MinIO S3 compatible store in your K8S cluster

{% hint style="info" %}
Skip this step if you are using the bucket on AWS and completed step [3.1](k8s-quickstart-guide.md#3-1-use-a-bucket-in-aws-s3) above
{% endhint %}

```bash
$ helm install s3-gateway --namespace logiq --set fullnameOverride=s3-gateway \
 --set accessKey=logiq_access,secretKey=logiq_secret \
 --set defaultBucket.enabled=true,defaultBucket.name=logiq \
 --set persistence.enabled=true,persistence.storageClass=standard \
 --set persistence.size=100Gi stable/minio
```

## 4 - Run LOGIQ server

LOGIQ server provides Ingest, log tailing, data indexing, query and search capabilities. You can use the [logiqbox LOGIQ CLI](https://docs.logiq.ai/logiq-cli) for accessing the above features.

### 4.1 - Install LOGIQ discovery server \( aka disco \)

```bash
$ helm install logiq-discovery --namespace logiq \
--set fullnameOverride=logiq-discovery \
--set persistence.storageClassName=standard logiq-repo/flash-discovery
```

| HELM chart option | Description |
| :--- | :--- |
| **namespace** | K8S namespace for the deployment |
| **persistence.storageClassName** | K8S storage class |

### 4.2 - Install LOGIQ server config

The LOGIQ server needs a configuration file for ingest, parsing, routing and storing data. More details on the configuration can be found in the [LOGIQ Server Configuration](https://docs.logiq.ai/logiq-server-configuration) Section.

An example configuration file for the setup detailed here is provided at the below url. A K8S config map for  the same is also provided.

{% hint style="info" %}
Please make sure to edit the configMap to include the correct bucket name e.g. if .you decided to use bucket name `my-awesome-bucket`,the configMap would be modified as follows

```text
   destinations:
      -

        name: default_log_store

        partition: p_scheme

        s3:

          bucket: my-awesome-bucket 
```
{% endhint %}

Let us now download the configMap

{% tabs %}
{% tab title="LOGIQ server configuration : K8S ConfigMap" %}
{% embed url="https://logiqcf.s3.amazonaws.com/configs/configMap.yaml" caption="LOGIQ Configuration K8S ConfigMap" %}
{% endtab %}
{% endtabs %}

Download the K8S _ConfigMap_ and store it to a file e.g. `configMap.yaml`. You can now apply the config map as follows. This will install a config map named `logiq-config`

```bash
$ kubectl --namespace logiq apply -f configMap.yaml
```

### 4.3 - Install LOGIQ server certificates and Client CA `[OPTIONAL]`

LOGIQ supports TLS for all ingest. We also enable non-TLS ports by default. It is however recommended that  non-TLS ports not be used unless running in a secure VPC or cluster. The certificates can be provided to the cluster using K8S secrets. Replace the template sections below with your Base64 encoded secret files.

{% hint style="info" %}
If you skip this step, LOGIQ server automatically generates a ca and a pair of client and server certificates for you to use. you can get them from the ingest server pods under the folder `/flash/certs`
{% endhint %}

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

### 4.4 - Install ingest server \(aka flash\)

Use the discovery server name from [step 4.1](k8s-quickstart-guide.md#4-1-install-discovery-server) above. Each ingest server pod also provisions 120Mi of log storage.

{% hint style="info" %}
If you skipped [4.3](k8s-quickstart-guide.md#4-3-install-logiq-server-certificates-and-client-ca-optional) above and did not install a secret, exclude `--set secrets_name` when you run the below command
{% endhint %}

```bash
$ helm install logiq-server --namespace logiq \
--set persistence.storageClassName=standard \
--set discoveryService.name=logiq-discovery \
--set configMapName=logiq-config \
--set secrets_name=logiq-certs logiq-repo/logiq-flash
```

| HELM chart option | Description | Default |
| :--- | :--- | :--- |
| **configMapName** | LOGIQ server configuration | logiq-config |
| **secrets\_name** | LOGIQ server certificates | logiq-certs |
| **namespace** | K8S namespace for the deployment |  |
| **persistence.storageClassName** | K8S storage class |  |
| **persistence.pod\_storage** | POD storage used by LOGIQ | 10Gi |
| **discoveryService** | LOGIQ discovery server DNS |  |
| **replicaCount** | Number of ingest replica nodes to run | 3 |
| **redis\_host** | Redis server DNS | redis-master |
| **redis\_port** | Redis server port | 6379 |
| **postgres\_host** | Postgres server DNS | postgres |
| **postgres\_port** | Postgres server port | 5432 |
| **postgres\_user** | Postgres user | postgres |
| **postgres\_password** | Postgres password | postgres |

## 5 - Run LOGIQ UI

### 5.1 - Install Brew

```bash
$ helm install flash-brew --namespace logiq \
--set environment.s3_url=http://s3-gateway:9000 --set environment.s3_bucket=logiq \
--set environment.s3_access=logiq_access --set environment.s3_secret=logiq_secret \
logiq-repo/flash-brew-helm
```

{% hint style="info" %}
Please make sure that the s3\_bucket matches the bucket name used in [step 3](k8s-quickstart-guide.md#3-configuring-your-s3-bucket)
{% endhint %}

| HELM Chart option | Description | Default |
| :--- | :--- | :--- |
| **environment.s3\_url** | S3 gateway URL | http://s3-gateway:9000 |
| **environment.s3\_bucket** | S3 bucket name | logiq |
| **environment.s3\_access** | S3 gateway Login _\( this is not the access for the AWS S3 bucket \)_ | logiq\_access |
| **environment.s3\_secret** | S3 gateway Password _\( this is not the secret for the AWS S3 bucket \)_ | logiq\_secret |
| **environment.postgres\_host** | Postgres server DNS | postgres |
| **environment.postgres\_port** | Postgres server port | 5432 |
| **environment.postgres\_user** | Postgres user | postgres |
| **environment.postgres\_password** | Postgres password | postgres |

### 5.2 - Install Coffee

```bash
$ helm install flash-coffee --namespace logiq logiq-repo/flash-coffee
```

| HELM Chart option | Description | Default |
| :--- | :--- | :--- |
| **environment.redis\_host** | Redis server DNS | redis-master |
| **environment.redis\_port** | Redis server port | 6379 |
| **environment.postgres\_host** | Postgres server DNS | postgres |
| **environment.postgres\_port** | Postgres server port | 5432 |
| **environment.postgres\_user** | Postgres user | postgres |
| **environment.postgres\_password** | Postgres password | postgres |
| **coffee\_worker.replicas** | Number of UI replicas | 4 |

## 6 - Secure the installation Install ingress controller and rules

The final step is installing an ingress controller for the UI so all access is via HTTPS only. Please use the below ingress definition for LOGIQ front end. The annotations will depend on your specific ingress controller. Below we have an example with using the popular [traefik ingress controller](https://docs.traefik.io/)

### 6.1 Install the server certificate

We will now install a server certificate to secure our traffic via https. You will need to create a server certificate key pair first for the domain you want the UI exposed for e.g. lets assume you are hosting the UI on `logiq.my-domain.com`and lets assume your certificate files are  `server-cert.pem` and `server-cert.key`

```bash
kubectl create secret tls ui-secret --namespace logiq \
--key server-key.pem --cert server.pem
```

### 6.2 Install traefik ingress controller

```bash
$ kubectl create clusterrolebinding --user system:serviceaccount:kube-system:default \
kube-system-cluster-admin --clusterrole cluster-admin
$ helm install traefik-ingress-logiq -n kube-system \
--set kubernetes.ingressClass="traefik-logiq" \
--set ssl.enabled=true,ssl.enforced=true stable/traefik
```

### 6.3 Install LOGIQ UI Ingress rules

{% hint style="info" %}
NOTE that the `kubernetes.io/ingress.class` annotation is used by the ingress controller to bind the service in the namespace. We pass thee `kubernetes.ingressClass` to traefik when we create the ingress controller. This allows multiple ingress controller to co-exist
{% endhint %}

```bash
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ui-ingress
  annotations:
    ingress.kubernetes.io/proxy-body-size: 100M
    kubernetes.io/ingress.class: "traefik-logiq"
    ingress.kubernetes.io/app-root: "/"
spec:
  tls:
  - hosts:
    - "logiq.my-domain.com"
    secretName: ui-secret
  rules:
  - host: "logiq.my-domain.com"
    http:
      paths:
      - path: /
        backend:
          serviceName: coffee
          servicePort: 80
      - path: /v1
        backend:
          serviceName: logiq-flash
          servicePort: 9999
```

Save this YAML and update with the appropriate `host domain` and `kubernetes.io/ingress.class`. You can then apply the ingress yaml to your namespace e.g. logiq in this case. Lets call this yaml `logiq-ingress.yaml` 

```bash
$ kubectl -n logiq apply -f logiq-ingress.yaml
```

You should now be able to login to LOGIQ UI at your domain using `https://logiq.my-domain.com` that you set in the ingress after you have updated your DNS server to point to the Ingress controller service IP.

![](../.gitbook/assets/screen-shot-2020-03-24-at-3.42.55-pm.png)

