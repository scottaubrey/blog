---
title: "Testing terraform with moto, part 1"
date: 2022-03-11T17:00:00
draft: false
---

For my first 10% day at my new employer, I decided to experiment with testing of our Terraform infrastructure-as-code for our Kubernetes clusters that I'm the maintainer for.

The current setup utilises a [home-grown tool](https://github.com/elifesciences/builder/) that works with AWS CloudFormation and Terraform. The tool is primarily built around CloudFormation, and expresses infrastructure as projects - the encompass many similar "stacks" in CloudFormation terminology, but they represent different environments of the same services in builder. It adds some conventions, defaults for all projects, and further inherited configuration for different environments, creating a consitency across applications with a relatively small amount of configuration. The tool has Terraform capabilities to extend projects that use non-AWS services (thus is not supported by CloudFormation).

Despite eLife using AWS's EKS service for kubernetes, the bootstrapping is built out entirely upon the terraform generation part of builder. There are tests, but they amount to checking that the expected terraform configuration is generated based on changes to the project yaml config file. I was hoping to extend testing to include to a mock AWS service of some description - allowing us to test the full effect of changes (as best we can without running directly on AWS) from generating changes to existing infrastructure during plan, to applying  a verifying it is in a correct state.

I considered two projects that fit the bill for an "emulated" AWS service - [Localstack](https://github.com/localstack/localstack) and [Moto](https://github.com/spulec/moto). Moto comes out of the boto3 project's desire for testing their AWS API client, and Localstack is built on top of Moto. It can be considered a big brother of Moto - extended the APIs to not just mock services, but ***run*** services. For my purposes, I decided to start with just Moto - primarily to start small, and becuase I assume I can move to localstack if and when I extend this to validate an actual cluster is created.

So, with our scope set, I set off on my adventure.

## Getting started

First thing was to get Moto up and running. I used the docker container available [here](https://hub.docker.com/r/motoserver/moto/tags), and just used docker locally to run this container:

```shell-session
> docker run --rm  --name kubernetes-cluster-provisioning-test -p 5000:5000 motoserver/moto:latest
```

To test this mocked AWS server, I created an [AWS CLI named profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) with the [aws-plugin-endpoint](https://github.com/wbingli/awscli-plugin-endpoint). Following the instructions at that git repo, on my mac I ended up with this in my `~/.aws/config`:

```ini
[plugins]
endpoint = awscli_plugin_endpoint
cli_legacy_plugin_path = "/opt/homebrew/lib/python3.9/site-packages"


[local]
region = us-east-1
output = json


[profile local]
eks =
    endpoint_url = http://localhost:5000/
apigateway =
    endpoint_url = http://localhost:5000/
kinesis =
    endpoint_url = http://localhost:5000/
dynamodb =
    endpoint_url = http://localhost:5000/
s3 =
    endpoint_url = http://localhost:5000/
firehose =
    endpoint_url = http://localhost:5000/
lambda =
    endpoint_url = http://localhost:5000/
sns =
    endpoint_url = http://localhost:5000/
sqs =
    endpoint_url = http://localhost:5000/
redshift =
    endpoint_url = http://localhost:5000/
elasticsearch =
    endpoint_url = http://localhost:5000/
ses =
    endpoint_url = http://localhost:5000/
route53 =
    endpoint_url = http://localhost:5000/
cloudformation =
    endpoint_url = http://localhost:5000/
cloudwatch =
    endpoint_url = http://localhost:5000/
ssm =
    endpoint_url = http://localhost:5000/
secretsmanager =
    endpoint_url = http://localhost:5000/
stepfunctions =
    endpoint_url = http://localhost:5000/
eventbridge =
    endpoint_url = http://localhost:5000/
sts =
    endpoint_url = http://localhost:5000/
iam =
    endpoint_url = http://localhost:5000/
ec2 =
    endpoint_url = http://localhost:5000/
```

I also added the test authentication to `~/.aws/credentials`:

```ini
[local]
aws_access_key_id = test
aws_secret_access_key = test
```

With both those in place, I was able to run a few different aws-cli commands against the Mocked AWS:

```shell-session
> aws --region  us-east-1 ec2 describe-images --filters Name=name,Values=amazon-eks-node-*
{
    "Images": [
        {
            "Architecture": "x86_64",
            "CreationDate": "2022-03-13T20:01:33.000Z",
            "ImageId": "ami-ekslinux",
            "ImageLocation": "amazon/amazon-eks",
            "ImageType": "machine",
            "Public": true,
            "KernelId": "None",
            "OwnerId": "801119661308",
            "Platform": "Linux/UNIX",
            "RamdiskId": "ari-1a2b3c4d",
            "State": "available",
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/sda1",
                    "Ebs": {
                        "DeleteOnTermination": false,
                        "SnapshotId": "snap-87e311c4",
                        "VolumeSize": 15,
                        "VolumeType": "standard"
                    }
                }
            ],
            "Description": "EKS Kubernetes Worker AMI with AmazonLinux2 image",
            "Hypervisor": "xen",
            "ImageOwnerAlias": "amazon",
            "Name": "amazon-eks-node-linux",
            "RootDeviceName": "/dev/sda1",
            "RootDeviceType": "ebs",
            "Tags": [],
            "VirtualizationType": "hvm"
        }
    ]
}
> echo test > test.txt

> aws s3 mb s3://testbucket
make_bucket: testbucket

> aws s3 cp ./test.txt s3://testbucket/
upload: ./test.txt to s3://testbucket/test

> aws s3 ls testbucket
2022-03-11 16:24:53          5 test.txt
```

## Configure Terraform

To connect Terraform to the Moto instance, I used this provider config in a file called `test_moto.tf` new directory:

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
```

I've just added endpoints to the services I expect to use, but there is a whole list of other endpoints that can be overridden for terraform [here](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints).

At this point you can run `terraform init` to get the aws module:

```shell-session
> terraform init

Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v4.4.0...
- Installed hashicorp/aws v4.4.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Finally, to test that it was working correctly, I added some data providers to read the same values as above with the aws-cli tool:

```terraform
# get test.txt
data "aws_s3_object" "test_file" {
  bucket = "testbucket"
  key    = "test.txt"
}

# get the ami's filters by amazon-eks-node*
data "aws_ami" "test_ami" {
  filter {
    values = ["amazon-eks-node*"]
    name   = "name"
  }

  most_recent = true
  owners      = ["amazon"]
}

# output during plan/apply
output "bucket_file_body" {
  value = data.aws_s3_object.test_file.body
}
output "aws_ami_architecture" {
  value = data.aws_ami.test_ami.architecture
}
```

If all is well, when you run `terraform plan`, your output should looks something like this:

```shell-session
> terraform plan

Changes to Outputs:
  + aws_ami_architecture = "x86_64"
  + bucket_file_body     = <<-EOT
        test
    EOT

You can apply this plan to save these new output values to the Terraform state, without changing any real infrastructure.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.
```

That's it for this post, next time We'll attempt to connect up real terraform config, and see how much we far we can provision.

\- Scott
