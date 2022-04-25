---
title: "Testing terraform with moto, part 3 - terratest"
date: 2022-04-25T16:50:00
draft: false
---

In [my last post](/blog/2022-03-11-testing-terraform-with-moto-part-2/) I ran into a roadblock related to the moto project, and it's API responses.

In the interim, the moto bblommers fixed [my issue](https://github.com/spulec/moto/issues/4979), and so the latest release of moto works just fine.

In this post we will explore the basics of testing out our module with terratest. What is it?

> *"Terratest is a Go library that provides patterns and helper functions for testing infrastructure, with 1st-class support for Terraform"*
>
>  -- <cite>[Terratest homepage](https://terratest.gruntwork.io/)</cite>

Think of it as a pre-defined set of helpers, build on go's testing framework, designed to enable repetitive automation and give assertions entirely focused on infrastructure automation. It means, you don't have to write the majority of the handling terraform, API api, etc. Instead, terratest gives you lots of helpers, and you focus on expressing your infrastructure tests.

For our purposes, the helpers around terraform automate the running of terraform plans, parsing the output, and cleaning up afterwards. Later on, we'll explore if we can also verify the resources created against AWS API

## Let's get started

So, following the quickstart applied to our repo from part 1, we'll create the following files:

```text
+-- tests/
    +-- examples/
        +-- test.tf # this will contain the root module we will test, importing our module under test
    +-- kubernetes-aws.go # this will contain the go code that runs, asserts and cleans the test environment
```

I've created my `tests/examples/test.tf` like so:

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

    // override endpoints to point to localstack
    endpoints {
        ec2            = "http://localhost:5000"
        eks            = "http://localhost:5000"
        iam            = "http://localhost:5000"
        autoscaling    = "http://localhost:5000"
    }
}

// use to get the VPC from moto
data "aws_vpc" "default" {
}
data "aws_subnets" "default" {
    filter {
        name   = "vpc-id"
        values = [data.aws_vpc.default.id]
    }
}

// configure the module with some example values,
module "kubernetes-aws" {
  source            = "../../../modules/kubernetes-aws"
  cluster_name      = "kubernetes-test"
  cluster_version   = "1.21"
  project           = "terraform-testing"
  environment       = "test"
  min_workers       = 1
  max_workers       = 6
  desired_workers   = 3
  instance_type     = "t3.large"
  is_testing        = true

  // connect up to pre-existing vpc and subnet
  vpc_id            = data.aws_vpc.default.id
  subnets           = data.aws_subnets.default.ids
}


// output some useful values
output "name" {
    value = module.kubernetes-aws.name
}
output "endpoint" {
    value = module.kubernetes-aws.endpoint
}
output "user_arn" {
    value = module.kubernetes-aws.aws_iam_role_user_arn
}
```

and `tests/kubernetes-aws_test.go` contains (mostly from the getting started tutorial):

```go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestTerraformKubernetesAwsModule(t *testing.T) {
	// retryable errors in terraform testing.
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "examples/kubernetes-aws",
	})

	defer terraform.Destroy(t, terraformOptions)

	terraform.InitAndApply(t, terraformOptions)

	name := terraform.Output(t, terraformOptions, "name")
	endpoint := terraform.Output(t, terraformOptions, "endpoint")
	user_arn := terraform.Output(t, terraformOptions, "user_arn")
	assert.Equal(t, name, "kubernetes-test")
	assert.Contains(t, endpoint, "us-east-1.eks.amazonaws.com")
	assert.Contains(t, user_arn, "role/kubernetes-test--AmazonEKSUserRole")
}
```

Then I created a little `Makefile` target that runs moto locally, and then runs the tests:
```Makefile
test-infra:
	docker stop kubernetes-cluster-provisioning-test || true
	docker run --rm -d --name kubernetes-cluster-provisioning-test -p 5000:5000 motoserver/moto:latest
	cd tests && go test -v -timeout 30m
```

run `make test-infra` and:

```shell
<snip a lot of terraform output>
--- PASS: TestTerraformKubernetesAwsModule (45.62s)
PASS
ok  	github.com/elifesciences/kubernetes-cluster-provisioning/v2	45.854s
```

# 🎉💃🕺🎉

Remember - this is running against the local moto (disconnect from your AWS account to be sure). It also runs faster than against AWS, and obviously saves money.

Next time, we will look into using more examples and assertions against the moto API to verify if the resources have been created correctly.
