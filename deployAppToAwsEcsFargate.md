## Introduction
This guide will help you create a minimal app deployment on AWS ECS fargate.
This deployment is flexible for web apps, APIs and 3rd party apps like Nginx.

## Prerequisites
The following must be met before engaging:
* A base understanding of terraform (state management, modularization)
* A base understanding of AWS cloud
* AWS sandbox environment (created with terraform module)
* Pre-prepared container image hosted on ECR
* The following tools installed locally:
  * Terraform
  * AWS CLI
  * Git

## InnerSource Modules
Use the necessary modules (e.g. tf-aws-codebuild-container-utilities) for deploying solution components.
The Terraform module `source` and version`1.0.1` reference would be as follows:
```
module "tf-aws-codebuild-container-utilities" {
    source = "https://REPO_URL/location/of/module.git?ref=1.0.1"
    #...other stuff
}
```

using versions references will help you be coherent with terraform configuration,
however, ensure to upgrade to the latest versions

## Base Setup
### Terraform Setup
Review Terraform best-practices on how to set up a Terraform directory structure, state resources, and variables.
```
module "tf-aws-codebuild-container-utilities" {
    source        = "https://REPO_URL/location/of/module.git?ref=1.0.1"
    app_id        = var.app_id
    chargeback_id = var.chargeback_id
    product_name  = var.product_name
    workspace     = var.workspace
    tags = {}.
```

### ECS Container Image
You should have a pre-prepared container image hosted on ECR, that we can use for the ECS fargate service.
If you don't have one, then you can use Nginx from DockerHub

Once our container image is ready then we add the following variables and define defaults for the ECR repo URL and image tag:
```
variable "container_config" {
    description = "Config objects that defines the setting for the container image to pull from ECR"
    default     = {
        ecr_repository_url = "<value>"
        ecr_image_tag      = "<value>"
    }
    type        = object({
        ecr_repository_url = string
        ecr_image_tag      = string
    })
}
```

## ECS Cluster
An ECS cluster doesn't incur cost by itself, but by the resources deployed to it and consuming RAM/vCPU.
The cluster can support up to 1000 parallel Fargate tasks,
and multiple clusters can be deployed per account.

However, each Fargate task will consume 1 private IP from the VPC CIDR.

We will use the ECS cluster module to create the cluster, which also provides the following foundational resources to support ECS deployment:
* IAM role for Fargate task execution
* Log group for task runtime logs
* (optional) Service Discovery for inter-service communication
* (optional) Cloudwatch alarm for task monitoring

```
module "ecs_cluster" {
    source        = "https://REPO_URL/location/of/ecs_cluster_module.git?ref=1.0.1"
    app_id        = var.app_id
    chargeback_id = var.chargeback_id
    product_name  = var.product_name
    workspace     = module.context.workspace

    alarm_settings = {
      enable_alert = false
    }
    
    name = var.app_name
}
```

### IAM Role Custom Permissions
The ECS cluster module creates a default policy for the Fargate task IAM role that grants the required permissions for Fargate task execution (i.e. pulling from ECR, writing to Cloudwatch logs).
To provide custom permissions to the Fargate role, such as read/write access for s3 buckets, pass in a defined policy via the module `role_policy` argument.
```
data "aws_iam_policy_document" "example" {
  #...policy
}

module "ecs_cluster" {
  #...other stuff
  role_policy = data.aws_iam_policy_document.example.json
}
```

### Monitoring
The module we are using creates cloudwatch alarms which monitor Fargate task rotation in the cluster.
e.g. during a failed health check, or task initialization issue.

To enable the alerts, provide the following configurations to the module definition and adjust the `task_failure_alerting_config` variable for alarm customization (e.g. set a threshold of 5 tasks rotating within a 5-minute period before alerting)
```
data "aws_sns_topic" "alarms" {
  name = "<topic-name>"
}

module "ecs_cluster" {
  #...other stuff
  alarm_settings = {
    enable_alert      = true
    action_topic_name = data.aws_sns_topic.alarms.name
  }
  
  # notification if the task rotates 5 times over a 5-minute period
  task_failure_alerting_config = {
    evaluaution_periods     = 1
    log_record_retention    = 14
    notification_period     = 300
    notification_threshold  = 5
  }
}
```

## Application Load Balancer
This is the ALB component of the architecture, which will serve as entry point for the Fargate service traffic.
It will provide the following capabilities:
* Load balancing
* TLS termination
* Application health checks
* Custom DNS name

### Security Group
Before we create the ALB, we must first define a security group (SG) that controls the application ingress(receiving traffic) and load balancer target egress(transmitting traffic).
In this example, 443(HTTPS) ingress to our network, and 80(HTTP) egress to the ECS Security Group (we will have to create the ECS_SG).
```
module "alb_sg" {
  source        = "https://REPO_URL/location/of/security_group_module.git?ref=1.0.1"
  app_id        = var.app_id
  chargeback_id = var.chargeback_id
  product_name  = var.product_name
  workspace     = module.context.workspace
  
  name = "${var.app_name}-alb"
  rules = [
    {
      type        = "ingress"
      description = "HTTPS"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"] # modify based on requirements
    }
  ]
}

# seperate resources to prevent module cycle errors
resource "aws_security_group_rule" "alb_ecs_egress" {
  type                      = "egress"
  from_port                 = 80
  to_port                   = 80
  protocol                  = "tcp"
  security_group_id         = module.alb_sg.id
  source_security_group_id  = module.ecs_service_sg.id
}
```

### Certificate
To enable HTTPS, we must upload a TLS certificate from a certificate authority to AWS ACM,
and map it to the relevant load balancer listener.

Once we have our certificate in ACM , we can then perform a lookup using the domain to prepare for ALB creation (use the context module `account_private_domain` output)
```
data "aws_acm_certificate" "cert" {
  domain      = "*.${module.context.account_private_domain}"
  statuses    = ["ISSUED"]
}
```

### ALB
Use ALB module to create Load Balancer, which have the following properties:
* HTTPS listener for port 443
* HTTP target group for target type `ip` (required type for Fargate task targets)
* A mapped certificate from ACM for HTTPS
* A health check for Load balancer Targets which will run every 30 seconds and expect a 200 response from the `/` path
* Compatibility with AWS CodeDeploy Blue/Green deployment (the `allow_blue_green` parameter will create a listener which ignores changes to the target group configuration as this will be managed outside of Terraform)
```
module "alb" {
  source      = "https://REPO_URL/location/of/load_balancer_module.git?ref=1.0.1"
  app_id        = var.app_id
  chargeback_id = var.chargeback_id
  product_name  = var.product_name
  workspace     = module.context.workspace
  
  alarm_topic_name = data.aws_sns_topic.alarms.name
  allow_blue_green = true
  name             = var.app_name
  security_groups  = [module.alb_sg.id]  
  
  # protocol and ports
  https_enabled            = true
  log_bucket_force_destroy = true
  port                     = 80
  protocol                 = "HTTP"
  ssl_certificate_arn      = data.aws_acm_certificate.cert.arn
  target_type              = "ip"
  
  # health check
  check_interval      = 30
  matcher             = "200"
  path                = "/"
  unhealthy_threshold = 3
}
```

### Route53 Domain Name
We will define the Route53 CNAME record for the Load Balancer to map a friendly domain name for clients.
Provide a Route53 zone, and use Terraform `data` lookup to locate the zone with the `account_private_domain` output of the context module used as the `name` value:
```
data "aws_route53_zone" "zone" {
  name         = module.context.account_private_domain
  private_zone = true
}
```
Now use the module for route53 records to create a CNAME Record with your custom domain name,
that points to the AWS-generated domain name of the ALB
```
module "route53_cname_record" {
  source = "git::https://REPO_URL/location/of/route53_record.git?ref=1.0.1"
  records = [
    {
      name    = var.app_name
      type    = "CNAME"
      records = [module.alb.dns_name]
      ttl     = 60
    },
  ]
  zone_id = data.aws_route53_zone.zone.id
}
```

### Monitoring
The ALB module has the default CloudWatch alarms for ALB alert scenarios,
such as tracking 400 (and other 4xx) responses over a given period of time.
The CloudWatch Alarms will send notifications to the SNS Topic via the module `alarms_topic_name` variable.

### Stickiness
Stickiness allows to anchor traffic from a specific client to a specific LoadBalancer,
this is useful for applications that maintain state information on connected clients.
Stickiness is managed by returning a cookie to the client that is used to identify the session and relevant backend target.
This cookie can be generated by the LoadBalancer(duration-based) or by the application(application-based).

Stickiness can be set on the ALB Target using the `stickiness` variable and LoadBalancer cookie.
```
module "alb" {
  #...other stuff
  stickiness = {
    enabled = true
    type    = "lb_cookie"
  }
}
```

### Serving Multiple Apps
When working with multiple devs, deploy a dedicated environment to each dev.
To reduce costs, a common pattern is to use the same LoadBalancer across deployment, with multiple listeners and Target Group for dynamically directing traffic.

## ECS Service
This section will focus on deploying ECS service which will deploy and manage Fargate container tasks.

### Task Definition
Before an ECS Service can be created we first need to create an ECS Task Definition which defines the task RAM, CPU, Logging, and container configuration.
The Task Definition is created as part of the ECS Service module, and is customized via the module `container_definitions` parameter.

Copy the following JSON representation for the container definitions and add to a template file within your repo,
e.g. `template/container-definitions.json`:
```
[
  {
    "image": "${ECR_REPOSITORY_URL}:${ECR_IMAGE_TAG}",
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-create-group": "true",
        "awslogs-group": "${LOG_GROUP_NAME}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "name": "${CONTAINER_NAME}",
    "portMappings": [
      {
        "containerPort": 80,
        "hostPort": 80,
        "protocol": "tcp"
      }
    ],
    "essential": true,
    "cpu": 0
  }
]
```
The placeholders (e.g. `ECR_REPOSITORY_URL` and others) will be passed in and rendered when deploying the module.

### Security Group
We also need to create a security group for the ECS Task which enables ingress from the ALB and egress to required endpoints (at minimum VPC endpoints for ECR and S3).
Add the following snippet and include any additional rules you might require
```
module "ecs_service_sg" {
  source = "git::https://REPO_URL/location/of/security_group.git?ref=1.0.1"
  app_id        = var.app_id
  chargeback_id = var.chargeback_id
  product_name  = var.product_name
  workspace     = var.workspace

  name        = "${var.app_name}-ecs"
  description = "ECS security group"
  rules = [
    {
      type                     = "ingress"
      description              = "HTTP ALB"
      protocol                 = "tcp"
      from_port                = 80
      to_port                  = 80
      source_security_group_id = module.alb_sg.id
    },
    {
      type        = "egress"
      description = "HTTPS VPC Endpoints"
      protocol    = "tcp"
      from_port   = 443
      to_port     = 443
      cidr_blocks = data.aws_vpc.vpc.cidr_block_associations.*.cidr_block
    },
    {
      type            = "egress"
      description     = "HTTPS S3"
      protocol        = "tcp"
      from_port       = 443
      to_port         = 443
      prefix_list_ids = [data.aws_prefix_list.s3.id]
    }
  ]
}

data "aws_region" "current" {}
data "aws_prefix_list" "s3" {
  name = "com.amazonaws.${data.aws_region.current.name}.s3"
}
```

### ECS Service
We will now add the module definition for the ECS service. Key Variables to be aware of when adding this definition include:
* `allow_blue_green`: if `true` this will ignore changes to the Service ALB Config. 
Required when using CodeDeploy as this will manage ALB/ECS settings outside of Terraform.
* `deployment_controller`: Defines how the service tasks will be managed. Typically, this will be using CodeDeploy
* `container_definitions`: Notice how we are using the Terraform `templatefile()` function to load our template definitions file and render included variables.
* `desired_count`: The base number of ECS Tasks that will be always running as part of the Service.
* `vpc_config.subnet`: Use this to define the target deployment subnets. Use specified subnet for non-routable subnets.
 ```
module "ecs_service" {
  source = "git::https://REPO_URL/location/of/ecs-service.git?ref=1.0.1"
  app_id        = var.app_id
  chargeback_id = var.chargeback_id
  product_name  = var.product_name
  workspace     = module.context.workspace
  
  allow_blue_green = true
  cluster_arn = module.ecs_cluster.arn
  container_definitions = templatefile(
    {
      LOG_GROUP_NAME      = module.ecs_cluster.runtime_log_group_name
      CONTAINER_NAME      = "container"
      ECR_REPOSITORY_NAME = var.container_config.ecr_repository_url
      ECR_IMAGE_TAG       = var.container_config.ecr_image_tag
    }
  )
  deployment_controller    = "CODE_DEPLOY"
  desired_count            = 1
  ecr_repo_url             = var.container_config.ecr_repository_url
  fargate_platform_version = "1.4.0"
  load_balancer_target = [
    {
      target_group_arn = module.alb.target_arn
      container_name   = "container"
      container_port   = 80
    }
  ]
  log_retention_in_days = 90
  name                  = var.app_name
  vpc_config            = {
    subnet = "specified_private_subnet"
    security_groups = [
      module.ecs_service_sg.id
    ]
  }
}
```

### Auto-Scaling
Auto-scaling can be enabled for the service via the module `autoscaling_policy` parameter. 
The below policy provides an example scaling configuration which tracks aggregates CPU utilization of the overall service and aims to keep this value around the 75% threshold (`target_value`).
After performing a scaling action `_cooldown` parameters specify a min wait of 120 seconds before another scaling action is performed.
```
module "ecs_service" {
  #...other stuff
  autoscaling_policy = {
    "ecs-scale-cpu" = {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
      disable_scale_in       = false
      target_value           = 75
      scale_in_cooldown      = 120
      scale_in_cooldown      = 120
      scale_out_cooldown     = 120
    }
  }
}
```

### Monitoring
The module provides a number of out-of-the-box Cloudwatch alarms which monitor the resource consumption of the Service.
To enable this monitoring simply pass a value for the `alarms_topic_arn` variable, and optionally customize the default alarm settings.
```
module "ecs_service" {
  #...other stuff
  alarm_topic_arn = data.aws_sns_topic.alarms.arn
  ecs_cpu_cloudwatch_configuration = {
    "severe": {
      "evaluation_period": 3,
      "period": 180,
      "threshold": 90,
      "type": "CALLOUT"
    }
  }
}
```

### Service Updates
The module takes an opinionated view that updates to the ECS service will be managed as part of the application release pipeline,
preferably using CodeDeploy service. This means that while Terraform will update the ECS Task Definition during apply, 
the ECS Service will still be pointing to the old ECS Task Definition and will not auto-rotate Tasks.

if you are not using CodeDeploy for pushing updates to the ECS Service then the following AWSCLI command can instead be used which will perform in-place rotation rather than blue/green:
```
aws ecs update-service --cluster <cluster-name> --service <service-name> --force-new-deployment
```

## CODEDEPLOY CONFIGURATION
This section will discuss CodeDeploy component of the application,
which provides blue/green automated deployment capabilities for the Fargate service.
We will be configuring CodeDeploy, key development patterns offered by the service,
and integration with AWS Cloudwatch alarm for custom monitoring.

### Additional Target Group
Before we add the CodeDeploy module, we must first create an additional ALB Target Group for our Fargate Service. 
This Target Group will remain idle until the first CodeDeploy deployment,
at which point it will assume the role of the "green" Target Group serving the updated Fargate Task.
If this deployment is successful then the "green" deployment is now considered as the stable "blue" deployment and the process repeats itself on the next update.

Add the following snippet into your deployment stack, updating any variables as necessary.
```
resource "aws_lb_target_group "green" {
  deregistration_delay          = 60
  load_balancing_algorithm_type = "round_robin"
  name                          = "${module.context.workspace}-${var.app_name}-2"
  port                          = 80
  protocol                      = "HTTP"
  target_type                   = "ip"
  vpc_id                        = data.aws_vpc.vpc.id
  
  health_check {
    enabled = true
    healthy_threshold = 3
    interval = 30
    unhealthy_threshold = 3
    timeout = 10
    protocol = "HTTP"
    path = "/"
    matcher = "200"
  }
  tags = module.context.all_tags
}

data "aws_vpc" "vpc" {
  filter {
    name = "tag:Name"
    values = ["selected-vpc"]
  }
}
```

### Deployment Patterns
When using CodeDeploy with ECS Fargate there are 2 deployment patterns to be aware of:
* Time-Based Canary: A percentage of listener traffic is switched to green deployment for a period of time,
before the remaining traffic is fully switched over (e.g. 10% traffic is switched for 5 minutes, before the remaining 90% is switched)
* Time-Based Linear: A percentage of listener traffic is switched to green deployment based on defined time interval,
until all traffic is switched over (e.g. 10% of traffic is switched every 5 minutes)

AWS provides some default deployment configurations for these patterns that can used with the CodeDeploy module using the `default_deployment_config_name` variable.
Alternatively you can use the `traffic_routing` variable for more granular control over the time period and percentage of traffic to switch:
```
traffic_routing = {
  type = "TimeBasedCanary"
  interval = 5 #minutes before switching all traffic
  percentage = 10 #percent of traffic to initially switch to green
}
```
A further configuration is to add a listener specially for test purpose, and use this for routing traffic to green environment.
This listener can then be exposed to simulated traffic rather than live client traffic, which may be a requirement for certain applications.

### Module
The below snippet provides an example implementation of the innerSource CodeDeploy module following a canary deployment pattern.

Key variables to be aware of include:
* `blue_green.deployment_ready`: The number of minutes after the green environment is ready before switching traffic.
*  `blue_green.blue_fleet`: The number of minutes after a successful deployment before terminating the blue environment.
*  `deployment_style`: The deployment style for CodeDeploy, either blue-green or in-place.
CodeDeploy does not support in-place deployments for ECS, so this variable must be set to `BLUE_GREEN` and `WITH_TRAFFIC_CONTROL`.
*  `load_balancer.target_group_pair`: The ALB configuration for the deployment. If you are using a test listener for green deployments then set this via the `test_route` parameter.
```
module "codedeploy" {
  source = "git::https://REPO_URL/location/of/codedeploy.git?ref=1.0.1"
  app_id        = var.app_id
  chargeback_id = var.chargeback_id
  product_name  = var.product_name
  workspace     = module.context.workspace 

  application_name               = var.app_name
  group_name                     = "${var.app_name}-canary-10-percent-5-minutes"
  compute_platform               = "ECS"
  default_deployment_config_name = "CodeDeployDefault.ECSCanary10Percent5Minutes"
  ecs_service = {
    cluster_name = module.ecs_cluster.name
    service_name = module.ecs_service.service_name
  }
  deployment_style = {
    options = "WITH_TRAFFIC_CONTROL"
    type    = "BLUE_GREEN"
  }
  blue_green = {
    deployment_ready = {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
      wait_time_minutes = 0
    },
    blue_fleet = {
      action            = "TERMINATE"
      wait_time_minutes = 6
    }
  }
  load_balancer = {
    target_group_pair = {
      prod_route    = [module.alb.https_listener_arn]
      target_groups = [module.alb.target_group_name, aws_lb_target_group.green.name]
      test_route    = []
    }
  }
}
```

### Cloudwatch Alarm Monitoring
CodeDeploy can be integrated with CloudWatch to pass or fail a deployment based on the status of CloudWatch Alarms.
An example pattern can include alarms which monitor the 4xx/5xx errors being returned by each ALB Target Group,
and trigger after surpassing a defined threshold. If these alarms do trigger then CodeDeploy will revert all ALB traffic back to the blue environment.

This Alarm monitoring is configured via the `alarms` parameter of the module:
```
alarms = {
  enabled = true
  alarms = [
    aws_cloudwatch_metric_alarm.errors_4xx_tg_a.arn,
    aws_cloudwatch_metric_alarm.errors_5xx_tg_a.arn
  ]
}
```

## CodeDeploy: Deployment
This section outlines the steps for triggering a CodeDeployment.
CodeDeploy can be triggered from any AWS SDK (e.g. AWS CLI, boto3), 
and out-of-the-box integration are provided by common orchestration tools: 
* CodeDeploy Jenkins Plugin
* AWS CodePipeline Action Integration

### AppSpec File
When working with CodeDeploy and ECS Fargate we need to create an Application Specification (AppSpec) file,
which will be uploaded to S3 and referenced when triggering our CodeDeploy deployment (via an AWS SDK or CI/CD management tool).
An AppSpec file is a YAML or JSON document which tells CodeDeploy which Task Definition and containers to target,
and includes optional settings for Lambda deployment hooks and VPC network changes.

A standard pattern is to upload the AppSpec file to S3 as part of the Terraform apply operation, after the updates have been applied to the Task Definition and ECS Service.
This can be achieved by creating a template for the AppSpec in the Terraform directory, then using the `aws_s3_object` resource in combination with the Terraform `templatefile` function to render dynamic variables:
```
resource "aws_s3_object" "appspec" {
  depends_on = [module.ecs_service]
  bucket     = data.aws_s3_bucket.artifacts.id
  key        = "example/key/appspec.yaml"
  content    = templatefile(
    "${path.module}/template/appspec.yml",
    {
      TASK_DEFINITION_ARN = module.ecs_service.task_definition_arn
      CONTAINER_NAME      = "container"
      CONTAINER_PORT      = 80
    }
  )
}
```

### Deployment (AWS SDK - AWS CLI)
This section will use the AWS CLI to manage a CodeDeployment deployment,
however, the same commands are available from other AWS SDKs (e.g. Python's boto3).
This code can be deployed from any CI/CD management tool with networking connectivity to AWS APIs
1. Set values for the following variables:
* `CODEDEPLOY_APP_NAME`: The name of the CodeDeploy application
* `CODEDEPLOY_DEPLOYMENT_GROUP_NAME`: The name of the CodeDeploy Deploy Group
* `ARTIFACTS_BUCKET_ID`: The name of the S3 Bucket where the AppSpec is stored
* `APPSPEC_OBJECT_KEY`: The key of the AppSpec S3 Object

2. Trigger the deployment and obtain the deployment ID:
```
DEPLOYMENT_ID=`aws deploy create-deployment \
  --application-name ${CODEDEPLOY_APP_NAME} \
  --deployment-group-name ${CODEDEPLOY_DEPLOYMENT_GROUP_NAME} \
  --revision revisionType=S3,s3Location={bucket=${ARTIFACTS_BUCKET_ID},key=${APPSPEC_OBJECT_KEY},bundleType=YAML} \
  --query 'deploymentId' --output text`
```

3. Await completion of the deployment (for async use `aws deploy get-deployment` in a loop):
```
aws deploy wait deployment-sucessful --deployment-id ${DEPLOYMENT_ID}
```

4. Get the final deployment status:
```
DEPLOYMENT_STATUS=`aws deploy get-deployment --${DEPLOYMENT_ID} --query 'deploymentInfo.status' --output text`
echo $DEPLOYMENT_STATUS
```
After step 2 you can visit the AWS console and view the CodeDeploy dashboard to see the deployment

### Deployment (CodePipeline)
When using the native CodeDeploy integration offered by CodePipeline,
we need to pass our AppSpec file via the CodePipeline `input_artifiact` parameter.
We also need to pass in a task definition file as an input artifact,
which can be obtained using the same method for generating the AppSpec file,
or you can run the below CLI command:
```
aws ecs describe-task-definition --task-definition ${TASK_DEFINITION_NAME} --query 'taskDefinition' > /tmp/taskdef.json
```

A common pattern when using CodePipeline is to generate both of these files in a previous deployment action,
and use the `output_artifacts` parameter of this action to make the files available for use later in the pipeline.
Assuming a zip file containing both the AppSpec and task definition is used as the artifact,
the CodePipeline action definition would take the below structure:
```
stage {
  name = "Deploy"
  action {
    run_order        = 1
    name             = "generate-codedeploy-artifact"
    output_artifacts = ["codedeploy_artifact"]
    #...other stuff
  }
  action {
    run_order       = 2
    category        = "Deploy"
    name            = "codedeploy-fargate"
    owner           = "AWS"
    provider        = "CodeDeployToECS"
    input_artifacts = ["codedeploy_artifact"]
    version         = "1"
    configuration = {
      ApplicationName                = "${module.context.prefix}-${var.app_name}"
      DeploymentGroupName            = "${module.context.prefix}-${var.app_name}"
      TaskDefinitionTemplateArtifact = "codedeploy_artifact"
      AppSpecTemplateArtifact        = "codedeploy_artifact"
      AppSpecTemplatePath            = "appspec.yml"
      TaskDefinitionTemplatePath     = "taskdef.json"
    }
  }
}
```
Refer to AWS docs for definition of `action` parameters.
To set up a cross-account CodeDeploy action using CodePipeline (i.e. deploying from a pipeline in `pre-prod` account to an ECS service in `prod` account),
use the tutorial on AWS docs "Create a Pipeline in CodePipeline that uses resources from another AWS account"

### Deployment (Jenkins)
If you are using AWS SDK when deploying from Jenkins, 
then follow the example provided above for AWS CLI
