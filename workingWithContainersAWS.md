## Introduction
This codelab provides a guide on key concepts to be aware of when working with containers in AWS cloud.
Before engaging in this guide, you should have foundational understanding of the mechanics related to containerization.

## Core Commands
This section outlines the core docker commands for building an image, tagging, and pushing the image to AWS ECR repo.

1. Set working variables.
```
export AWS_ACCOUNT_ID=
export AWS_REGION=
export REPOSITORY_NAME=
export REPOSITORY_IMAGE_TAG=
```

2. `cd` to the container directory and use `docker build` command to build the image.
Use `-t` option to assign the intended image repo and tag.
```
docker build -t ${REPOSITORY_NAME}:{REPOSITORY_IMAGE_TAG} .
```

3. Use the `docker tag` command to map the local image to the intended ECR repository.
```
docker tag ${REPOSITORY_NAME}:{REPOSITORY_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPOSITORY_NAME}:{REPOSITORY_IMAGE_TAG}
```

4. Obtain login password for the ECR service and pass this to the `docker login` command.
```
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

5. Now use `docker push` command to push the local image to ECR repository.
```
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPOSITORY_NAME}:{REPOSITORY_IMAGE_TAG}
```

## Proxy Service
If the container build process includes installation of packages from the public internet then variables for the proxy must be set within the Dockerfile.
To set these variables only for the build process use `ARG`,
or use `ENV` if the proxy settings should persist to deployed containers.

Instead of hard-coding the variables,
they should instead be parameterized and passed in via the docker `--build-arg` argument,
which will allow for dynamic settings between deployment and development environments.
```
docker build -t ${REPOSITORY_NAME}:{REPOSITORY_IMAGE_TAG} . --build-arg HTTP_PROXY=${HTTP_PROXY} --build-arg HTTPS_PROXY=${HTTPS_PROXY} --build-arg NO_PROXY=${NO_PROXY}
```

Ensure to set the `ARG/ENV` arguments within the Dockerfile as otherwise this setting will not be applied:
```
ARG    HTTP_PROXY=${HTTP_PROXY}
ARG    HTTPS_PROXY=${HTTPS_PROXY}
ARG    NO_PROXY=${NO_PROXY}
```

## CodeBuild Utilities

## Immutable Tags
A best practice when working with containers is to enable immutable tags,
which ensures that each image added to the registry must be assigned a unique tag.
This approach prevents using a static tag such as `latest` to target the image used for the live deployment.
This improves traceability and facilitates the process of reverting to a previous stable image version if a disruption event occurs.

The unique tag to apply to images should provide a straightforward means of tracing the ECR image back to the event in source control that triggered the deployment (e.g. commit, PR).
One option for achieving this is to use the ID of the pipeline execution that built the image, which will typically have associated metadata for the artifacts brought in from source control.

Alternatively the commit ID can also be used for the image tag,
however a unique value (e.g. timestamp) should also be appended to allow for image rebuilds using the same commit.

### Jenkins Example
For jenkins the executionID can be accessed within the pipeline vai the `BUILD_ID` environment variable.
This ID is incremental rather than a UUID, so you should also append a unique value to account for a potential reset in build history:
```
pipeline {
    agent any
    
    environment {
        def TIMESTAMP = sh(script: "echo `date +%s`", returnStdout: true).trim()
        def ECR_IMAGE_TAG = "BUILD_ID-$TIMESTAMP"
    }
    
    stages {
        stage("echo-tag") {
            steps {
                echo "ecr-image-tag: $ECR_IMAGE_TAG"
            }
        }
    }
}
```

### CodePipeline Example
The CodePipeline executionID is accessible within the pipeline by using [AWS CodePipeline Varaibles](https://docs.aws.amazon.com/codepipeline/latest/userguide/reference-variables.html),
which can be used to dynamically pass CodePipeline metadata to action integrations.
For example, to pass the executionID (referenced via `#{codepipeline.PipelineExecutionId}`) to a Codebuild project for building containers,
add this variable to the required environment variables:
```
EnvironmentVariables = jsonencode([
    #...other stuff
    { name = "DESTINATION_TAG", value = "#{codepipeline.PipelineExecutionId}", type = "PLAINTEXT" }
])
```

The executionID can then be passed to later actions which require the image tag,
e.g. a Terraform deployment action with a variable for the image tag and ECR repository.

## ECR Environment Management (dev/prd)
The same image that was built and tested in `dev` environment should be used in the `prd` environment,
as rebuilding once testing is complete will create an untested artefact which could have vulnerabilities or breaking changes.

A straightforward means of achieving this is to define a policy for the ECR repository in the `dev` account which permits cross-account access from resources in the `prd` account.
```
variable "prd_account_id" {
    description = "AWS Account ID of `prd` account"
    type        = string
}

# pre-requisite: a data file should already be present outside the script 
data "aws_iam_policy_document" "ecr_cross_account" {
    statement {
        effect = "Allow"
        actions = [
            "ecr:GetDownloadUrlForLayer",
            "ecr:BatchGetImage",
            "ecr:BatchCheckLayerAvailability",
        ]
        principals {
            type         = "AWS"
            indentifiers = ["arn:aws:iam::${var.prd_account_id}:root"] 
        }
    }
}

module "ecr_repository" {
    #...other stuff
    additional_policy_document = [data.aws_iam_policy_document.ecr_cross_account.json]
}
```

Alternatively you can maintain a separate ECR repository in the `prd` account and replicate the `dev` image during the pipeline release.
This provides isolation from disruption events in the `dev` account (e.g. images being deleted) and increased security controls over `prd` images.
Use the CodeBuild Utilites module as resource.

## ECR Vulnerability Scanning
ECR provides out-of-the-box functionality for vulnerability scanning of container images.
After an image has been uploaded to ECR it will be scanned and a report will be returned with a list of detected vulnerabilities and their associated criticality level.

### ECR Scan module

### SDK Commands
This example uses boto3 SDK for ECR scan management.
To obtain scan results and validate no `CRITICAL` findings are present:
```
import boto3

ecr = boto3.client("ecr")

scan_findings = ecr.describe_image_scan_findings(
    repositoryName="<repo-name>"
    imageId={
        "imageTag": "<image-tag>"
    }
)
assert scan_findings["imageScanStatus"] == "COMPLETE"
for findings in scan_findings["imageScanFindings"]["findings"]:
    assert findings["severity"] not in ["CRITICAL", "HIGH"]
```

to trigger an image rescan:
```
response = ecr.start_image_scan(
    repositoryName="<repo-name>",
    imageId={
        "imageTag": "<image-tag>"
    }
)
assert response["imageScanStatus"]["status"] in ["IN_PROGRESS", "PENDING"]
```
Note: there is a hard-limit of one rescan for each ECR image per 24hr period.

## Secrets Management
ECR offers a native integration with AWS Secrets Manager so that secrets can be provisioned into containers as environment variables during init.

To achieve this a policy needs to be assigned to the ECS Task IAM Role which permits Secrets read access,
and a mapping of Secrets ARNs to the environment variables needs to be specified within the `Secrets` block of an AWS ECS Task Definition.

