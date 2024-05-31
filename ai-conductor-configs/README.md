# Kubeflow for AI Conductor

Overlay manifests of Kubeflow for AI Conductor

Directories are pointed by `tests/e2e/resources/installation_config/ai-conductor.yaml`

## How to use

### Install Kubeflow

Modify `tests/e2e/utils/kubeflow_installation.py` file to use INSTALLTION_CONFIG file to use `ai-conductor.yaml` (for now)
* Additional script for AI Conductor will be implemented

### Expose Kubeflow services

TBA

### Install AI Conductor

Set envorionemnt variables needed for installation

```
export CLUSTER_NAME=REPLACE_WITH_EKS_CLUSTER_NAME
export CLUSTER_REGION=REPLACE_WITH_AWS_REGION_CODE (ex. ap-northeast-2)

export S3_SECRET_NAME="<S3 secret name stored on Secrets Manager>"
export MINIO_AWS_ACCESS_KEY_ID="<your s3 user access key>"
export MINIO_AWS_SECRET_ACCESS_KEY="<your s3 user secret  key>"
export MINIO_SERVICE_HOST=s3.amazonaws.com

export RDS_SECRET_NAME="<your rds secret name>"
export DB_HOST="<your rds db host>"
export DB_USERNAME="<your rds db username>"
export DB_PASSWORD="<your rds db password>"
export DB_PORT="<yout rds db port>"
export MLMD_DB=metadb
```

Store S3 secret information if needed

```bash
aws secretsmanager create-secret --name $S3_SECRET --secret-string '{"accesskey":"'$MINIO_AWS_ACCESS_KEY_ID'","secretkey":"'$MINIO_AWS_SECRET_ACCESS_KEY'"}' --region $CLUSTER_REGION
```

Change S3 configuration

```bash
yq e -i '.spec.parameters.objects |= sub("s3-secret",env(S3_SECRET_NAME))' awsconfigs/common/aws-secrets-manager/s3/secret-provider.yaml

printf '
bucketName='$S3_BUCKET_NAME'
minioServiceHost='$MINIO_SERVICE_HOST'
minioServiceRegion='$CLUSTER_REGION'
' > awsconfigs/apps/pipeline/s3/params.env
```

Store RDS secret information if needed

```bash
aws secretsmanager create-secret --name $RDS_SECRET_NAME --secret-string '{"username":"'$DB_USERNAME'","password":"'$DB_PASSWORD'","database":"kubeflow","host":"'$DB_HOST'","port":"'$DB_PORT'"}' --region $CLUSTER_REGION
```

Change RDS secret provider with yq

```bash
yq e -i '.spec.parameters.objects |= sub("rds-secret",env(RDS_SECRET_NAME))' awsconfigs/common/aws-secrets-manager/rds/secret-provider.yaml

printf '
dbHost='$DB_HOST'
mlmdDb='$MLMD_DB'
' > awsconfigs/apps/pipeline/rds/params.env

printf '
dbPort=$DB_PORT
' > ai-conductor-configs/apps/pipeline-static/params.env
```

[Install Secrets Store CSI Driver for AWS](https://awslabs.github.io/kubeflow-manifests/docs/deployment/rds-s3/guide/#install-csi-driver-and-update-kfp-configurations)

* Create sevice account with role
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:ListSecrets",
                "secretsmanager:GetSecretValue",
                "ssm:GetParameters",
                "ssm:GetParameter"
            ],
            "Resource": [
                "arn:aws:ssm:$AWS_REGION:$AWS_ACCOUNT_ID:parameter/aws/reference/secretsmanager/*",
                "arn:aws:secretsmanager:$AWS_REGION:$AWS_ACCOUNT_ID:secret:*"
            ]
        }
    ]
  }
  ```

Deploy Kubeflow with custom script

```bash
make -f Makefile.aiconductor deploy-ai-conductor INSTALLATION_OPTION=kustomize DEPLOYMENT_OPTION=rds-s3 PIPELINE_S3_CREDENTIAL_OPTION=static
```

## Difference from upstream

* Disabled AWS Telemetry
* Disabled ACK SageMaker Controller
* Applied Node Selector to all workloads (Deployments, StatefulSets)
* Disabled database schema/table modification (including alterations)
  * Applied `skip_db_migration` flag on `ml-metadata` 
  * Used custom image that disables schema/table modification

### Custom Image Reference

| Component Name | GitLab URL | Image URL |
| - | - | - |
| ml-pipeline/apiserver | [smartdata/acp/pipelines/2.0.3-ai-conductor](http://mod.lge.com/hub/smartdata/acp/pipelines/-/tree/2.0.0-alpha.7-ai-conductor?ref_type=heads) | 123456789012.dkr.ecr.region.amazonaws.com/ecr-repo-regionAlias-infraName-deployEnv/ml-pipeline/api-server:2.0.3 |
| ml-pipeline/cache-server | [smartdata/acp/pipelines/2.0.0-ai-conductor](http://mod.lge.com/hub/smartdata/acp/pipelines/-/tree/2.0.0-rc.2-ai-conductor?ref_type=heads) | 123456789012.dkr.ecr.region.amazonaws.com/ecr-repo-regionAlias-infraName-deployEnv/ml-pipeline/cache-server:2.0.0 |
| kubeflowkatib/katib-db-manager | [smartdata/acp/katib/v0.15.0-ai-conductor](http://mod.lge.com/hub/smartdata/acp/katib/-/commits/v0.15.0-ai-conductor) | 123456789012.dkr.ecr.region.amazonaws.com/ecr-repo-regionAlias-infraName-deployEnv/kubeflowkatib/katib-db-manager:v0.15.0 |
