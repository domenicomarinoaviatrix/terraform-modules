## Aviatrix - Terraform Modules

### Description
These Terraform modules allow you to build Aviatrix Controllers and prepare accounts for connecting to an existing Aviatrix Controller.

>**NOTE:** It is ***highly*** recommended to have the following modules in a directory separate from the rest of your Terraform pipeline. These modules only need to be run once to build the Controller and dependent resources.

### Available Modules

| Module  | Description | Prerequisites |
| ------- | ----------- | ------------- |
| [aviatrix-controller-iam-roles](./aviatrix-controller-iam-roles) | Builds the IAM roles required for Aviatrix to connect with this AWS account | None |
| [aviatrix-controller-build](./aviatrix-controller-build) | Builds the controller in an AWS VPC | <ul><li>Existing VPC,</li><li>Public subnet,</li><li>An AWS Key Pair,</li><li>IAM roles created</li></ul> |
| [aviatrix-controller-initialize](./aviatrix-controller-initialize) | Initializes the Controller (Upgrade, set admin e-mail, set admin password). | <ul><li>Aviatrix Controller</li></ul> |

### Basic Usage

```
module "<<name>>" {
  source = "github.com/AviatrixSystems/terraform-modules.git/<<module-name>>"
  var1   = "<<value1>>"
}
```

### Building a Controller and Initializing it

The Aviatrix provider requires the Controller to be up and running before it can operate, the initialization can be
separated from building the Controller:

#### 1. Create the IAM roles

**createIAM.tf**

``` hcl

provider "aws" {
  <<< your credentials and region >>>
}

module "iam_roles" {
  source = "github.com/AviatrixSystems/terraform-modules.git//aviatrix-controller-iam-roles?ref=master"
}

```

*Execute*

``` shell
> cd iam
> terraform init
> terraform apply
> cd ..
```

#### 2. Build the Controller

**build.tf**

> **NOTE:** **incoming_ssl_cidr** is a required variable. To use the module **aviatrix-controller-initialize**, please add the VPC CIDR in the list.

``` hcl

provider "aws" {
  <<< your credentials and region >>>
}

module "aviatrixcontroller" {
  source            = "github.com/AviatrixSystems/terraform-modules.git//aviatrix-controller-build?ref=master"
  vpc               = "<<< your VPC ID >>>"
  subnet            = "<<< your public Subnet ID >>>"
  keypair           = "<<< your EC2 key pair name >>>"
  ec2role           = "aviatrix-role-ec2"
  incoming_ssl_cidr = ["<<< VPC CIDR >>>", "<<< another CIDR allowed for HTTPS access >>>", ...]
  name_prefix       = "<<< unique for specified controller >>>"
}

output "controller_private_ip" {
  value = module.aviatrixcontroller.private_ip
}

output "controller_public_ip" {
  value = module.aviatrixcontroller.public_ip
}

```

*Execute*

``` shell
> cd build
> terraform init
> terraform apply
> terraform output | sed -ne 's/\(.*\) = \(.*\)/\1="\2"/p' > ../aviatrix_controller.tfvars
> cd ..
```

#### 3. Initialize the Controller

**init.tf**

``` hcl
provider "aws" {
  <<< your credentials and region >>>
}

variable "controller_private_ip" {}

variable "controller_public_ip" {}

module "aviatrix_controller_init" {
  source              = "github.com/AviatrixSystems/terraform-modules.git//aviatrix-controller-initialize?ref=master"
  admin_email         = "<<< your administrator email address >>>"
  admin_password      = "<<< your new admin password >>>"
  private_ip          = var.controller_private_ip
  public_ip           = var.controller_public_ip
  access_account_name = "<<< your account name mapping to your AWS account in the Aviatrix Controller >>>"
  aws_account_id      = "<<< your aws account id >>>"
  vpc_id              = "<<< your vpc id here >>>"
  subnet_id           = "<<< your public subnet id >>>"
  customer_license_id = "<<< your customer license id (optional) >>>"   
}

output "lambda_result" {
  value = "${module.aviatrix_controller_init.result}"
}

```

*Execute*

``` shell
> cd init
> terraform init
> terraform apply -var-file=../aviatrix_controller.tfvars
```

> **NOTE:** If the Lambda function times out, please try to increase `wait_time_for_instance`, which is 120s at default.

### Putting it all together

You can run each of these steps in a single .tf file.  Here is an example:

``` hcl

provider "aws" {
  <<< your credentials and region >>>
}

data "aws_caller_identity" "current" {}

module "aviatrix-iam-roles" {
  source = "github.com/AviatrixSystems/terraform-modules.git//aviatrix-controller-iam-roles?ref=master"
}

module "aviatrix-controller-build" {
  source            = "github.com/AviatrixSystems/terraform-modules.git//aviatrix-controller-build?ref=master"
  vpc               = "<<< VPC ID >>>"
  subnet            = "<<< Subnet ID >>>"
  keypair           = "<<< Keypair name >>>"
  ec2role           = module.aviatrix-iam-roles.aviatrix-role-ec2-name
  incoming_ssl_cidr = ["<<< VPC CIDR >>>", "<<< another CIDR allowed for HTTPS access >>>", ...]
}

provider "aviatrix" {
  username      = "admin"
  password      = module.aviatrix-controller-build.private_ip
  controller_ip = module.aviatrix-controller-build.public_ip
}

module "aviatrix-controller-initialize" {
  source              = "github.com/AviatrixSystems/terraform-modules.git//aviatrix-controller-initialize?ref=master"
  admin_password      = "<<< new admin password >>>"
  admin_email         = "<<< admin email address >>>"
  private_ip          = module.aviatrix-controller-build.private_ip
  public_ip           = module.aviatrix-controller-build.public_ip
  access_account_name = "<<< account name for this AWS account >>>"
  aws_account_id      = data.aws_caller_identity.current.account_id
  vpc_id              = module.aviatrix-controller-build.vpc_id
  subnet_id           = module.aviatrix-controller-build.subnet_id
}

output "result" {
  value = module.aviatrix-controller-initialize.result
}

output "controller_private_ip" {
  value = module.aviatrix-controller-build.private_ip
}

output "controller_public_ip" {
  value = module.aviatrix-controller-build.public_ip
}
```
