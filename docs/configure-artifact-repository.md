# Configuring Your Artifact Repository

To run Argo workflows that use artifacts, you must configure and use an artifact
repository. Argo supports any S3 compatible artifact repository such as AWS, GCS
and MinIO. This section shows how to configure the artifact repository.
Subsequent sections will show how to use it.

| Name | Inputs | Outputs | Usage (Feb 2020) |
|---|---|---|---|
| Artifactory | Yes | Yes | 11% |
| Azure Blob | Yes | Yes | - |
| GCS | Yes | Yes | - |
| Git | Yes | No | - |
| HDFS | Yes | Yes | 3% |
| HTTP | Yes | No | 2% |
| OSS | Yes | Yes | - |
| Raw | Yes | No | 5% |
| S3 | Yes | Yes | 86% |

The actual repository used by a workflow is chosen by the following rules:

1. Anything explicitly configured using [Artifact Repository Ref](artifact-repository-ref.md). This is the most flexible, safe, and secure option.
2. From a config map named `artifact-repositories` if it has the `workflows.argoproj.io/default-artifact-repository` annotation in the workflow's namespace.
3. From a workflow controller config-map.

## Configuring MinIO

NOTE: MinIO is already included in the [quick-start manifests](quick-start.md).

```bash
brew install helm # mac, helm 3.x
helm repo add minio https://helm.min.io/ # official minio Helm charts
helm repo update
helm install argo-artifacts minio/minio --set service.type=LoadBalancer --set fullnameOverride=argo-artifacts
```

Login to the MinIO UI using a web browser (port 9000) after obtaining the
external IP using `kubectl`.

```bash
kubectl get service argo-artifacts
```

On Minikube:

```bash
minikube service --url argo-artifacts
```

NOTE: When MinIO is installed via Helm, it generates
credentials, which you will use to login to the UI:
Use the commands shown below to see the credentials

- `AccessKey`: `kubectl get secret argo-artifacts -o jsonpath='{.data.accesskey}' | base64 --decode`
- `SecretKey`: `kubectl get secret argo-artifacts -o jsonpath='{.data.secretkey}' | base64 --decode`

Create a bucket named `my-bucket` from the MinIO UI.

## Configuring AWS S3

Create your bucket and access keys for the bucket. AWS access keys have the same
permissions as the user they are associated with. In particular, you cannot
create access keys with reduced scope. If you want to limit the permissions for
an access key, you will need to create a user with just the permissions you want
to associate with the access key. Otherwise, you can just create an access key
using your existing user account.

```bash
$ export mybucket=bucket249
$ cat > policy.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:PutObject",
            "s3:GetObject",
            "s3:ListBucket"
         ],
         "Resource":"arn:aws:s3:::$mybucket/*"
      }
   ]
}
EOF
$ aws s3 mb s3://$mybucket [--region xxx]
$ aws iam create-user --user-name $mybucket-user
$ aws iam put-user-policy --user-name $mybucket-user --policy-name $mybucket-policy --policy-document file://policy.json
$ aws iam create-access-key --user-name $mybucket-user > access-key.json
```

If you have Artifact Garbage Collection configured, you should also add "s3:DeleteObject" to the list of Actions above.

NOTE: if you want argo to figure out which region your buckets belong in, you
must additionally set the following statement policy. Otherwise, you must
specify a bucket region in your workflow configuration.

```json
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation"
         ],
         "Resource":"arn:aws:s3:::*"
      }
    ...
```

## Configuring GCS (Google Cloud Storage)

Create a bucket from the GCP Console
(<https://console.cloud.google.com/storage/browser>).

There are 2 ways to configure a Google Cloud Storage.

### Through Native GCS APIs

- Create and download a Google Cloud service account key.
- Create a kubernetes secret to store the key.
- Configure `gcs` artifact as following in the yaml.

```yaml
artifacts:
  - name: message
    path: /tmp/message
    gcs:
      bucket: my-bucket-name
      key: path/in/bucket
      # serviceAccountKeySecret is a secret selector.
      # It references the k8s secret named 'my-gcs-credentials'.
      # This secret is expected to have have the key 'serviceAccountKey',
      # containing the base64 encoded credentials
      # to the bucket.
      #
      # If it's running on GKE and Workload Identity is used,
      # serviceAccountKeySecret is not needed.
      serviceAccountKeySecret:
        name: my-gcs-credentials
        key: serviceAccountKey
```

If it's a GKE cluster, and Workload Identity is configured, there's no need to
create the service account key and store it as a Kubernetes secret,
`serviceAccountKeySecret` is also not needed in this case. Please follow the
link to configure Workload Identity
(<https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity>).

### Use S3 APIs

Enable S3 compatible access and create an access key. Note that S3 compatible
access is on a per project rather than per bucket basis.

- Navigate to Storage > Settings
  (<https://console.cloud.google.com/storage/settings>).
- Enable interoperability access if needed.
- Create a new key if needed.
- Configure `s3` artifact as following example.

```yaml
artifacts:
  - name: my-output-artifact
    path: /my-output-artifact
    s3:
      endpoint: storage.googleapis.com
      bucket: my-gcs-bucket-name
      # NOTE that, by default, all output artifacts are automatically tarred and
      # gzipped before saving. So as a best practice, .tgz or .tar.gz
      # should be incorporated into the key name so the resulting file
      # has an accurate file extension.
      key: path/in/bucket/my-output-artifact.tgz
      accessKeySecret:
        name: my-gcs-s3-credentials
        key: accessKey
      secretKeySecret:
        name: my-gcs-s3-credentials
        key: secretKey
```

## Configuring Alibaba Cloud OSS (Object Storage Service)

Create your bucket and access key for the bucket. Suggest to limit the permission
for the access key, you will need to create a user with just the permissions you
want to associate with the access key. Otherwise, you can just create an access key
using your existing user account.

Setup [Alibaba Cloud CLI](https://www.alibabacloud.com/help/en/alibaba-cloud-cli/latest/product-introduction)
and follow the steps to configure the artifact storage for your workflow:

```bash
$ export mybucket=bucket-workflow-artifect
$ export myregion=cn-zhangjiakou
$ # limit permission to read/write the bucket.
$ cat > policy.json <<EOF
{
    "Version": "1",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
              "oss:PutObject",
              "oss:GetObject"
            ],
            "Resource": "acs:oss:*:*:$mybucket/*"
        }
    ]
}
EOF
$ # create bucket.
$ aliyun oss mb oss://$mybucket --region $myregion
$ # show endpoint of bucket.
$ aliyun oss stat oss://$mybucket
$ #create a ram user to access bucket.
$ aliyun ram CreateUser --UserName $mybucket-user
$ # create ram policy with the limit permission.
$ aliyun ram CreatePolicy --PolicyName $mybucket-policy --PolicyDocument "$(cat policy.json)"
$ # attch ram policy to the ram user.
$ aliyun ram AttachPolicyToUser --UserName $mybucket-user --PolicyName $mybucket-policy --PolicyType Custom
$ # create access key and secret key for the ram user.
$ aliyun ram CreateAccessKey --UserName $mybucket-user > access-key.json
$ # create secret in demo namespace, replace demo with your namespace.
$ kubectl create secret generic $mybucket-credentials -n demo\
  --from-literal "accessKey=$(cat access-key.json | jq -r .AccessKey.AccessKeyId)" \
  --from-literal "secretKey=$(cat access-key.json | jq -r .AccessKey.AccessKeySecret)"
$ # create configmap to config default artifact for a namespace.
$ cat > default-artifact-repository.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  # If you want to use this config map by default, name it "artifact-repositories". Otherwise, you can provide a reference to a
  # different config map in `artifactRepositoryRef.configMap`.
  name: artifact-repositories
  annotations:
    # v3.0 and after - if you want to use a specific key, put that key into this annotation.
    workflows.argoproj.io/default-artifact-repository: default-oss-artifact-repository
data:
  default-oss-artifact-repository: |
    oss:
      endpoint: http://oss-cn-zhangjiakou-internal.aliyuncs.com
      bucket: $mybucket
      # accessKeySecret and secretKeySecret are secret selectors.
      # It references the k8s secret named 'bucket-workflow-artifect-credentials'.
      # This secret is expected to have have the keys 'accessKey'
      # and 'secretKey', containing the base64 encoded credentials
      # to the bucket.
      accessKeySecret:
        name: $mybucket-credentials
        key: accessKey
      secretKeySecret:
        name: $mybucket-credentials
        key: secretKey
EOF
# create cm in demo namespace, replace demo with your namespace.
$ k apply -f default-artifact-repository.yaml -n demo
```

You can also set `createBucketIfNotPresent` to `true` to tell the artifact driver to automatically create the OSS bucket if it doesn't exist yet when saving artifacts. Note that you'll need to set additional permission for your OSS account to create new buckets.

## Configuring Azure Blob Storage

Create an Azure Storage account and a container within that account. There are a number of
ways to accomplish this, including the [Azure Portal](https://portal.azure.com) or the
[CLI](https://docs.microsoft.com/en-us/cli/azure/).

1. Retrieve the blob service endpoint for the storage account. For example:

   ```bash
   az storage account show -n mystorageaccountname --query 'primaryEndpoints.blob' -otsv
   ```

2. Retrieve the access key for the storage account. For example:

   ```bash
   az storage account keys list -n mystorageaccountname --query '[0].value' -otsv
   ```

3. Create a kubernetes secret to hold the storage account key. For example:

   ```bash
   kubectl create secret generic my-azure-storage-credentials \
     --from-literal "account-access-key=$(az storage account keys list -n mystorageaccountname --query '[0].value' -otsv)"
   ```

4. Configure `azure` artifact as following in the yaml.

```yaml
artifacts:
  - name: message
    path: /tmp/message
    azure:
      endpoint: https://mystorageaccountname.blob.core.windows.net
      container: my-container-name
      blob: path/in/container
      # accountKeySecret is a secret selector.
      # It references the k8s secret named 'my-azure-storage-credentials'.
      # This secret is expected to have have the key 'account-access-key',
      # containing the base64 encoded credentials to the storage account.
      #
      # If a managed identity has been assigned to the machines running the
      # workflow (e.g., https://docs.microsoft.com/en-us/azure/aks/use-managed-identity)
      # then accountKeySecret is not needed, and useSDKCreds should be
      # set to true instead:
      # useSDKCreds: true
      accountKeySecret:
        name: my-azure-storage-credentials
        key: account-access-key     
```

If `useSDKCreds` is set to `true`, then the `accountKeySecret` value is not
used and authentication with Azure will be attempted using a
[`DefaultAzureCredential`](https://docs.microsoft.com/en-us/azure/developer/go/azure-sdk-authentication)
instead.

## Configure the Default Artifact Repository

In order for Argo to use your artifact repository, you can configure it as the
default repository. Edit the workflow-controller config map with the correct
endpoint and access/secret keys for your repository.

### S3 compatible artifact repository bucket (such as AWS, GCS, MinIO, and Alibaba Cloud OSS)

Use the `endpoint` corresponding to your provider:

- AWS: `s3.amazonaws.com`
- GCS: `storage.googleapis.com`
- MinIO: `my-minio-endpoint.default:9000`
- Alibaba Cloud OSS: `oss-cn-hangzhou-zmf.aliyuncs.com`

The `key` is name of the object in the `bucket` The `accessKeySecret` and
`secretKeySecret` are secret selectors that reference the specified kubernetes
secret. The secret is expected to have the keys `accessKey` and `secretKey`,
containing the `base64` encoded credentials to the bucket.

For AWS, the `accessKeySecret` and `secretKeySecret` correspond to
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` respectively.

EC2 provides a meta-data API via which applications using the AWS SDK may assume
IAM roles associated with the instance. If you are running argo on EC2 and the
instance role allows access to your S3 bucket, you can configure the workflow
step pods to assume the role. To do so, simply omit the `accessKeySecret` and
`secretKeySecret` fields.

For GCS, the `accessKeySecret` and `secretKeySecret` for S3 compatible access
can be obtained from the GCP Console. Note that S3 compatible access is on a per
project rather than per bucket basis.

- Navigate to Storage > Settings
  (<https://console.cloud.google.com/storage/settings>).
- Enable interoperability access if needed.
- Create a new key if needed.

For MinIO, the `accessKeySecret` and `secretKeySecret` naturally correspond the
`AccessKey` and `SecretKey`.

For Alibaba Cloud OSS, the `accessKeySecret` and `secretKeySecret` corresponds to
`accessKeyID` `and accessKeySecret` respectively.

Example:

```bash
$ kubectl edit configmap workflow-controller-configmap -n argo  # assumes argo was installed in the argo namespace
...
data:
  artifactRepository: |
    s3:
      bucket: my-bucket
      keyFormat: prefix/in/bucket     #optional
      endpoint: my-minio-endpoint.default:9000        #AWS => s3.amazonaws.com; GCS => storage.googleapis.com
      insecure: true                  #omit for S3/GCS. Needed when minio runs without TLS
      accessKeySecret:                #omit if accessing via AWS IAM
        name: my-minio-cred
        key: accessKey
      secretKeySecret:                #omit if accessing via AWS IAM
        name: my-minio-cred
        key: secretKey
      useSDKCreds: true               #tells argo to use AWS SDK's default provider chain, enable for things like IRSA support
```

The secrets are retrieved from the namespace you use to run your workflows. Note
that you can specify a `keyFormat`.

### Google Cloud Storage (GCS)

Argo also can use native GCS APIs to access a Google Cloud Storage bucket.

`serviceAccountKeySecret` references to a Kubernetes secret which stores a Google Cloud
service account key to access the bucket.

Example:

```bash
$ kubectl edit configmap workflow-controller-configmap -n argo  # assumes argo was installed in the argo namespace
...
data:
  artifactRepository: |
    gcs:
      bucket: my-bucket
      keyFormat: prefix/in/bucket     #optional, it could reference workflow variables, such as "{{workflow.name}}/{{pod.name}}"
      serviceAccountKeySecret:
        name: my-gcs-credentials
        key: serviceAccountKey
```

## Accessing Non-Default Artifact Repositories

This section shows how to access artifacts from non-default artifact
repositories.

The `endpoint`, `accessKeySecret` and `secretKeySecret` are the same as for
configuring the default artifact repository described previously.

```yaml
  templates:
  - name: artifact-example
    inputs:
      artifacts:
      - name: my-input-artifact
        path: /my-input-artifact
        s3:
          endpoint: s3.amazonaws.com
          bucket: my-aws-bucket-name
          key: path/in/bucket/my-input-artifact.tgz
          accessKeySecret:
            name: my-aws-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-aws-s3-credentials
            key: secretKey
    outputs:
      artifacts:
      - name: my-output-artifact
        path: /my-output-artifact
        s3:
          endpoint: storage.googleapis.com
          bucket: my-gcs-bucket-name
          # NOTE that, by default, all output artifacts are automatically tarred and
          # gzipped before saving. So as a best practice, .tgz or .tar.gz
          # should be incorporated into the key name so the resulting file
          # has an accurate file extension.
          key: path/in/bucket/my-output-artifact.tgz
          accessKeySecret:
            name: my-gcs-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-gcs-s3-credentials
            key: secretKey
          region: my-GCS-storage-bucket-region
    container:
      image: debian:latest
      command: [sh, -c]
      args: ["cp -r /my-input-artifact /my-output-artifact"]
```
