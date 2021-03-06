<!-- 














  ** DO NOT EDIT THIS FILE
  ** 
  ** This file was automatically generated by the `build-harness`. 
  ** 1) Make all changes to `README.yaml` 
  ** 2) Run `make init` (you only need to do this once)
  ** 3) Run`make readme` to rebuild this file. 
  **
  ** (We maintain HUNDREDS of open source projects. This is how we maintain our sanity.)
  **















  -->
[![README Header][readme_header_img]][readme_header_link]

[![Cloud Posse][logo]](https://cpco.io/homepage)

# terraform-aws-eks-workers [![Codefresh Build Status](https://g.codefresh.io/api/badges/pipeline/cloudposse/terraform-modules%2Fterraform-aws-eks-workers?type=cf-1)](https://g.codefresh.io/public/accounts/cloudposse/pipelines/5d8c0fe0c9daf686b5858180) [![Latest Release](https://img.shields.io/github/release/cloudposse/terraform-aws-eks-workers.svg)](https://github.com/cloudposse/terraform-aws-eks-workers/releases/latest) [![Slack Community](https://slack.cloudposse.com/badge.svg)](https://slack.cloudposse.com)


Terraform module to provision AWS resources to run EC2 worker nodes for [Elastic Container Service for Kubernetes](https://aws.amazon.com/eks/).

Instantiate it multiple times to create many EKS worker node pools with specific settings such as GPUs, EC2 instance types, or autoscale parameters.


---

This project is part of our comprehensive ["SweetOps"](https://cpco.io/sweetops) approach towards DevOps. 
[<img align="right" title="Share via Email" src="https://docs.cloudposse.com/images/ionicons/ios-email-outline-2.0.1-16x16-999999.svg"/>][share_email]
[<img align="right" title="Share on Google+" src="https://docs.cloudposse.com/images/ionicons/social-googleplus-outline-2.0.1-16x16-999999.svg" />][share_googleplus]
[<img align="right" title="Share on Facebook" src="https://docs.cloudposse.com/images/ionicons/social-facebook-outline-2.0.1-16x16-999999.svg" />][share_facebook]
[<img align="right" title="Share on Reddit" src="https://docs.cloudposse.com/images/ionicons/social-reddit-outline-2.0.1-16x16-999999.svg" />][share_reddit]
[<img align="right" title="Share on LinkedIn" src="https://docs.cloudposse.com/images/ionicons/social-linkedin-outline-2.0.1-16x16-999999.svg" />][share_linkedin]
[<img align="right" title="Share on Twitter" src="https://docs.cloudposse.com/images/ionicons/social-twitter-outline-2.0.1-16x16-999999.svg" />][share_twitter]


[![Terraform Open Source Modules](https://docs.cloudposse.com/images/terraform-open-source-modules.svg)][terraform_modules]



It's 100% Open Source and licensed under the [APACHE2](LICENSE).







We literally have [*hundreds of terraform modules*][terraform_modules] that are Open Source and well-maintained. Check them out! 






## Introduction

The module provisions the following resources:

- IAM Role and Instance Profile to allow Kubernetes nodes to access other AWS services
- Security Group with rules for EKS workers to allow networking traffic
- AutoScaling Group with Launch Template to configure and launch worker instances
- AutoScaling Policies and CloudWatch Metric Alarms to monitor CPU utilization on the EC2 instances and scale the number of instance in the AutoScaling Group up or down.
If you don't want to use the provided functionality, or want to provide your own policies, disable it by setting the variable `autoscaling_policies_enabled` to `"false"`.

## Usage


**IMPORTANT:** The `master` branch is used in `source` just as an example. In your code, do not pin to `master` because there may be breaking changes between releases.
Instead pin to the release tag (e.g. `?ref=tags/x.y.z`) of one of our [latest releases](https://github.com/cloudposse/terraform-aws-eks-workers/releases).



For a complete example, see [examples/complete](examples/complete)

```hcl
  provider "aws" {
    region = var.region
  }

  locals {
    # The usage of the specific kubernetes.io/cluster/* resource tags below are required
    # for EKS and Kubernetes to discover and manage networking resources
    # https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html#base-vpc-networking
    tags = merge(var.tags, map("kubernetes.io/cluster/${var.cluster_name}", "shared"))
  }

  module "vpc" {
    source     = "git::https://github.com/cloudposse/terraform-aws-vpc.git?ref=tags/0.8.0"
    namespace  = var.namespace
    stage      = var.stage
    name       = var.name
    cidr_block = "172.16.0.0/16"
    tags       = local.tags
  }

  module "subnets" {
    source               = "git::https://github.com/cloudposse/terraform-aws-dynamic-subnets.git?ref=tags/0.16.0"
    availability_zones   = var.availability_zones
    namespace            = var.namespace
    stage                = var.stage
    name                 = var.name
    vpc_id               = module.vpc.vpc_id
    igw_id               = module.vpc.igw_id
    cidr_block           = module.vpc.vpc_cidr_block
    nat_gateway_enabled  = false
    nat_instance_enabled = false
    tags                 = local.tags
  }

  module "eks_workers" {
    source                             = "git::https://github.com/cloudposse/terraform-aws-eks-workers.git?ref=master"
    namespace                          = var.namespace
    stage                              = var.stage
    name                               = var.name
    instance_type                      = var.instance_type
    vpc_id                             = module.vpc.vpc_id
    subnet_ids                         = module.subnets.public_subnet_ids
    health_check_type                  = var.health_check_type
    min_size                           = var.min_size
    max_size                           = var.max_size
    wait_for_capacity_timeout          = var.wait_for_capacity_timeout
    cluster_name                       = var.cluster_name
    cluster_endpoint                   = var.cluster_endpoint
    cluster_certificate_authority_data = var.cluster_certificate_authority_data
    cluster_security_group_id          = var.cluster_security_group_id

    # Auto-scaling policies and CloudWatch metric alarms
    autoscaling_policies_enabled           = var.autoscaling_policies_enabled
    cpu_utilization_high_threshold_percent = var.cpu_utilization_high_threshold_percent
    cpu_utilization_low_threshold_percent  = var.cpu_utilization_low_threshold_percent
  }
```






## Makefile Targets
```
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| additional_security_group_ids | Additional list of security groups that will be attached to the autoscaling group | list(string) | `<list>` | no |
| after_cluster_joining_userdata | Additional commands to execute on each worker node after joining the EKS cluster (after executing the `bootstrap.sh` script). For mot info, see https://kubedex.com/90-days-of-aws-eks-in-production | string | `` | no |
| allowed_cidr_blocks | List of CIDR blocks to be allowed to connect to the worker nodes | list(string) | `<list>` | no |
| allowed_security_groups | List of Security Group IDs to be allowed to connect to the worker nodes | list(string) | `<list>` | no |
| associate_public_ip_address | Associate a public IP address with an instance in a VPC | bool | `false` | no |
| attributes | Additional attributes (e.g. `1`) | list(string) | `<list>` | no |
| autoscaling_policies_enabled | Whether to create `aws_autoscaling_policy` and `aws_cloudwatch_metric_alarm` resources to control Auto Scaling | bool | `true` | no |
| aws_iam_instance_profile_name | The name of the existing instance profile that will be used in autoscaling group for EKS workers. If empty will create a new instance profile. | string | `` | no |
| before_cluster_joining_userdata | Additional commands to execute on each worker node before joining the EKS cluster (before executing the `bootstrap.sh` script). For mot info, see https://kubedex.com/90-days-of-aws-eks-in-production | string | `` | no |
| block_device_mappings | Specify volumes to attach to the instance besides the volumes specified by the AMI | object | `<list>` | no |
| bootstrap_extra_args | Passed to the `bootstrap.sh` script to enable `--kublet-extra-args` or `--use-max-pods` | string | `` | no |
| cluster_certificate_authority_data | The base64 encoded certificate data required to communicate with the cluster | string | - | yes |
| cluster_endpoint | EKS cluster endpoint | string | - | yes |
| cluster_name | The name of the EKS cluster | string | - | yes |
| cluster_security_group_id | Security Group ID of the EKS cluster | string | - | yes |
| cluster_security_group_ingress_enabled | Whether to enable the EKS cluster Security Group as ingress to workers Security Group | bool | `true` | no |
| cpu_utilization_high_evaluation_periods | The number of periods over which data is compared to the specified threshold | number | `2` | no |
| cpu_utilization_high_period_seconds | The period in seconds over which the specified statistic is applied | number | `300` | no |
| cpu_utilization_high_statistic | The statistic to apply to the alarm's associated metric. Either of the following is supported: `SampleCount`, `Average`, `Sum`, `Minimum`, `Maximum` | string | `Average` | no |
| cpu_utilization_high_threshold_percent | The value against which the specified statistic is compared | number | `90` | no |
| cpu_utilization_low_evaluation_periods | The number of periods over which data is compared to the specified threshold | number | `2` | no |
| cpu_utilization_low_period_seconds | The period in seconds over which the specified statistic is applied | number | `300` | no |
| cpu_utilization_low_statistic | The statistic to apply to the alarm's associated metric. Either of the following is supported: `SampleCount`, `Average`, `Sum`, `Minimum`, `Maximum` | string | `Average` | no |
| cpu_utilization_low_threshold_percent | The value against which the specified statistic is compared | number | `10` | no |
| credit_specification | Customize the credit specification of the instances | object | `null` | no |
| default_cooldown | The amount of time, in seconds, after a scaling activity completes before another scaling activity can start | number | `300` | no |
| delimiter | Delimiter to be used between `namespace`, `stage`, `name` and `attributes` | string | `-` | no |
| disable_api_termination | If `true`, enables EC2 Instance Termination Protection | bool | `false` | no |
| ebs_optimized | If true, the launched EC2 instance will be EBS-optimized | bool | `false` | no |
| eks_worker_ami_name_filter | AMI name filter to lookup the most recent EKS AMI if `image_id` is not provided | string | `amazon-eks-node-*` | no |
| eks_worker_ami_name_regex | A regex string to apply to the AMI list returned by AWS | string | `^amazon-eks-node-[1-9,\.]+-v\d{8}$` | no |
| elastic_gpu_specifications | Specifications of Elastic GPU to attach to the instances | object | `null` | no |
| enable_monitoring | Enable/disable detailed monitoring | bool | `true` | no |
| enabled | Whether to create the resources. Set to `false` to prevent the module from creating any resources | bool | `true` | no |
| enabled_metrics | A list of metrics to collect. The allowed values are `GroupMinSize`, `GroupMaxSize`, `GroupDesiredCapacity`, `GroupInServiceInstances`, `GroupPendingInstances`, `GroupStandbyInstances`, `GroupTerminatingInstances`, `GroupTotalInstances` | list(string) | `<list>` | no |
| force_delete | Allows deleting the autoscaling group without waiting for all instances in the pool to terminate. You can force an autoscaling group to delete even if it's in the process of scaling a resource. Normally, Terraform drains all the instances before deleting the group. This bypasses that behavior and potentially leaves resources dangling | bool | `false` | no |
| health_check_grace_period | Time (in seconds) after instance comes into service before checking health | number | `300` | no |
| health_check_type | Controls how health checking is done. Valid values are `EC2` or `ELB` | string | `EC2` | no |
| image_id | EC2 image ID to launch. If not provided, the module will lookup the most recent EKS AMI. See https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html for more details on EKS-optimized images | string | `` | no |
| instance_initiated_shutdown_behavior | Shutdown behavior for the instances. Can be `stop` or `terminate` | string | `terminate` | no |
| instance_market_options | The market (purchasing) option for the instances | object | `null` | no |
| instance_type | Instance type to launch | string | - | yes |
| key_name | SSH key name that should be used for the instance | string | `` | no |
| load_balancers | A list of elastic load balancer names to add to the autoscaling group. Only valid for classic load balancers. For ALBs, use `target_group_arns` instead | list(string) | `<list>` | no |
| max_size | The maximum size of the autoscale group | number | - | yes |
| metrics_granularity | The granularity to associate with the metrics to collect. The only valid value is 1Minute | string | `1Minute` | no |
| min_elb_capacity | Setting this causes Terraform to wait for this number of instances to show up healthy in the ELB only on creation. Updates will not wait on ELB instance number changes | number | `0` | no |
| min_size | The minimum size of the autoscale group | number | - | yes |
| mixed_instances_policy | policy to used mixed group of on demand/spot of differing types. Launch template is automatically generated. https://www.terraform.io/docs/providers/aws/r/autoscaling_group.html#mixed_instances_policy-1 | object | `null` | no |
| name | Solution name, e.g. 'app' or 'cluster' | string | `app` | no |
| namespace | Namespace, which could be your organization name, e.g. 'eg' or 'cp' | string | `` | no |
| placement | The placement specifications of the instances | object | `null` | no |
| placement_group | The name of the placement group into which you'll launch your instances, if any | string | `` | no |
| protect_from_scale_in | Allows setting instance protection. The autoscaling group will not select instances with this setting for terminination during scale in events | bool | `false` | no |
| scale_down_adjustment_type | Specifies whether the adjustment is an absolute number or a percentage of the current capacity. Valid values are `ChangeInCapacity`, `ExactCapacity` and `PercentChangeInCapacity` | string | `ChangeInCapacity` | no |
| scale_down_cooldown_seconds | The amount of time, in seconds, after a scaling activity completes and before the next scaling activity can start | number | `300` | no |
| scale_down_policy_type | The scalling policy type, either `SimpleScaling`, `StepScaling` or `TargetTrackingScaling` | string | `SimpleScaling` | no |
| scale_down_scaling_adjustment | The number of instances by which to scale. `scale_down_scaling_adjustment` determines the interpretation of this number (e.g. as an absolute number or as a percentage of the existing Auto Scaling group size). A positive increment adds to the current capacity and a negative value removes from the current capacity | number | `-1` | no |
| scale_up_adjustment_type | Specifies whether the adjustment is an absolute number or a percentage of the current capacity. Valid values are `ChangeInCapacity`, `ExactCapacity` and `PercentChangeInCapacity` | string | `ChangeInCapacity` | no |
| scale_up_cooldown_seconds | The amount of time, in seconds, after a scaling activity completes and before the next scaling activity can start | number | `300` | no |
| scale_up_policy_type | The scalling policy type, either `SimpleScaling`, `StepScaling` or `TargetTrackingScaling` | string | `SimpleScaling` | no |
| scale_up_scaling_adjustment | The number of instances by which to scale. `scale_up_adjustment_type` determines the interpretation of this number (e.g. as an absolute number or as a percentage of the existing Auto Scaling group size). A positive increment adds to the current capacity and a negative value removes from the current capacity | number | `1` | no |
| service_linked_role_arn | The ARN of the service-linked role that the ASG will use to call other AWS services | string | `` | no |
| stage | Stage, e.g. 'prod', 'staging', 'dev', or 'test' | string | `` | no |
| subnet_ids | A list of subnet IDs to launch resources in | list(string) | - | yes |
| suspended_processes | A list of processes to suspend for the AutoScaling Group. The allowed values are `Launch`, `Terminate`, `HealthCheck`, `ReplaceUnhealthy`, `AZRebalance`, `AlarmNotification`, `ScheduledActions`, `AddToLoadBalancer`. Note that if you suspend either the `Launch` or `Terminate` process types, it can prevent your autoscaling group from functioning properly. | list(string) | `<list>` | no |
| tags | Additional tags (e.g. `{ BusinessUnit = "XYZ" }` | map(string) | `<map>` | no |
| target_group_arns | A list of aws_alb_target_group ARNs, for use with Application Load Balancing | list(string) | `<list>` | no |
| termination_policies | A list of policies to decide how the instances in the auto scale group should be terminated. The allowed values are `OldestInstance`, `NewestInstance`, `OldestLaunchConfiguration`, `ClosestToNextInstanceHour`, `Default` | list(string) | `<list>` | no |
| use_custom_image_id | If set to `true`, will use variable `image_id` for the EKS workers inside autoscaling group | bool | `false` | no |
| use_existing_aws_iam_instance_profile | If set to `true`, will use variable `aws_iam_instance_profile_name` to run EKS workers using an existing AWS instance profile that was created outside of this module, workaround for error like `count cannot be computed` | bool | `false` | no |
| use_existing_security_group | If set to `true`, will use variable `workers_security_group_id` to run EKS workers using an existing security group that was created outside of this module, workaround for errors like `count cannot be computed` | bool | `false` | no |
| vpc_id | VPC ID for the EKS cluster | string | - | yes |
| wait_for_capacity_timeout | A maximum duration that Terraform should wait for ASG instances to be healthy before timing out. Setting this to '0' causes Terraform to skip all Capacity Waiting behavior | string | `10m` | no |
| wait_for_elb_capacity | Setting this will cause Terraform to wait for exactly this number of healthy instances in all attached load balancers on both create and update operations. Takes precedence over `min_elb_capacity` behavior | number | `0` | no |
| workers_role_policy_arns | List of policy ARNs that will be attached to the workers default role on creation | list(string) | `<list>` | no |
| workers_role_policy_arns_count | Count of policy ARNs that will be attached to the workers default role on creation. Needed to prevent Terraform error `count can't be computed` | number | `0` | no |
| workers_security_group_id | The name of the existing security group that will be used in autoscaling group for EKS workers. If empty, a new security group will be created | string | `` | no |

## Outputs

| Name | Description |
|------|-------------|
| autoscaling_group_arn | ARN of the AutoScaling Group |
| autoscaling_group_default_cooldown | Time between a scaling activity and the succeeding scaling activity |
| autoscaling_group_desired_capacity | The number of Amazon EC2 instances that should be running in the group |
| autoscaling_group_health_check_grace_period | Time after instance comes into service before checking health |
| autoscaling_group_health_check_type | `EC2` or `ELB`. Controls how health checking is done |
| autoscaling_group_id | The AutoScaling Group ID |
| autoscaling_group_max_size | The maximum size of the AutoScaling Group |
| autoscaling_group_min_size | The minimum size of the AutoScaling Group |
| autoscaling_group_name | The AutoScaling Group name |
| launch_template_arn | ARN of the launch template |
| launch_template_id | The ID of the launch template |
| security_group_arn | ARN of the worker nodes Security Group |
| security_group_id | ID of the worker nodes Security Group |
| security_group_name | Name of the worker nodes Security Group |
| workers_role_arn | ARN of the worker nodes IAM role |
| workers_role_name | Name of the worker nodes IAM role |




## Share the Love 

Like this project? Please give it a ★ on [our GitHub](https://github.com/cloudposse/terraform-aws-eks-workers)! (it helps us **a lot**) 

Are you using this project or any of our other projects? Consider [leaving a testimonial][testimonial]. =)


## Related Projects

Check out these related projects.

- [terraform-aws-ec2-autoscale-group](https://github.com/cloudposse/terraform-aws-ec2-autoscale-group) - Terraform module to provision Auto Scaling Group and Launch Template on AWS
- [terraform-aws-ecs-container-definition](https://github.com/cloudposse/terraform-aws-ecs-container-definition) - Terraform module to generate well-formed JSON documents (container definitions) that are passed to the  aws_ecs_task_definition Terraform resource
- [terraform-aws-ecs-alb-service-task](https://github.com/cloudposse/terraform-aws-ecs-alb-service-task) - Terraform module which implements an ECS service which exposes a web service via ALB
- [terraform-aws-ecs-web-app](https://github.com/cloudposse/terraform-aws-ecs-web-app) - Terraform module that implements a web app on ECS and supports autoscaling, CI/CD, monitoring, ALB integration, and much more
- [terraform-aws-ecs-codepipeline](https://github.com/cloudposse/terraform-aws-ecs-codepipeline) - Terraform module for CI/CD with AWS Code Pipeline and Code Build for ECS
- [terraform-aws-ecs-cloudwatch-autoscaling](https://github.com/cloudposse/terraform-aws-ecs-cloudwatch-autoscaling) - Terraform module to autoscale ECS Service based on CloudWatch metrics
- [terraform-aws-ecs-cloudwatch-sns-alarms](https://github.com/cloudposse/terraform-aws-ecs-cloudwatch-sns-alarms) - Terraform module to create CloudWatch Alarms on ECS Service level metrics
- [terraform-aws-ec2-instance](https://github.com/cloudposse/terraform-aws-ec2-instance) - Terraform module for providing a general purpose EC2 instance
- [terraform-aws-ec2-instance-group](https://github.com/cloudposse/terraform-aws-ec2-instance-group) - Terraform module for provisioning multiple general purpose EC2 hosts for stateful applications



## Help

**Got a question?** We got answers. 

File a GitHub [issue](https://github.com/cloudposse/terraform-aws-eks-workers/issues), send us an [email][email] or join our [Slack Community][slack].

[![README Commercial Support][readme_commercial_support_img]][readme_commercial_support_link]

## DevOps Accelerator for Startups


We are a [**DevOps Accelerator**][commercial_support]. We'll help you build your cloud infrastructure from the ground up so you can own it. Then we'll show you how to operate it and stick around for as long as you need us. 

[![Learn More](https://img.shields.io/badge/learn%20more-success.svg?style=for-the-badge)][commercial_support]

Work directly with our team of DevOps experts via email, slack, and video conferencing.

We deliver 10x the value for a fraction of the cost of a full-time engineer. Our track record is not even funny. If you want things done right and you need it done FAST, then we're your best bet.

- **Reference Architecture.** You'll get everything you need from the ground up built using 100% infrastructure as code.
- **Release Engineering.** You'll have end-to-end CI/CD with unlimited staging environments.
- **Site Reliability Engineering.** You'll have total visibility into your apps and microservices.
- **Security Baseline.** You'll have built-in governance with accountability and audit logs for all changes.
- **GitOps.** You'll be able to operate your infrastructure via Pull Requests.
- **Training.** You'll receive hands-on training so your team can operate what we build.
- **Questions.** You'll have a direct line of communication between our teams via a Shared Slack channel.
- **Troubleshooting.** You'll get help to triage when things aren't working.
- **Code Reviews.** You'll receive constructive feedback on Pull Requests.
- **Bug Fixes.** We'll rapidly work with you to fix any bugs in our projects.

## Slack Community

Join our [Open Source Community][slack] on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

## Newsletter

Sign up for [our newsletter][newsletter] that covers everything on our technology radar.  Receive updates on what we're up to on GitHub as well as awesome new projects we discover. 

## Office Hours

[Join us every Wednesday via Zoom][office_hours] for our weekly "Lunch & Learn" sessions. It's **FREE** for everyone! 

[![zoom](https://img.cloudposse.com/fit-in/200x200/https://cloudposse.com/wp-content/uploads/2019/08/Powered-by-Zoom.png")][office_hours]

## Contributing

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/cloudposse/terraform-aws-eks-workers/issues) to report any bugs or file feature requests.

### Developing

If you are interested in being a contributor and want to get involved in developing this project or [help out](https://cpco.io/help-out) with our other projects, we would love to hear from you! Shoot us an [email][email].

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Commit** changes to your own branch
 4. **Push** your work back up to your fork
 5. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!


## Copyright

Copyright © 2017-2020 [Cloud Posse, LLC](https://cpco.io/copyright)



## License 

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) 

See [LICENSE](LICENSE) for full details.

    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

      https://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.









## Trademarks

All other trademarks referenced herein are the property of their respective owners.

## About

This project is maintained and funded by [Cloud Posse, LLC][website]. Like it? Please let us know by [leaving a testimonial][testimonial]!

[![Cloud Posse][logo]][website]

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We ❤️  [Open Source Software][we_love_open_source].

We offer [paid support][commercial_support] on all of our projects.  

Check out [our other projects][github], [follow us on twitter][twitter], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.



### Contributors

|  [![Erik Osterman][osterman_avatar]][osterman_homepage]<br/>[Erik Osterman][osterman_homepage] | [![Andriy Knysh][aknysh_avatar]][aknysh_homepage]<br/>[Andriy Knysh][aknysh_homepage] | [![Igor Rodionov][goruha_avatar]][goruha_homepage]<br/>[Igor Rodionov][goruha_homepage] |
|---|---|---|

  [osterman_homepage]: https://github.com/osterman
  [osterman_avatar]: https://img.cloudposse.com/150x150/https://github.com/osterman.png
  [aknysh_homepage]: https://github.com/aknysh
  [aknysh_avatar]: https://img.cloudposse.com/150x150/https://github.com/aknysh.png
  [goruha_homepage]: https://github.com/goruha
  [goruha_avatar]: https://img.cloudposse.com/150x150/https://github.com/goruha.png

[![README Footer][readme_footer_img]][readme_footer_link]
[![Beacon][beacon]][website]

  [logo]: https://cloudposse.com/logo-300x69.svg
  [docs]: https://cpco.io/docs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=docs
  [website]: https://cpco.io/homepage?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=website
  [github]: https://cpco.io/github?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=github
  [jobs]: https://cpco.io/jobs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=jobs
  [hire]: https://cpco.io/hire?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=hire
  [slack]: https://cpco.io/slack?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=slack
  [linkedin]: https://cpco.io/linkedin?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=linkedin
  [twitter]: https://cpco.io/twitter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=twitter
  [testimonial]: https://cpco.io/leave-testimonial?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=testimonial
  [office_hours]: https://cloudposse.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=office_hours
  [newsletter]: https://cpco.io/newsletter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=newsletter
  [email]: https://cpco.io/email?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=email
  [commercial_support]: https://cpco.io/commercial-support?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=commercial_support
  [we_love_open_source]: https://cpco.io/we-love-open-source?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=we_love_open_source
  [terraform_modules]: https://cpco.io/terraform-modules?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=terraform_modules
  [readme_header_img]: https://cloudposse.com/readme/header/img
  [readme_header_link]: https://cloudposse.com/readme/header/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=readme_header_link
  [readme_footer_img]: https://cloudposse.com/readme/footer/img
  [readme_footer_link]: https://cloudposse.com/readme/footer/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=readme_footer_link
  [readme_commercial_support_img]: https://cloudposse.com/readme/commercial-support/img
  [readme_commercial_support_link]: https://cloudposse.com/readme/commercial-support/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-aws-eks-workers&utm_content=readme_commercial_support_link
  [share_twitter]: https://twitter.com/intent/tweet/?text=terraform-aws-eks-workers&url=https://github.com/cloudposse/terraform-aws-eks-workers
  [share_linkedin]: https://www.linkedin.com/shareArticle?mini=true&title=terraform-aws-eks-workers&url=https://github.com/cloudposse/terraform-aws-eks-workers
  [share_reddit]: https://reddit.com/submit/?url=https://github.com/cloudposse/terraform-aws-eks-workers
  [share_facebook]: https://facebook.com/sharer/sharer.php?u=https://github.com/cloudposse/terraform-aws-eks-workers
  [share_googleplus]: https://plus.google.com/share?url=https://github.com/cloudposse/terraform-aws-eks-workers
  [share_email]: mailto:?subject=terraform-aws-eks-workers&body=https://github.com/cloudposse/terraform-aws-eks-workers
  [beacon]: https://ga-beacon.cloudposse.com/UA-76589703-4/cloudposse/terraform-aws-eks-workers?pixel&cs=github&cm=readme&an=terraform-aws-eks-workers
