# terraform-aws-cloudwatch-agent [![Build Status](https://travis-ci.org/cloudposse/terraform-aws-cloudwatch-agent.svg?branch=master)](https://travis-ci.org/cloudposse/terraform-aws-cloudwatch-agent) [![Latest Release](https://img.shields.io/github/release/cloudposse/terraform-aws-cloudwatch-agent.svg)](https://github.com/cloudposse/terraform-aws-cloudwatch-agent/releases/latest) [![Slack Community](https://slack.cloudposse.com/badge.svg)](https://slack.cloudposse.com)


Terraform module to install the CloudWatch agent on EC2 instances using `cloud-init`.


---

[![Terraform Open Source Modules](https://docs.cloudposse.com/images/terraform-open-source-modules.svg)][terraform_modules]



It's 100% Open Source and licensed under the [APACHE2](LICENSE).





## Usage


**IMPORTANT:** The `master` branch is used in `source` just as an example. In your code, do not pin to `master` because there may be breaking changes between releases.
Instead pin to the release tag (e.g. `?ref=tags/x.y.z`) of one of our [latest releases](https://github.com/cloudposse/terraform-aws-cloudwatch-agent/releases).



### Example with launch configuration:

```hcl
module "cloudwatch_agent" {
  source = "git::https://github.com/cloudposse/terraform-aws-cloudwatch-agent?ref=master"

  name = "cloudwatch_agent"
}

resource "aws_launch_configuration" "multipart" {
  name_prefix          = "cloudwatch_agent"
  image_id             = "${data.aws_ami.ecs_optimized.id}"
  iam_instance_profile = "${aws_iam_instance_profile.cloudwatch_agent.name}"
  instance_type        = "t2.micro"
  user_data_base64     = "${module.cloudwatch_agent.user_data}"
  security_groups      = ["${aws_security_group.ecs.id}"]
  key_name             = "${var.ssh_key_pair}"

  lifecycle {
    create_before_destroy = true
  }
}

data "aws_ami" "ecs_optimized" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-ecs-hvm-*-x86_64-*"]
  }

  owners = [
    "amazon",
  ]
}
```

### Example with using the iam_policy_document and aws_iam_role:

```hcl
locals {
  application {
    name      = "cloudwatch_agent"
    stage     = "dev"
    namespace = "eg"
  }
}

module "label" {
  source      = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  stage       = "${local.application["stage"]}"
  name        = "${local.application["name"]}"
  namespace   = "${local.application["namespace"]}"
}

module "cloudwatch_agent" {
  source = "git::https://github.com/cloudposse/terraform-aws-cloudwatch-agent?ref=master"

  name      = "${module.label.name}"
  stage     = "${module.label.stage}"
  namespace = "${module.label.namespace}"
}

resource "aws_launch_configuration" "multipart" {
  name_prefix          = "${module.label.name}"
  image_id             = "${data.aws_ami.ecs_optimized.id}"
  iam_instance_profile = "${aws_iam_instance_profile.cloudwatch_agent.name}"
  instance_type        = "t2.micro"
  user_data_base64     = "${module.cloudwatch_agent.user_data}"
  security_groups      = ["${aws_security_group.ecs.id}"]
  key_name             = "${var.ssh_key_pair}"

  lifecycle {
    create_before_destroy = true
  }
}

data "aws_ami" "ecs_optimized" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-ecs-hvm-*-x86_64-*"]
  }

  owners = [
    "amazon",
  ]
}

data "aws_iam_policy_document" "ec2_cloudwatch" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals = {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "ec2" {
  name = "${module.label.id}"

  assume_role_policy = "${data.aws_iam_policy_document.ec2_cloudwatch.json}"

  tags = {
    Name = "${module.label.id}"
  }
}

resource "aws_iam_role_policy" "cloudwatch_agent" {
  name   = "${module.label.id}"
  policy = "${module.cloudwatch_agent.iam_policy_document}"
  role   = "${aws_iam_role.ec2.id}"
}

resource "aws_iam_instance_profile" "cloudwatch_agent" {
  name_prefix = "${module.label.name}"
  role        = "${aws_iam_role.ec2.name}"
}
```

### Example with passing user-data and using the role from the module using advanced metrics configuration:

```hcl
module "cloudwatch_agent" {
  source = "git::https://github.com/cloudposse/terraform-aws-cloudwatch-agent?ref=master"

  name      = "cloudwatch_agent"
  stage     = "dev"
  namespace = "eg"

  metrics_config        = "advanced"
  userdata_part_content = "${data.template_file.cloud-init.rendered}"
}

data "template_file" "cloud-init" {
  template = "${file("${path.module}/cloud-init.yml")}"
}

resource "aws_launch_configuration" "multipart" {
  name_prefix          = "cloudwatch_agent"
  image_id             = "${data.aws_ami.ecs_optimized.id}"
  iam_instance_profile = "${aws_iam_instance_profile.cloudwatch_agent.name}"
  instance_type        = "t2.micro"
  user_data_base64     = "${module.cloudwatch_agent.user_data}"
  security_groups      = ["${aws_security_group.ecs.id}"]
  key_name             = "${var.ssh_key_pair}"

  lifecycle {
    create_before_destroy = true
  }
}

data "aws_ami" "ecs_optimized" {
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-ecs-hvm-*-x86_64-*"]
  }

  owners = [
    "amazon",
  ]
}

resource "aws_iam_instance_profile" "cloudwatch_agent" {
  name_prefix = "cloudwatch_agent"
  role        = "${module.cloudwatch_agent.role_name}"
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
| aggregation_dimensions | Specifies the dimensions that collected metrics are to be aggregated on. | list | `<list>` | no |
| attributes | Add a suffix to the resource names. | list | `<list>` | no |
| cpu_resources | Specifies that per-cpu metrics are to be collected. The only allowed value is *. If you include this field and value, per-cpu metrics are collected. | string | `"resources": ["*"],` | no |
| disk_resources | Specifies an array of disk mount points. This field limits CloudWatch to collect metrics from only the listed mount points. You can specify * as the value to collect metrics from all mount points. Defaults to the root / mountpount. | list | `<list>` | no |
| metrics_collection_interval | Specifies how often to collect the cpu metrics, overriding the global metrics_collection_interval specified in the agent section of the configuration file. If you set this value below 60 seconds, each metric is collected as a high-resolution metric. | string | `60` | no |
| metrics_config | "Which metrics should we send to cloudwatch, the default is standard. Setting this variable to advanced will send all the available metrics that are provided by the agent.   You can find more information here https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-cloudwatch-agent-configuration-file-wizard.html." | string | `standard` | no |
| name | Solution name, e.g. 'app'. | string | - | yes |
| namespace | Namespace, which could be your organization name or abbreviation, e.g. 'eg' or 'cp'. | string | - | yes |
| stage | Stage, e.g. 'prod', 'staging', 'dev', or 'test'. | string | `` | no |
| userdata_part_content | The user data that should be passed along from the caller of the module. | string | `` | no |
| userdata_part_content_type | What format is userdata_part_content in - eg 'text/cloud-config' or 'text/x-shellscript'. | string | `text/cloud-config` | no |
| userdata_part_merge_type | Control how cloud-init merges user-data sections. | string | `list(append)+dict(recurse_array)+str()` | no |

## Outputs

| Name | Description |
|------|-------------|
| iam_policy_document | The IAM policy document that can be attached to a role policy |
| role_name | The name of the created IAM role that can be assumed by the instance |
| user_data | The user_data with the cloudwatch_agent configuration in base64 and gzipped |

## Copyright

Copyright © 2017-2019 [Cloud Posse, LLC](https://cpco.io/copyright)

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

