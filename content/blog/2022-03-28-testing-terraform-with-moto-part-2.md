---
title: "Testing terraform with moto, part 2 (An Interlude)"
date: 2022-03-28T15:00:00
draft: false
---

In (my last post)[/blog/2022-03-11-testing-terraform-with-moto-part-1/] I started to set up the terraform and moto environment. In this post, we'll attempt to connect an existing module up to moto server.

I had previously (exported and converted the terraform output of builder)[https://github.com/elifesciences/kubernetes-cluster-provisioning] to standard terraform HCL, and decided to build upon that work to:

1) Give the scenario a real-world setup
2) Save myself time trying to write it from scratch.

There is a branch called [`refactor-into-a-module`](https://github.com/elifesciences/kubernetes-cluster-provisioning/tree/refactor-into-a-module) That I used as the base version. Then I used the basic config in our last post as a test terraform module utilising the module. The code in that repo is, unfortunately, stuck on terraform 0.11, so I needed to update quite a bit of it to bring it up to date to latest terraform coding pracices. In the end, we had something like this

```terraform
// setup provider for localstack
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  s3_use_path_style           = true
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    ec2 = "http://localhost:5000"
    eks = "http://localhost:5000"
    iam = "http://localhost:5000"
    s3  = "http://localhost:5000"
  }
}

module "kubernetes-aws" {
  source            = "../../../modules/kubernetes-aws"
  cluster_name      = "kubernetes-test"
  cluster_version   = "1.19"
  project           = "terraform-testing"
  environment       = "test"
  min_workers       = "1"
  max_workers       = "6"
  desired_workers   = "3"
  instance_type     = "t3.large"
}
```

I then worked in a `terraform init` and `terraform plan`, trying to correct any issues I could see with the code. In the end, I needed to replace some hardcoded VPCs, a call to AWS for EKS ec2 images (instead, make sure the call returns a moto mocked image), and some small tweaks to changes in the resources available in the `aws` plugin.

In the end, I had something like this:

```terraform
// setup provider for localstack
provider "aws" {
  region  = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  s3_use_path_style           = true
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    ec2            = "http://localhost:5000"
    eks            = "http://localhost:5000"
    iam            = "http://localhost:5000"
  }
}

# query moto to get the default VPC and subnet
data "aws_vpc" "default" {
}
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

module "kubernetes-aws" {
  source            = "../../../modules/kubernetes-aws"
  cluster_name      = "kubernetes-test"
  cluster_version   = "1.19"
  project           = "terraform-testing"
  environment       = "test"
  min_workers       = 1
  max_workers       = 6
  desired_workers   = 3
  instance_type     = "t3.large"

  is_testing        = true                          # added to signal which image filter to look for
  vpc_id            = data.aws_vpc.default.id       # to replace the hardcoded VPC ids
  subnets           = data.aws_subnets.default.ids  # ... and subnets with moto ones
}
```

At this point, in theory the module should apply cleanly to AWS infrastructure. Unfortunately, I ran into an issue with moto:

```
│ Error: error creating EKS Cluster (kubernetes-test): ResourceInUseException: Cluster already exists with name: kubernetes-test
│ {
│   RespMetadata: {
│     StatusCode: 409,
│     RequestID: ""
│   },
│   ClusterName: "kubernetes-test",
│   Message_: "Cluster already exists with name: kubernetes-test"
│ }
│
│   with module.kubernetes-aws.aws_eks_cluster.main,
│   on ../../../modules/kubernetes-aws/main.tf line 206, in resource "aws_eks_cluster" "main":
│  206: resource "aws_eks_cluster" "main" {
│
```

Bringing the module down the bare minimum:

```terraform
// setup provider for localstack
provider "aws" {
    region                      = "us-east-1"
    access_key                  = "test"
    secret_key                  = "test"
    s3_use_path_style           = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true

    endpoints {
        ec2 = "http://localhost:5000"
        eks = "http://localhost:5000"
        iam = "http://localhost:5000"
        s3  = "http://localhost:5000"
    }
}


# get the default VPC out of moto
data "aws_vpc" "default" {
}
data "aws_subnets" "default" {
    filter {
        name   = "vpc-id"
        values = [data.aws_vpc.default.id]
    }
}

#create a basic IAM role
resource "aws_iam_role" "master" {
  name               = "test"
  assume_role_policy = "{\"Version\": \"2012-10-17\", \"Statement\": [{\"Action\": \"sts:AssumeRole\", \"Effect\": \"Allow\", \"Principal\": {\"Service\": \"eks.amazonaws.com\"}}]}"
}

#create a basic security group
resource "aws_security_group" "master" {
  name   = "test"
  vpc_id = data.aws_vpc.default.id

  egress {
    cidr_blocks = [
      "0.0.0.0/0",
    ]

    from_port = 0
    to_port   = 0
    protocol  = -1
  }

  description = "test"
}

# create a basic eks cluster
resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.aws_subnets.default.ids

    security_group_ids = [
      "${aws_security_group.master.id}",
    ]
  }


  version  = "1.21"
  role_arn = "${aws_iam_role.master.arn}"
  name     = "test"
}
```

and running `terraform apply` with the environment variable `TF_LOG=debug` we can see where things go wrong:

```text
2022-03-28T07:37:38.803+0100 [DEBUG] provider.terraform-provider-aws_v4.5.0_x5: [aws-sdk-go] <lengthy json>: timestamp=2022-03-28T07:37:38.803+0100
2022-03-28T07:37:38.803+0100 [DEBUG] provider.terraform-provider-aws_v4.5.0_x5: [aws-sdk-go] DEBUG: Unmarshal Response eks/CreateCluster failed, attempt 0/25, error SerializationError: failed decoding JSON RPC response
	status code: 200, request id:
caused by: JSON value is not a list (map[string]interface {}{}): timestamp=2022-03-28T07:37:38.803+0100
2022-03-28T07:37:38.862+0100 [DEBUG] provider.terraform-provider-aws_v4.5.0_x5: [aws-sdk-go] DEBUG: Retrying Request eks/CreateCluster, attempt 1: timestamp=2022-03-28T07:37:38.862+0100
```

The try then fails, as the (fake) cluster was already created.

From the error output, it would seem terraform code is expecting something in that json response to be a list, but is finding it as a map.

Sure enough, the [moto codebase](https://github.com/spulec/moto/blob/4e106995af6f2820273528fca8a4e9ee288690a5/moto/eks/models.py#L116) shows that it is returning a dict():

```python
class Cluster:
    def __init__(
        self,
        name,
        role_arn,
        resources_vpc_config,
        region_name,
        aws_partition,
        version=None,
        kubernetes_network_config=None,
        logging=None,
        client_request_token=None,
        tags=None,
        encryption_config=None,
    ):
        if encryption_config is None:
            encryption_config = dict() # THIS LINE
        if tags is None:
            tags = dict()
```

Where as aws's [API documentation says it's an array](https://docs.aws.amazon.com/eks/latest/APIReference/API_Cluster.html#AmazonEKS-Type-Cluster-encryptionConfig) and[`aws-sdk-go`](https://github.com/aws/aws-sdk-go/blob/7f50d8698cdad3dd5e653c426d2874163f832e5c/service/eks/api.go#L4747) expects a list in the response.

I've posted a bug request [here](https://github.com/spulec/moto/issues/4979), hopefully this will be ironed out upstream.

In the next post, I'll explore a temporary fix for this, and then using terratest to set out examples and tests. Until next time.

\- Scott
